<script>
    let isEditing = false;
    let value = '';
    let rawValue = '';

    function handleClick() {
        isEditing = true;
    }

    function handleBlur() {
        isEditing = false;
    }
</script>

<div class="cell" on:click={handleClick}>
    {#if isEditing}
        <input type="text" bind:value={value} on:blur={handleBlur} autofocus>
    {:else}
        {value}
    {/if}
</div>

<style>
    .cell {
        position: relative;
        border: 1px solid #ccc;
        padding: 5px;
        width: 180px;  /* 200pxのセルサイズからpaddingやborderを引いたサイズ */
        height: 30px;
        line-height: 30px;
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
