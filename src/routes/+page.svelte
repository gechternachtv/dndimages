<script>
  import { Canvas, Image as FabricImage } from "fabric";
  import { onMount } from "svelte";
  import { listen } from "@tauri-apps/api/event";
  import { convertFileSrc } from "@tauri-apps/api/core";
  import { open } from "@tauri-apps/plugin-dialog";

  const pickopenfile = async () => {
    const file = await open({
      multiple: true,
      directory: true,
    });
    return file;
  };

  onMount(async () => {
    const canvas = new Canvas("can", {
      selection: true,
      preserveObjectStacking: true,
      selectionColor: "rgba(0, 120, 215, 0.3)",
      selectionBorderColor: "rgba(0, 120, 215, 1)",
      selectionLineWidth: 1,
      selectionDashArray: [4, 2],
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
        console.log(e);

        const pointer = canvas.getPointer({
          clientX: e.payload.position.x,
          clientY: e.payload.position.y,
        });

        let currentX = pointer.x;
        let currentY = pointer.y;

        const tryGrid = e.payload.paths?.length > 3;
        console.log("drag-drop event", e);
        // const maxWidth = canvas.getWidth() / canvas.getZoom();

        for (let i = 0; i < e.payload.paths.length; i++) {
          const url = convertFileSrc(e.payload.paths[i]);
          FabricImage.fromURL(url)
            .then((img) => {
              if (i === 0) {
                currentX = currentX - img.width / 2;
                currentY = currentY - img.height / 2;
              }
              // if (img.width > maxWidth) {
              //   const scale = maxWidth / img.width;
              //   img.scaleX = scale;
              //   img.scaleY = scale;
              // }

              canvas.add(img);
              img.lockRotation = true;
              img.setControlsVisibility({ mtr: false });
              img.set({
                left: currentX,
                top: currentY,
                originX: "left",
                originY: "top",
              });

              img.setCoords();
              canvas.renderAll();
              canvas.setActiveObject(img);
              currentX += img.getScaledWidth();
            })

            .catch((err) => console.error(err));
        }
      } catch (error) {
        console.warn(error);
      }
    });

    window.addEventListener("keydown", function (e) {
      if (e.ctrlKey && e.key === "s") {
        e.preventDefault();
        const allObjects = canvas.getObjects();
        for (let i = 0; i < allObjects.length; i++) {
          console.log(allObjects[i]._element.src);
        }
      }
    });

    window.addEventListener("keydown", (e) => {
      if (e.key === "Delete" || e.key === "Backspace") {
        const selectedObjects = canvas.getActiveObjects();
        console.log(selectedObjects); //
        for (let i = 0; i < selectedObjects.length; i++) {
          canvas.remove(selectedObjects[i]);
        }
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
    let isMoving = false;
    let lastPosX = 0;
    let lastPosY = 0;

    window.addEventListener("mousedown", (e) => {
      //canvas.on("mouse:down", (opt) => {
      if (e.button === 1) {
        console.log("middle");
        isMoving = true;
        lastPosX = e.clientX;
        lastPosY = e.clientY;
        canvas.selection = false;
      }
    });
    window.addEventListener("mouseup", (e) => {
      //canvas.on("mouse:up", (opt) => {
      if (e.button === 1) {
        console.log("middle");
        isMoving = false;
        canvas.selection = true;
      }
    });

    canvas.on("mouse:move", function (opt) {
      if (isMoving) {
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
  <div class="title-thing">
    dnd image collection <button on:click={pickopenfile}>open</button>
  </div>
  <canvas id="can"></canvas>
</main>

<style>
  .container {
    background-color: rgba(255, 255, 255, 0.014);
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
    z-index: 2;
  }

  canvas {
    display: block;
    width: 100vw;
    height: 100vh;
  }
</style>
