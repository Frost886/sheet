<script>
	import { createEventDispatcher } from "svelte";

    let isEditing = false;
    export let cell;
    const dispatch = createEventDispatcher();

    function handleClick() {
        isEditing = true;
    }

    function handleBlur() {
        isEditing = false;
        // cell.value = cell.rawValue;
        dispatch('change', {
            cell
        });
    }
</script>

<div class="cell" on:click={handleClick}>
    {#if isEditing}
        <input type="text" bind:value={cell.rawValue} on:blur={handleBlur} autofocus>
    {:else if cell.hasError}
        <span style="color: red;">Error</span>
    {:else}
        {cell.value}
    {/if}
</div>

<style>
    .cell {
        position: relative;
        border: 1px solid #ccc;
        padding: 5px;
        width: 120px;  /* 200pxのセルサイズからpaddingやborderを引いたサイズ */
        height: 20px;
        line-height: 20px;
        text-align: center;
        cursor: pointer;  /* セルがクリック可能であることを示すカーソルスタイル */
    }

    input {
        width: 100%;
        height: 100%;
        border: none;
        padding: 5px;
        box-sizing: border-box;
    }
</style>
