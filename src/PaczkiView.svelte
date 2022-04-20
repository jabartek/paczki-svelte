<script>
    import { default as module } from "./paczki_plusplus.js";
    import { onMount } from "svelte";
    import { writable } from "svelte/store";
    import { get } from "svelte/store";

    let canvas;

    let module_proper;

    const active_packet = writable(0);
    $: apv = $active_packet;

    onMount(() => {
        module({ canvas: canvas }).then((self) => {
            module_proper = self;
            module_proper.addStore("active_packet", active_packet);
        });
    });
</script>

<canvas bind:this={canvas} oncontextmenu="return false" id="canvas" />

<button
    on:click={() => {
        module_proper.alertMessage();
    }}>Oh no!</button
>

<h1>{apv}</h1>
