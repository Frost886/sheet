<script>
	import CellInput from './CellInput.svelte';
	import Papa from 'papaparse';

	let cells = new Array(50).fill(null).map(() =>
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
							if (results.data[i][j] === undefined) {
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
		EOF: 4,
	});

	class Token {
		constructor(type, str = '', value = 0) {
			this.type = type;
			this.next = null;
			this.str = str;
			this.value = value;
		}
	}

	function readPunct(str) {
		const kw2 = ["<=", ">="];
		const kw1 = ["<", ">", "=", "(", ")", "+", "-", "*", "/", ":", "&"];
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
			if (/^\d+(\.\d+)?|\.\d+/.test(s)) {
				const value = s.match(/^\d+(\.\d+)?|\.\d+/)[0];
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
			// variable
			if (/^[a-zA-Z]\d+/.test(s)) {
				const value = s.match(/^[a-zA-Z]\d+/)[0];
				cur = cur.next = new Token(TK.VAR, value);
				refs.add(value);
				i += value.length;
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

	// relational = concat ("=" concat | "<" concat | ">" concat | "<=" concat | ">=" concat)*
	// T->1, F->0
	function relational() {
		let value = concat();
		while (true) {
			if (consume('=')) {
				value = value === concat() ? 1: 0;
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

	// primary = num | var | '"' str '"' | "(" expr ")"
	function primary() {
		if (tok.type === TK.NUM) {
			const value = tok.value;
			tok = tok.next;
			return value;
		}
		if (tok.type === TK.VAR) {
			const ref = tok.str;
			const val = parseCellReference(ref).value;
			
			// if (typeof val !== 'number') {
			// 	console.log(typeof val);
			// 	throw new Error('unexpected string');
			// }
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
