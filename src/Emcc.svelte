<script>
    import { default as module } from "./paczki_plusplus.js";
    import { onMount } from "svelte";

    let canvas;

    let module_proper;

    onMount(() => {
        let module_promise = module({ canvas: canvas });
        module_promise.then((self) => {
            module_proper = self;
        });
    });

    var async = function (func) {
        return function () {
            var args = arguments;
            setTimeout(function () {
                func.apply(this, args);
            }, 0);
        };
    };
</script>

<canvas bind:this={canvas} />

<button
    on:click={() => {
        module_proper.alertMessage();
    }}>Oh no!</button
>
