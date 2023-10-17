<script>
	import { createEventDispatcher } from 'svelte';

	export let value = '';

	const dispatch = createEventDispatcher();

	// @ts-ignore
	const select = (num) => () => (value += num);
	const clear = () => (value = '');
	const submit = () => dispatch('submit');

	let data = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 
				'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 
				'u', 'v', 'w', 'x', 'y', 'z'];
</script>

<div class="keypad">
	{#each {length: 50} as _, i}
		{#each {length: 18} as _, j}
			<input value='' placeholder={String.fromCharCode(j + 65) + (i+1)} />
		{/each}
	{/each}
</div>

<div>
	<table>
		<thead>
			<tr>
				{#each {length: 18} as _, i}
					<th>{String.fromCharCode(i + 65)}</th>
				{/each}
			</tr>
		</thead>
		<tbody>
			{#each {length: 50} as _, i}
				<tr>
					{#each {length: 18} as _, j}
						<td><input value='' /></td>
					{/each}
				</tr>
			{/each}
		</tbody>
	</table>
</div>

<style>
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
