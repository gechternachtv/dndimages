<script>
  // @ts-nocheck
  import { Canvas, Image as FabricImage } from "fabric";
  import { onMount } from "svelte";
  import { listen } from "@tauri-apps/api/event";
  import { convertFileSrc } from "@tauri-apps/api/core";

  onMount(() => {
    const canvas = new Canvas("can", {
      selection: true,
      preserveObjectStacking: true,
    });

    function resizeCanvas() {
      canvas.setWidth(window.innerWidth);
      canvas.setHeight(window.innerHeight);
      canvas.renderAll();
    }
    window.addEventListener("resize", resizeCanvas);
    resizeCanvas();
    listen("tauri://drag-drop", (e) => {
      try {
        let currentX = 100;

        const tryGrid = e.payload.paths?.length > 3;
        console.log("drag-drop event", e);
        e.payload.paths.forEach((droppedEl, i) => {
          const url = convertFileSrc(droppedEl);
          FabricImage.fromURL(url)
            .then((img) => {
              canvas.add(img);
              img.lockRotation = true;
              img.setControlsVisibility({ mtr: false });
              img.set({
                left: currentX,
                top: 100,
                originX: "left",
                originY: "top",
              });

              const maxWidth = canvas.width;
              if (img.width > maxWidth) {
                const scale = maxWidth / img.width;
                img.scaleX = scale;
                img.scaleY = scale;
              }

              img.setCoords();
              canvas.renderAll();

              currentX += img.getScaledWidth();
            })

            .catch((err) => console.error(err));
        });
      } catch (error) {
        console.warn(error);
      }
    });

    canvas.on("mouse:wheel", function (opt) {
      const delta = opt.e.deltaY;
      let zoom = canvas.getZoom();
      zoom *= 0.999 ** delta;
      zoom = Math.min(Math.max(zoom, 0.1), 10);

      canvas.zoomToPoint({ x: opt.e.offsetX, y: opt.e.offsetY }, zoom);

      opt.e.preventDefault();
      opt.e.stopPropagation();
    });
    let isPanning = false;
    let lastPosX = 0;
    let lastPosY = 0;
    window.addEventListener("mousedown", (e) => {
      if (e.button === 1) {
        console.log("middle");
        isPanning = true;
        lastPosX = e.clientX;
        lastPosY = e.clientY;
        canvas.selection = false;
      }
    });
    window.addEventListener("mouseup", (e) => {
      if (e.button === 1) {
        console.log("middle");
        isPanning = false;
        canvas.selection = true;
      }
    });

    canvas.on("mouse:move", function (opt) {
      if (isPanning) {
        const e = opt.e;
        const deltaX = e.clientX - lastPosX;
        const deltaY = e.clientY - lastPosY;
        lastPosX = e.clientX;
        lastPosY = e.clientY;
        canvas.relativePan({ x: deltaX, y: deltaY });
      }
    });
    //onmount
  });
</script>

<main class="container">
  <div class="title-thing">dnd image collection</div>
  <canvas id="can" width="1200" height="800"></canvas>
</main>

<style>
  .container {
    background: transparent;
  }
  .title-thing {
    font-family: "MingLiU";
    background: rgb(65, 12, 12);
    color: white;
    width: 100vw;
    position: fixed;
    bottom: 0;
    left: 0;
    font-size: 11px;
    letter-spacing: 3px;
  }

  canvas {
    display: block;
    width: 100vw;
    height: 100vh;
  }
</style>
