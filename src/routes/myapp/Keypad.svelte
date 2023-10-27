<script>
	import { createEventDispatcher } from 'svelte';
	import CellInput from './CellInput.svelte';

	export let value = '';

	const dispatch = createEventDispatcher();

	// @ts-ignore
	const select = (num) => () => (value += num);
	const clear = () => (value = '');
	const submit = () => dispatch('submit');

	const arr = new Array(50).fill(null).map(() => new Array(18).fill(''));
	const arr2 = new Array(50).fill(null).map(() => new Array(18).fill(''));

	// セルごとに参照先（親）、こちらを参照しているセル（子）の値を保持する
	let row = 0;
	let col = 0;
	let v = 0;
	function test(e) {
		v = e.target.value;
		row = e.target.parentElement.parentElement.rowIndex;
		col = e.target.parentElement.cellIndex;
		const i = row - 1;
		const j = col - 1;
		var input = document.getElementById((i * 19 + j).toString());
		arr[i][j] = v;
	}

</script>
row {row} col {col} v {v}
arr[0][0] {arr[0][0]}
<div>
	<table>
		<thead>
			<tr>
				<th></th>
				{#each {length: 18} as _, i}
					<th>{String.fromCharCode(i + 65)}</th>
				{/each}
			</tr>
		</thead>
		<tbody>
			{#each {length: 50} as _, i}
				<tr>
					<th>{i + 1}</th>
					{#each {length: 18} as _, j}
						<td on:click={test}>
							<!-- <input id={i * 19 + j}.toString() value={arr[i][j]} on:keydown={test} type="hidden" /> -->
							<CellInput />
						</td>
					{/each}
				</tr>
			{/each}
		</tbody>
	</table>
</div>

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
