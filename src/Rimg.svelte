<script>
    // @ts-nocheck
    import { get } from "svelte/store";
    import { guidelines } from "./guidelinesStore";
    import { onMount, onDestroy, createEventDispatcher } from "svelte";
    import Moveable from "moveable";

    export let src;

    let moveableobj;
    let moveableInstance;

    const dispatch = createEventDispatcher();

    onMount(() => {
        guidelines.set([...document.querySelectorAll(".moveable")]);

        moveableInstance = new Moveable(document.body, {
            target: moveableobj,
            container: document.body,
            snappable: true,
            draggable: true,
            resizable: true,
            scalable: true,
            keepRatio: true,
            snapThreshold: 200,
            elementGuidelines: get(guidelines),
        });

        moveableInstance
            .on("drag", ({ target, left, top }) => {
                target.style.left = `${left}px`;
                target.style.top = `${top}px`;
            })
            .on("resize", ({ target, width, height, delta }) => {
                if (delta[0]) target.style.width = `${width}px`;
                if (delta[1]) target.style.height = `${height}px`;
            });

        guidelines.subscribe((value) => {
            moveableInstance.elementGuidelines = value;
        });
    });

    onDestroy(() => {
        if (moveableInstance) {
            moveableInstance.destroy();
            moveableInstance = null;
        }
    });

    const handleclick = (e) => {
        e.stopPropagation();
    };
</script>

<img
    {src}
    alt=""
    class="moveable"
    bind:this={moveableobj}
    style="position:absolute;"
    on:click={handleclick}
/>

<style>
    .moveable {
        width: fit-content;
    }
</style>
