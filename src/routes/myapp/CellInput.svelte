<script>
// @ts-nocheck

	import { createEventDispatcher } from "svelte";

    export let cell;
    let isComposing = false;
    const dispatch = createEventDispatcher();

    function handleClick() {
        cell.isSelected = true;
    }

    function handleBlur() {
        cell.isSelected = false;
        dispatch('change', {
            cell
        });
    }

    function handleKeydown(event) {
        if (isComposing) {
            return;
        }

        if (event.key === 'Enter') {
            if (event.shiftKey) {
                if (cell.up !== undefined) {
                    handleBlur();
                    cell.up.isSelected = true;
                }
            } else {
                if (cell.down !== undefined) {
                    handleBlur();
                    cell.down.isSelected = true;
                }
            }
        } else if (event.key === 'Tab') {
            if (event.shiftKey) {
                if (cell.left !== undefined) {
                    handleBlur();
                    cell.left.isSelected = true;
                }
            } else {
                if (cell.right !== undefined) {
                    handleBlur();
                    cell.right.isSelected = true;
                }
            }
            event.preventDefault();
        }
    }
</script>

<td>
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div class="cell"
        on:click={handleClick}
        class:selected-cell={cell.isSelected}
        class:numcell={typeof cell.value === "number"}>
    {#if cell.isSelected}
        <!-- svelte-ignore a11y-autofocus -->
        <!-- svelte-ignore a11y-autofocus -->
        <input type="text"
            bind:value={cell.rawValue}
            on:keydown={handleKeydown}
            on:blur={handleBlur}
            on:compositionstart={() => isComposing = true}
            on:compositionend={() => isComposing = false}
            autofocus>
    {:else if cell.hasError}
        <span style="color: red;">Error</span>
    {:else}
        {cell.value}
    {/if}
    </div>
</td>

<style>
    td {
		border: 0.1px solid gray;
        word-wrap: break-word;
	}
    .cell {
        position: relative;
        padding: 0px;
        width: 100px;
        height: 20px;
        line-height: 20px;
        text-align: left;
        overflow: hidden;
        box-sizing: border-box;
    }

    .selected-cell {
        border: 2px solid blue;
    }

    .numcell {
        text-align: right;
    }

    input {
        width: 100%;
        height: 100%;
        border: None;
        padding: 0px;
        box-sizing: border-box;
        vertical-align: middle;
        outline: solid 1px;
    }
</style>
