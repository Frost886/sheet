<script>
	import { createEventDispatcher } from 'svelte';
	import CellInput from './CellInput.svelte';

	export let value = '';

	const dispatch = createEventDispatcher();

	// @ts-ignore
	const select = (num) => () => (value += num);
	const clear = () => (value = '');
	const submit = () => dispatch('submit');

	let cells = new Array(50).fill(null).map(() =>
		new Array(20).fill(null).map(() => ({
			value: '',
			rawValue: '',
			parents: [],
			children: [],
			hasError : false
		}))
	);

	// セルごとに参照先（親）、こちらを参照しているセル（子）の値を保持する

	function parseCellReference(str) {
		const row = parseInt(str.match(/\d+/)[0], 10) - 1;
		const col = str.match(/[A-Z]+/)[0].split('').reduce((acc, char) => acc * 26 + char.charCodeAt(0) - 65 + 1, 0) - 1;
		return cells[row][col];
	}

	function onRawValueChanged(event) {
		const { cell } = event.detail;
		updateValue(cell);
	}

	function updateValue(cell) {
		let rawValue = cell.rawValue;
		for (const parent of cell.parents) {
			parent.children = parent.children.filter(child => child !== cell);
		}
		cell.hasError = false;
		cell.parents = [];
		if (rawValue[0] === '=') {
			rawValue = rawValue.slice(1).toUpperCase();
			const refs = rawValue.match(/[A-Z]\d+/g);
			if (refs) {
				// refs.forEach(ref => {
				// 	const cell = parseCellReference(ref);
				// 	cell.children.push(cell);
				// 	cells[i][j].parents.push(cell);
				// });
				const parent = parseCellReference(refs[0]);
				cell.parents.push(parent);
				parent.children.push(cell);

				if (hasCycle(cell)) {
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
			} else {
				cell.value = cell.rawValue;
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
			cell.hasError = true;
			return true;
		}
		visited.add(cell);
		for (const child of cell.children) {
			if (hasCycle(child, visited)) {
				child.hasError = true;
				return true;
			}
		}
		visited.delete(cell);
		return false;
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
					<td>
						<CellInput bind:cell={cell} on:change={onRawValueChanged} />
					</td>
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
	}
	td, th, input {
		padding: 0;
		width: 6em;
		border: 0.1px solid gray;
	}
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
