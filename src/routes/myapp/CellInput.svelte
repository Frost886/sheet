<script>
// @ts-nocheck

	import { createEventDispatcher } from "svelte";

    // let isSelected = false;
    export let cell;
    let isComposing = false;
    let isEditing = false;
    const dispatch = createEventDispatcher();

    function handleClick() {
        cell.isSelected = true;
    }

    function handleDoubleClick() {
        cell.isSelected = false;
        isEditing = true;
    }

    function handleBlur() {
        cell.isSelected = false;
        // cell.value = cell.rawValue;
        dispatch('change', {
            cell
        });
    }

    function handleKeydown(event) {
        if (isComposing) {
            return;
        }

        if (cell.isSelected) {

        }

        if (event.key === 'Enter') {
            cell.isSelected = false;
            dispatch('change', {
                cell
            });
            //if (event.)
            cell.down.isSelected = true;
        } else if (event.key === 'Tab' && cell.right !== undefined) {
            cell.isSelected = false;
            dispatch('change', {
                cell
            });
            cell.right.isSelected = true;
            event.preventDefault();
        } else if (event.key === 'ArrowDown' && cell.down !== undefined) {
            cell.isSelected = false;
            dispatch('change', {
                cell
            });
            cell.down.isSelected = true;
        } else if (event.key === 'ArrowUp' && cell.up !== undefined) {
            cell.isSelected = false;
            dispatch('change', {
                cell
            });
            cell.up.isSelected = true;
        } else if (event.key === 'ArrowLeft' && cell.left !== undefined) {
            cell.isSelected = false;
            dispatch('change', {
                cell
            });
            cell.left.isSelected = true;
        } else if (event.key === 'ArrowRight' && cell.right !== undefined) {
            cell.isSelected = false;
            dispatch('change', {
                cell
            });
            cell.right.isSelected = true;
        }
    }
</script>

<td>
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div class="cell" on:click={handleClick} class:selected-cell={cell.isSelected}>
    {#if cell.isSelected}
        <!-- svelte-ignore a11y-autofocus -->
        <!-- svelte-ignore a11y-autofocus -->
        <input type="text"
            bind:value={cell.rawValue}
            on:blur={handleBlur}
            on:keydown={handleKeydown}
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
        text-align: right;
        overflow: hidden;
        box-sizing: border-box;
    }

    .selected-cell {
        border: 2px solid blue;
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
