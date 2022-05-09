<script>
    import { default as paczki_plusplus } from "./external/paczki_plusplus.js";
    import { default as packet_viewer } from "./external/packet_viewer.js";
    import { onMount } from "svelte";
    import { writable } from "svelte/store";
    import { get } from "svelte/store";

    let paczki_plusplus_canvas;
    let packet_viewer_canvas;

    let paczki_plusplus_proper;
    let packet_viewer_proper;

    const active_packet = writable(null);
    const box_type_list = writable(null);
    const box_pos_list = writable(null);
    const pallet_id_list = writable(null);
    const sku_list = writable(null);
    // $: apv = $active_packet;

    onMount(() => {
        paczki_plusplus({ canvas: paczki_plusplus_canvas }).then((self) => {
            paczki_plusplus_proper = self;
            paczki_plusplus_proper.addStore("active_packet", active_packet);
            paczki_plusplus_proper.addStore("box_type_list", box_type_list);
            paczki_plusplus_proper.addStore("box_pos_list", box_pos_list);
            paczki_plusplus_proper.addStore("pallet_id_list", pallet_id_list);
            paczki_plusplus_proper.addStore("sku_list", sku_list);
        });

        packet_viewer({ canvas: packet_viewer_canvas }).then((self) => {
            packet_viewer_proper = self;
        });
    });
</script>

{#if $pallet_id_list}
    Wybór palety:
    <select
        on:change={(e) => {
            console.log("Selected pallet " + e.target.value);
            paczki_plusplus_proper.selectPallet(e.target.value);
        }}
    >
        {#each $pallet_id_list as pallet}
            <option>{pallet}</option>
        {/each}
    </select>
{/if}

<canvas
    bind:this={paczki_plusplus_canvas}
    oncontextmenu="return false"
    id="canvas"
/>
<canvas
    bind:this={packet_viewer_canvas}
    oncontextmenu="return false"
    id="canvas2"
/>
<br />
<button
    on:click={() => {
        paczki_plusplus_proper.alertMessage();
    }}>Display alert</button
>
{#if $active_packet}
    <h1>Active packet: {$active_packet}</h1>
{/if}

<p>{$active_packet}</p>
{#if $box_type_list}
    <p>{JSON.stringify($box_type_list)}</p>
{/if}
{#if $box_pos_list}
    <p>{JSON.stringify($box_pos_list)}</p>
{/if}
{#if $pallet_id_list}
    <p>{JSON.stringify($pallet_id_list)}</p>
{/if}
{#if $sku_list}
    <p>{JSON.stringify($sku_list)}</p>
{/if}
