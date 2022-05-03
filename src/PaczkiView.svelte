<script>
    import { default as module } from "./external/paczki_plusplus.js";
    import { onMount } from "svelte";
    import { writable } from "svelte/store";
    import { get } from "svelte/store";

    let canvas;

    let module_proper;

    const active_packet = writable(null);
    const box_type_list = writable(null);
    const box_pos_list = writable(null);
    const pallet_id_list = writable(null);
    const sku_list = writable(null);
    // $: apv = $active_packet;

    onMount(() => {
        module({ canvas: canvas }).then((self) => {
            module_proper = self;
            module_proper.addStore("active_packet", active_packet);
            module_proper.addStore("box_type_list", box_type_list);
            module_proper.addStore("box_pos_list", box_pos_list);
            module_proper.addStore("pallet_id_list", pallet_id_list);
            module_proper.addStore("sku_list", sku_list);
        });
    });
</script>

{#if $pallet_id_list}
    Wybór palety: 
    <select
        on:change={(e) => {
            console.log("Selected pallet " + e.target.value);
            module_proper.selectPallet(e.target.value);
        }}
    >
        {#each $pallet_id_list as pallet}
            <option>{pallet}</option>
        {/each}
    </select>
{/if}

<canvas bind:this={canvas} oncontextmenu="return false" id="canvas" />
<br />
<button
    on:click={() => {
        module_proper.alertMessage();
    }}>Display alert</button
>
{#if $active_packet}
    <h1>Active packet: {$active_packet}</h1>
{/if}
