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
            rotatable: true,
            warpable: true,
            pinchable: true,
            origin: true,
            keepRatio: true,
            edge: false,
            throttleDrag: 0,
            throttleResize: 0,
            throttleScale: 0,
            throttleRotate: 0,
            snapThreshold: 200,
            elementGuidelines: get(guidelines),
            elementSnapDirections: {
                top: true,
                left: true,
                bottom: true,
                right: true,
                center: true,
                middle: true,
            },
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
            console.log("Count changed:", value);
            moveableInstance.elementGuidelines = value;
            // Do whatever you want whenever count changes here
        });
    });

    onDestroy(() => {
        if (moveableInstance) {
            moveableInstance.destroy();
            moveableInstance = null;
        }
    });
</script>

<img
    {src}
    alt=""
    class="moveable"
    bind:this={moveableobj}
    style="position:absolute;"
/>

<style>
    .moveable {
        width: fit-content;
    }
</style>
