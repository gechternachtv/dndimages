<script lang="ts">
    //@ts-nocheck
    import { onMount, onDestroy } from "svelte";
    import { listen } from "@tauri-apps/api/event";
    import { convertFileSrc } from "@tauri-apps/api/core";
    import { fabric } from "fabric";

    let canvasEl: HTMLCanvasElement;
    let canvas: fabric.Canvas;

    let snapEnabled = true;
    let snapThreshold = 10;

    function addImageFromURL(
        url: string,
        opts: {
            left?: number;
            top?: number;
            isGif?: boolean;
            width?: number;
        } = {},
    ) {
        fabric.Image.fromURL(
            url,
            (img) => {
                img.set({
                    left: opts.left ?? 100,
                    top: opts.top ?? 100,
                    originX: "left",
                    originY: "top",
                    cornerStyle: "rect",
                    transparentCorners: false,
                });
                img.setControlsVisibility({ mtr: true });
                img.set({
                    selectable: true,
                    hasBorders: true,
                    hasControls: true,
                });
                img.scaleToWidth(opts.width ?? 200);
                img.set("lockUniScaling", false);
                canvas.add(img);
                canvas.setActiveObject(img);
                if (/\.gif$/i.test(url) || opts.isGif) img._isGif = true;
                canvas.requestRenderAll();
            },
            { crossOrigin: "anonymous" },
        );
    }

    function snapObjectToOthers(target: fabric.Object) {
        if (!snapEnabled) return;
        const objects = canvas.getObjects().filter((o) => o !== target);
        const threshold = snapThreshold;

        const tLeft = target.left!;
        const tTop = target.top!;
        const tRight = tLeft + target.getScaledWidth();
        const tBottom = tTop + target.getScaledHeight();
        const tCenterX = tLeft + target.getScaledWidth() / 2;
        const tCenterY = tTop + target.getScaledHeight() / 2;

        for (const o of objects) {
            const oLeft = o.left!;
            const oTop = o.top!;
            const oRight = oLeft + o.getScaledWidth();
            const oBottom = oTop + o.getScaledHeight();
            const oCenterX = oLeft + o.getScaledWidth() / 2;
            const oCenterY = oTop + o.getScaledHeight() / 2;

            if (Math.abs(tLeft - oRight) <= threshold) target.left = oRight;
            if (Math.abs(tRight - oLeft) <= threshold)
                target.left = oLeft - target.getScaledWidth();
            if (Math.abs(tTop - oBottom) <= threshold) target.top = oBottom;
            if (Math.abs(tBottom - oTop) <= threshold)
                target.top = oTop - target.getScaledHeight();
            if (Math.abs(tCenterX - oCenterX) <= threshold)
                target.left = oCenterX - target.getScaledWidth() / 2;
            if (Math.abs(tCenterY - oCenterY) <= threshold)
                target.top = oCenterY - target.getScaledHeight() / 2;
        }
    }

    function hasGifObjects() {
        return canvas.getObjects().some((o) => (o as any)._isGif);
    }

    let animationFrameId: number;

    function animationLoop() {
        if (hasGifObjects()) canvas.requestRenderAll();
        animationFrameId = requestAnimationFrame(animationLoop);
    }

    onMount(() => {
        canvas = new fabric.Canvas(canvasEl, {
            selection: true,
            preserveObjectStacking: true,
        });
        canvas.setBackgroundColor("#fff", canvas.renderAll.bind(canvas));

        canvas.on("object:moving", (e) => snapObjectToOthers(e.target!));
        canvas.on("object:scaling", (e) => {
            const obj = e.target!;
            const w = obj.getScaledWidth();
            const h = obj.getScaledHeight();
            if (obj.left! + w > canvas.width!) obj.left = canvas.width! - w;
            if (obj.top! + h > canvas.height!) obj.top = canvas.height! - h;
        });

        document.addEventListener("keydown", (e) => {
            if (e.key === "Delete" || e.key === "Backspace") {
                const active = canvas.getActiveObjects();
                if (active.length) {
                    active.forEach((o) => canvas.remove(o));
                    canvas.discardActiveObject();
                    canvas.requestRenderAll();
                }
            }
        });

        canvas.on("mouse:dblclick", (e) => {
            if (e.target) canvas.bringToFront(e.target);
        });

        canvas.on("mouse:wheel", function (opt) {
            const delta = opt.e.deltaY;
            let zoom = canvas.getZoom();
            zoom *= 0.999 ** delta;
            if (zoom > 5) zoom = 5;
            if (zoom < 0.1) zoom = 0.1;
            canvas.zoomToPoint({ x: opt.e.offsetX, y: opt.e.offsetY }, zoom);
            opt.e.preventDefault();
            opt.e.stopPropagation();
        });

        // Listen to Tauri drag-drop events
        const unlistenPromise = listen("tauri://drag-drop", async (e) => {
            try {
                if (e.payload.paths && e.payload.paths.length) {
                    for (const path of e.payload.paths) {
                        const imgsrc = await convertFileSrc(path);
                        addImageFromURL(imgsrc, {
                            left: 100,
                            top: 100,
                            isGif: path.toLowerCase().endsWith(".gif"),
                        });
                    }
                }
            } catch (error) {
                console.warn("Failed to load dropped image:", error);
            }
        });

        animationLoop();

        onDestroy(() => {
            cancelAnimationFrame(animationFrameId);
            canvas.dispose();
            unlistenPromise.then((unlisten) => unlisten());
        });
    });
</script>

<div class="app">
    <div class="sidebar">
        <h3>FabricJS + Tauri drag-drop (Svelte)</h3>
        <p>
            Drop images from your OS onto the app window. Images are draggable,
            resizable, snap to others, zoom with mouse wheel.
        </p>

        <label>
            <input type="checkbox" bind:checked={snapEnabled} />
            Enable snapping
        </label>

        <label>
            Snap threshold:
            <input type="range" min="2" max="40" bind:value={snapThreshold} />
            {snapThreshold}
        </label>

        <div
            class="drop-hint"
            style="margin-top:12px; padding:10px; border:1px dashed #ddd; border-radius:6px; background:#fff; text-align:center;"
        >
            Drag and drop files from your desktop onto the app
        </div>
    </div>

    <div class="canvas-wrap">
        <canvas bind:this={canvasEl} width="1200" height="700"></canvas>
    </div>
</div>

<style>
    :global(body, html) {
        margin: 0;
        height: 100%;
        font-family:
            system-ui,
            Segoe UI,
            Roboto,
            Arial,
            sans-serif;
    }
    .app {
        display: flex;
        height: 100vh;
    }
    .sidebar {
        width: 260px;
        padding: 12px;
        border-right: 1px solid #eee;
        background: #fafafa;
    }
    .canvas-wrap {
        flex: 1;
        display: flex;
        align-items: stretch;
        justify-content: center;
        padding: 12px;
    }
    canvas {
        border: 1px solid #ccc;
        box-shadow: 0 4px 20px rgba(0, 0, 0, 0.05);
    }
    label {
        font-size: 13px;
        color: #333;
        display: block;
        margin-bottom: 8px;
    }
</style>
