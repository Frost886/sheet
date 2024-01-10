<svelte:head>
	<title>About</title>
	<meta name="description" content="About this app" />
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
		max-height: 50em;
		border: 1px solid #dadada;
		overflow: scroll;
		overflow-x: scroll;
		overflow-y: scroll;
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
