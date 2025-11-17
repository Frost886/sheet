<svelte:head>
	<title>Sheet</title>
	<meta name="description" content="A sheet app made by Svelte." />
</svelte:head>

<script>
	import Sheet from './Sheet.svelte';
</script>

<div class="container">
	<Sheet/>
</div>
<style>
	.sheet {
		display: flex;
		justify-content: left;
		width: 100%;
	}

	.container {
		height: 98.5vh;
		max-height: none;
		width: 100%;
		border: 1px solid #dadada;
		overflow: auto;
	}

	/* Overwrite the default to keep the scrollbar always visible */

	::-webkit-scrollbar {
		-webkit-appearance: none;
		width: 7px;
		height: 7px;
	}

	::-webkit-scrollbar-thumb {
		border-radius: 4px;
		background-color: rgba(0, 0, 0, 0.5);
		-webkit-box-shadow: 0 0 1px rgba(255, 255, 255, 0.5);
	}


</style>
