<script>
	import { createEventDispatcher } from 'svelte';
	import CellInput from './CellInput.svelte';

	export let value = '';

	const dispatch = createEventDispatcher();

	const select = (num) => () => (value += num);
	const clear = () => (value = '');
	const submit = () => dispatch('submit');

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
		VAR: 2,
		EOF: 3,
	});

	class Token {
		constructor(type, str = '', value = 0) {
			this.type = type;
			this.next = null;
			this.str = str;
			this.value = value;
		}
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
			// punctuators
			if (/^[+\-*/()]/.test(s)) {
				cur = cur.next = new Token(TK.PUNCT, s[0]);
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
			// variable
			// ここでnumかstrに変換するのが良さそう
			if (/^[a-zA-Z]\d+/.test(s)) {
				const value = s.match(/^[a-zA-Z]\d+/)[0];
				cur = cur.next = new Token(TK.VAR, value);
				refs.add(value);
				i += value.length;
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
			return { value: e.message, hasError };
		}
	}

	// expr = term ("+" term | "-" term)*
	function expr() {
		let value = term();
		while (true) {
			if (consume('+')) {
				value += term();
			} else if (consume('-')) {
				value -= term();
			} else {
				return value;
			}
		}
	}

	// term = factor ("*" factor | "/" factor)*
	function term() {
		let value = factor();
		while (true) {
			if (consume('*')) {
				value *= factor();
			} else if (consume('/')) {
				value /= factor();
			} else {
				return value;
			}
		}
	}

	// factor = num | var | "(" expr ")"
	function factor() {
		if (tok.type === TK.NUM) {
			const value = tok.value;
			tok = tok.next;
			return value;
		}
		if (tok.type === TK.VAR) {
			const ref = tok.str;
			const str = parseCellReference(ref).value;
			let value = 0;
			if (/^\d+(\.\d+)?|\.\d+$/.test(str)) {
				value = parseFloat(str);
			}
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
	.keypad {
		display: grid;
        justify-content: center;
		grid-template-columns: repeat(18, 5em);
		grid-template-rows: repeat(50, 2em);
		grid-gap: 0em;
	}

	button {
		margin: 0;
	}
</style>
