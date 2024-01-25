<script>
	import CellInput from './CellInput.svelte';
	import Papa from 'papaparse';

	let cells = new Array(300).fill(null).map(() =>
		new Array(26).fill(null).map(() => ({
			value: '',
			rawValue: '',
			parents: [],
			children: [],
			hasError : false,
			isSelected: false,
			up: undefined,
			down: undefined,
			left: undefined,
			right: undefined,
		}))
	);

	//
	// Save and Load CSV
	//
	let fileName = '';
	function saveCsv() {
		const data = cells.map(row => row.map(cell => cell.rawValue));
		const csv = Papa.unparse(data);
		const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		let f = fileName;
		if (f === '') {
			f = 'untitled';
		}
		a.download = `${f}.csv`;
		a.click();
	}

	function loadCsv() {
		const input = document.createElement('input');
		input.type = 'file';
		input.accept = '.csv';
		input.onchange = () => {
			Papa.parse(input.files[0], {
				complete: function(results) {
					for (let i = 0; i < cells.length; i++) {
						for (let j = 0; j < cells[i].length; j++) {
							if (results.data.length <= i || results.data[i].length <= j) {
								cells[i][j].rawValue = '';
							} else {
								cells[i][j].rawValue = results.data[i][j];
							}
							cells[i][j].value = '';
							cells[i][j].parents = [];
							cells[i][j].children = [];
							cells[i][j].hasError = false;
						}
					}
					updateAll();
				}
			})
		};
		input.click();
	}

	function updateAll() {
		for (const row of cells) {
			for (const cell of row) {
				updateValue(cell);
			}
		}
	}

	// up, down, left, rightの参照を設定する
	for (let i = 0; i < cells.length; i++) {
		for (let j = 0; j < cells[i].length; j++) {
			cells[i][j].up = cells[i - 1]?.[j];
			cells[i][j].down = cells[i + 1]?.[j];
			cells[i][j].left = cells[i]?.[j - 1];
			cells[i][j].right = cells[i]?.[j + 1];
		}
	}

	// セルごとに参照先（親）、こちらを参照しているセル（子）の値を保持する

	function parseCellReference(str) {
		str = str.toUpperCase();
		const row = parseInt(str.match(/\d+/)[0], 10) - 1;
		const col = str.match(/[A-Z]+/)[0].split('').reduce((acc, char) => acc * 26 + char.charCodeAt(0) - 65 + 1, 0) - 1;
		return cells[row][col];
	}

	function rangeToIndices(str) {
		str = str.toUpperCase();
		const [start, end] = str.split(':');
		const i1 = parseInt(start.match(/\d+/)[0], 10) - 1;
		const j1 = start.match(/[A-Z]+/)[0].split('').reduce((acc, char) => acc * 26 + char.charCodeAt(0) - 65 + 1, 0) - 1;
		const i2 = parseInt(end.match(/\d+/)[0], 10) - 1;
		const j2 = end.match(/[A-Z]+/)[0].split('').reduce((acc, char) => acc * 26 + char.charCodeAt(0) - 65 + 1, 0) - 1;
		return [i1, j1, i2, j2];
	}

	function onRawValueChanged(event) {
		const { cell } = event.detail;
		updateValue(cell);
	}

	//
	// Tokenizer
	//
	const TK = Object.freeze({
		PUNCT: 0,
		NUM: 1,
		STR: 2,
		VAR: 3,
		RANGE: 4,
		FUNC: 5,
		EOF: 6,
	});

	class Token {
		constructor(type, str = '', value = 0) {
			this.type = type;
			this.next = null;
			this.str = str;
			this.value = value;
		}
	}

	function readFunc(str) {
		const funcs = ["IF", "NOT", "AND", "OR", "FIND", "INT", "ROUND", "ABS",
			"SUM", "AVERAGE", "MAX", "MIN", "COUNT", "COUNTIF", "INDEX",
			"LOOKUP", "VLOOKUP", "MATCH", "SUMIF", "AVERAGEIF"];
		for (const f of funcs) {
			if (str.toUpperCase().startsWith(f)) {
				return f.length;
			}
		}
		return 0;
	}

	function readPunct(str) {
		const kw2 = ["<=", ">=", "<>"];
		const kw1 = ["<", ">", "=", "(", ")", "+", "-", "*", "/", "&", ","];
		for (const k of kw2) {
			if (str.startsWith(k)) {
				return k.length;
			}
		};
		for (const k of kw1) {
			if (str.startsWith(k)) {
				return 1;
			}
		};
		return 0;
	}

	let tok = null;

	function tokenize(str) {
		let head = new Token(null);
		let cur = head;
		let i = 0;
		let hasError = false;
		let refs = new Set();
		while (i < str.length) {
			const s = str.slice(i);
			// skip whitespace
			if (/^\s/.test(s)) {
				i++;
				continue;
			}
			// number
			if (/^(\d+(\.\d+)?|\.\d+)/.test(s)) {
				const value = s.match(/^(\d+(\.\d+)?|\.\d+)/)[0];
				cur = cur.next = new Token(TK.NUM, value, parseFloat(value));
				i += value.length;
				continue;
			}
			// string
			if (/^"/.test(s)) {
				let str = s.slice(1);
				str = str.match(/^[^"]*/)[0];
				cur = cur.next = new Token(TK.STR, str);
				i += str.length + 2;
				continue;
			}
			if (/^'/.test(s)) {
				let str = s.slice(1);
				str = str.match(/^[^']*/)[0];
				cur = cur.next = new Token(TK.STR, str);
				i += str.length + 2;
				continue;
			}
			// range
			if (/^[a-zA-Z]\d+:[a-zA-Z]\d+/.test(s)) {
				const value = s.match(/^[a-zA-Z]\d+:[a-zA-Z]\d+/)[0];
				cur = cur.next = new Token(TK.RANGE, value);
				const [i1,j1,i2,j2] = rangeToIndices(value);
				for (let i = i1; i <= i2; i++) {
					for (let j = j1; j <= j2; j++) {
						refs.add(String.fromCharCode(j + 65) + (i + 1));
					}
				}
				i += value.length;
				continue;
			}
			// variable
			if (/^[a-zA-Z]\d+/.test(s)) {
				const value = s.match(/^[a-zA-Z]\d+/)[0];
				cur = cur.next = new Token(TK.VAR, value);
				refs.add(value);
				i += value.length;
				continue;
			}
			// function
			const func_len = readFunc(s);
			if (func_len) {
				cur = cur.next = new Token(TK.FUNC, s.slice(0, func_len));
				i += func_len;
				continue;
			}
			// punctuator
			const punct_len = readPunct(s);
			if (punct_len) {
				cur = cur.next = new Token(TK.PUNCT, s.slice(0, punct_len));
				i += punct_len;
				continue;
			}
			hasError = true;
			break;
		}
		cur.next = new Token(TK.EOF);
		tok = head.next;
		return { hasError, refs };
	}

	function consume(op) {
		if (tok.type !== TK.PUNCT || tok.str !== op) {
			return false;
		}
		tok = tok.next;
		return true;
	}

	function isFloat(str) {
		return /^-?(\d+(\.\d+)?|\.\d+)$/.test(str);
	}

	//
	// Parser
	//
	function parse() {
		let hasError = false;
		try {
			const value = expr();
			if (tok.type !== TK.EOF) {
				throw new Error('extra token');
			}
			return { value, hasError };
		} catch (e) {
			hasError = true;
			console.log(e.message);
			return { value: e.message, hasError };
		}
	}

	// expr = relational
	function expr() {
		return relational();
	}

	// relational = concat ("=" concat | "<>"" concat | "<" concat | ">" concat | "<=" concat | ">=" concat)*
	// T->1, F->0
	function relational() {
		let value = concat();
		while (true) {
			if (consume('=')) {
				value = value === concat() ? 1: 0;
			} else if (consume('<>')) {
				value = value !== concat() ? 1 : 0;
			} else if (consume('<')) {
				value = value < concat() ? 1 : 0;
			} else if (consume('>')) {
				value = value > concat() ? 1 : 0;
			} else if (consume('<=')) {
				value = value <= concat() ? 1 : 0;
			} else if (consume('>=')) {
				value = value >= concat() ? 1 : 0;
			} else {
				return value;
			}
		}
	}

	// concat = add ("&" add)*
	// str & str -> str
	function concat() {
		let value = add();
		while (true) {
			if (consume('&')) {
				const a = add();
				if (typeof value !== 'string' || typeof a !== 'string') {
					throw new Error('undefined opration for &');
				}
				value = value + a;
			} else {
				return value;
			}
		}
	}

	// add = mul ("+" mul | "-" mul)*
	// num op num -> num
	function add() {
		let value = mul();
		while (true) {
			if (consume('+')) {
				const m = mul();
				if (typeof value !== 'number' || typeof m !== 'number') {
					throw new Error('undefined opration for +');
				}
				value += m;
			} else if (consume('-')) {
				const m = mul();
				if (typeof value !== 'number' || typeof m !== 'number') {
					throw new Error('undefined opration for -');
				}
				value -= m;
			} else {
				return value;
			}
		}
	}

	// mul = unary ("*" unary | "/" unary)*
	// num op num -> num
	function mul() {
		let value = unary();
		while (true) {
			if (consume('*')) {
				let u = unary();
				if (typeof value !== 'number' || typeof u !== 'number') {
					throw new Error('undefined opration for *');
				}
				value *= u;
			} else if (consume('/')) {
				let u = unary();
				if (typeof value !== 'number' || typeof u !== 'number') {
					throw new Error('undefined opration for /');
				}
				value /= u;
			} else {
				return value;
			}
		}
	}

	// unary = ("+" | "-") unary | primary
	function unary() {
		if (consume('+')) {
			let u = unary();
			if (typeof u !== 'number') {
				throw new Error('undefined unary opration for +');
			}
			return u;
		}
		if (consume('-')) {
			let u = unary();
			if (typeof u !== 'number') {
				throw new Error('undefined unary opration for -');
			}
			return -u;
		}
		return primary();
	}

	// primary = num | var | str | "(" expr ")" | func "(" (expr | range) ("," (expr | range))* ")"
	function primary() {
		if (tok.type === TK.NUM) {
			const value = tok.value;
			tok = tok.next;
			return value;
		}
		if (tok.type === TK.VAR) {
			const ref = tok.str;
			const val = parseCellReference(ref).value;
			tok = tok.next;
			return val;
		}
		if (tok.type === TK.STR) {
			const value = tok.str;
			tok = tok.next;
			return value;
		}
		if (consume('(')) {
			const value = expr();
			if (!consume(')')) {
				throw new Error('missing )');
			}
			return value;
		}
		if (tok.type === TK.FUNC) {
			const func = tok.str.toUpperCase();
			tok = tok.next;
			if (!consume('(')) {
				throw new Error('missing (');
			}
			const args = [];
			if (!consume(')')) {
				if (tok.type === TK.RANGE) {
					const [i1,j1,i2,j2] = rangeToIndices(tok.str);
					args.push([i1, j1, i2, j2]);
					tok = tok.next;
				} else {
					args.push(expr());
				}
				while (consume(',')) {
					if (tok.type === TK.RANGE) {
						const [i1,j1,i2,j2] = rangeToIndices(tok.str);
						args.push([i1, j1, i2, j2]);
						tok = tok.next;
					} else {
						args.push(expr());
					}
				}
				if (!consume(')')) {
					throw new Error('missing )');
				}
			}
			if (func === 'IF') {
				if (args.length !== 2 && args.length !== 3) {
					throw new Error('IF takes 2 or 3 args');
				}
				if (typeof args[0] !== 'number') {
					throw new Error('IF arg1 takes number');
				}
				if (args[0] !== 0) {
					if (typeof args[1] === 'object') {
						throw new Error('IF arg2 takes number or string');
					}
					return args[1];
				}
				if (args.length === 3) {
					if (typeof args[2] === 'object') {
						throw new Error('IF arg3 takes number or string');
					}
					return args[2];
				}
				return 0;
			}
			if (func === 'NOT') {
				if (args.length !== 1) {
					throw new Error('NOT takes 1 argument');
				}
				if (typeof args[0] !== 'number') {
					throw new Error('NOT takes number');
				}
				return args[0] === 0 ? 1 : 0;
			}
			if (func === 'AND') {
				if (args.length === 0) {
					throw new Error('AND takes 1 or more arguments');
				}
				for (const arg of args) {
					if (typeof arg !== 'number') {
						throw new Error('AND takes number');
					}
					if (arg === 0) {
						return 0;
					}
				}
				return 1;
			}
			if (func === 'OR') {
				if (args.length === 0) {
					throw new Error('OR takes 1 or more arguments');
				}
				for (const arg of args) {
					if (typeof arg !== 'number') {
						throw new Error('OR takes number');
					}
					if (arg !== 0) {
						return 1;
					}
				}
				return 0;
			}
			if (func === "FIND") {
				if (args.length !== 2 && args.length !== 3) {
					throw new Error('FIND takes 2 or 3 arguments');
				}
				let start = 1;
				if (args.length === 3) {
					if (typeof args[2] !== 'number') {
						throw new Error('FIND arg3 must be number');
					}
					start = args[2] > 1 ? Math.floor(args[2]) : 1;
				}
				if (typeof args[0] !== 'string' || typeof args[1] !== 'string') {
					throw new Error('FIND arg1 and arg2 must be string');
				}
				let index = args[1].slice(start-1).indexOf(args[0]);
				if (index === -1) {
					return 0;
				}
				return index + start;
			}
			if (func === "INT") {
				if (args.length !== 1) {
					throw new Error('INT takes 1 argument');
				}
				if (typeof args[0] !== 'number') {
					throw new Error('INT takes number');
				}
				return Math.floor(args[0]);
			}
			if (func === "ROUND") {
				if (args.length !== 1 && args.length !== 2) {
					throw new Error('ROUND takes 1 or 2 arguments');
				}
				if (typeof args[0] !== 'number') {
					throw new Error('ROUND takes number');
				}
				let digit = 0;
				if (args.length === 2) {
					if (typeof args[1] !== 'number') {
						throw new Error('ROUND argument 2 must be number');
					}
					digit = Math.floor(args[1]);
				}
				return Math.round(args[0] * Math.pow(10, digit)) / Math.pow(10, digit);
			}
			if (func === "ABS") {
				if (args.length !== 1) {
					throw new Error('ABS takes 1 argument');
				}
				if (typeof args[0] !== 'number') {
					throw new Error('ABS takes number');
				}
				return Math.abs(args[0]);
			}
			if (func === "SUM") {
				if (args.length === 0) {
					throw new Error('SUM takes 1 or more arguments');
				}
				let sum = 0;
				for (const arg of args) {
					// argが文字列の場合は無視する
					if (typeof arg === 'number') {
						sum += arg;
					} else if (typeof arg === 'object') {
						const [i1,j1,i2,j2] = arg;
						for (let i = i1; i <= i2; i++) {
							for (let j = j1; j <= j2; j++) {
								if (typeof cells[i][j].value === 'number') {
									sum += cells[i][j].value;
								} 
							}
						}
					}
				}
				return sum;
			}
			if (func === "AVERAGE") {
				if (args.length === 0) {
					throw new Error('AVERAGE takes 1 or more arguments');
				}
				let sum = 0;
				let count = 0;
				for (const arg of args) {
					// argが文字列の場合は無視する
					if (typeof arg === 'number') {
						sum += arg;
						count++;
					} else if (typeof arg === 'object') {
						const [i1,j1,i2,j2] = arg;
						for (let i = i1; i <= i2; i++) {
							for (let j = j1; j <= j2; j++) {
								if (typeof cells[i][j].value === 'number') {
									sum += cells[i][j].value;
									count++;
								} 
							}
						}
					}
				}
				return sum / count;
			}
			if (func === "COUNT") {
				if (args.length === 0) {
					throw new Error('COUNT takes 1 or more arguments');
				}
				let count = 0;
				for (const arg of args) {
					// argが文字列の場合は無視する
					if (typeof arg === 'number') {
						count++;
					} else if (typeof arg === 'object') {
						const [i1,j1,i2,j2] = arg;
						for (let i = i1; i <= i2; i++) {
							for (let j = j1; j <= j2; j++) {
								if (typeof cells[i][j].value === 'number') {
									count++;
								} 
							}
						}
					}
				}
				return count;
			}
			if (func === "MAX") {
				if (args.length === 0) {
					throw new Error('MAX takes 1 or more arguments');
				}
				let max = -Infinity;
				for (const arg of args) {
					// argが文字列の場合は無視する
					if (typeof arg === 'number') {
						max = Math.max(max, arg);
					} else if (typeof arg === 'object') {
						const [i1,j1,i2,j2] = arg;
						for (let i = i1; i <= i2; i++) {
							for (let j = j1; j <= j2; j++) {
								if (typeof cells[i][j].value === 'number') {
									max = Math.max(max, cells[i][j].value);
								} 
							}
						}
					}
				}
				return max;
			}
			if (func === "MIN") {
				if (args.length === 0) {
					throw new Error('MIN takes 1 or more arguments');
				}
				let min = Infinity;
				for (const arg of args) {
					// argが文字列の場合は無視する
					if (typeof arg === 'number') {
						min = Math.min(min, arg);
					} else if (typeof arg === 'object') {
						const [i1,j1,i2,j2] = arg;
						for (let i = i1; i <= i2; i++) {
							for (let j = j1; j <= j2; j++) {
								if (typeof cells[i][j].value === 'number') {
									min = Math.min(min, cells[i][j].value);
								} 
							}
						}
					}
				}
				return min;
			}
			if (func === "INDEX") {
				if (args.length !== 3) {
					throw new Error('INDEX takes 3 arguments');
				}
				if (typeof args[0] !== 'object') {
					throw new Error('INDEX arg1 must be range');
				}
				if (typeof args[1] !== 'number') {
					throw new Error('INDEX arg2 must be number');
				}
				if (typeof args[2] !== 'number') {
					throw new Error('INDEX arg3 must be number');
				}
				const [i1,j1,i2,j2] = args[0];
				const k = Math.floor(args[1]);
				const l = Math.floor(args[2]);
				if (k < 1 || (i2 - i1 + 1) < k) {
					throw new Error('INDEX arg2 out of range');
				}
				if (l < 1 || (j2 - j1 + 1) < l) {
					throw new Error('INDEX arg3 out of range');
				}
				return cells[i1 + k - 1][j1 + l - 1].value;
			}
			if (func === "MATCH") {
				if (args.length !== 2) {
					throw new Error('MATCH takes 2 arguments');
				}
				if (typeof args[0] === 'object') {
					throw new Error('MATCH arg1 must be number or string');
				}
				if (typeof args[1] !== 'object') {
					throw new Error('MATCH arg2 must be range');
				}
				const [i1,j1,i2,j2] = args[1];
				if (i1 !== i2 && j1 !== j2) {
					throw new Error('MATCH arg2 must be row or column');
				}
				const key = args[0];
				if (i1 === i2) {
					for (let j = j1; j <= j2; j++) {
						if (cells[i1][j].value === key) {
							return j - j1 + 1;
						}
					}
				} else {
					for (let i = i1; i <= i2; i++) {
						if (cells[i][j1].value === key) {
							return i - i1 + 1;
						}
					}
				}
				throw new Error('MATCH key not found');
			}
			if (func === "VLOOKUP") {
				if (args.length !== 3) {
					throw new Error('VLOOKUP takes 3 arguments');
				}
				if (typeof args[0] === 'object') {
					throw new Error('VLOOKUP arg1 must be number or string');
				}
				if (typeof args[1] !== 'object') {
					throw new Error('VLOOKUP arg2 must be range');
				}
				if (typeof args[2] !== 'number') {
					throw new Error('VLOOKUP arg3 must be number');
				}
				const key = args[0];
				const [i1,j1,i2,j2] = args[1];
				const k = Math.floor(args[2]);
				if (k < 1 || (j2 - j1 + 1) < k) {
					throw new Error('VLOOKUP arg3 out of range');
				}
				for (let i = i1; i <= i2; i++) {
					if (cells[i][j1].value === key) {
						return cells[i][j1 + k - 1].value;
					}
				}
				throw new Error('VLOOKUP key not found');
			}
		}
		throw new Error('unexpected token');
	}

	function updateValue(cell) {
		let rawValue = cell.rawValue;
		for (const parent of cell.parents) {
			parent.children = parent.children.filter(child => child !== cell);
		}
		cell.hasError = false;
		cell.parents = [];
		if (rawValue[0] === '=') {
			rawValue = rawValue.slice(1);
			const { hasError, refs } = tokenize(rawValue);
			if (hasError) {
				cell.value = rawValue;
			} else {
				// 循環参照のチェック
				if (refs.size > 0) {
					for (const ref of refs) {
						const parent = parseCellReference(ref);
						cell.parents.push(parent);
						parent.children.push(cell);
					}
					
					let visited = new Set();
					if (hasCycle(cell, visited)) {
						for (const cell of visited) {
							cell.hasError = true;
						}
						cells = [...cells];
						return;
					}
					// 親セルにエラーがある場合は、子セルもエラーにする
					for (const parent of cell.parents) {
						if (parent.hasError) {
							cell.hasError = true;
							break;
						}
					}
					cell.value = cell.parents[0].value;
				}
				// 計算
				if (!cell.hasError) {
					const { value, hasError } = parse();
					if (hasError) {
						cell.hasError = true;
					} else {
						cell.value = value;
					}
				}
			}
		} else if (isFloat(rawValue)) {
			cell.value = parseFloat(rawValue);
		} else {
			cell.value = rawValue;
		}
		for (const child of cell.children) {
			updateValue(child);
		}
		// cellsの変更を検知するために、新しい配列を作成する
		cells = [...cells];
	}

	function hasCycle(cell, visited = new Set()) {
		if (visited.has(cell)) {
			return true;
		}
		visited.add(cell);
		let hasCycle_ = false;
		for (const child of cell.children) {
			if (hasCycle(child, visited)) {
				hasCycle_ = true;
			}
		}
		return hasCycle_;
	}

</script>
<div class="top-bar">
	<input type="text" bind:value={fileName} placeholder="untitled" />.csv
	<button on:click={saveCsv}>Save</button>
	<button on:click={loadCsv}>Load</button>
</div>
<table>
	<thead>
		<tr>
			<th></th>
			{#each cells[0] as _, i}
				<th>{String.fromCharCode(i + 65)}</th>
			{/each}
		</tr>
	</thead>
	<tbody>
		{#each cells as row, i}
			<tr>
				<th>{i + 1}</th>
				{#each row as cell, j}
					<!-- <td> -->
						<CellInput bind:cell={cell} on:change={onRawValueChanged} />
					<!-- </td> -->
				{/each}
			</tr>
		{/each}
	</tbody>
</table>

<style>
	table {
		border-collapse: collapse;
		border-spacing: 0;
		border: 1px solid gray;
		background-color: white;
	}
	th {
		padding: 0;
		width: 6em;
		border: 1px solid gray;
	}
	/* td{
		padding: 0;
		width: 6em;
		border: 0.1px solid gray;
	} */
	.top-bar {
		margin-right: 10px;
	}

	button {
		margin: 0;
	}
</style>
