<script lang="ts">
  import { onMount } from 'svelte';
  import '@mediapipe/face_mesh';
  import '@mediapipe/camera_utils';

  let videoEl: HTMLVideoElement;
  let canvasEl: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D;

  let started = false;
  let distortion = 5;
  let speed = 10;
  let frameCount = 0;

  let faceMesh: any;

  const drawPart = (sx, sy, sw, sh, dx, dy, alpha = 1) => {
    const imageData = ctx.getImageData(sx, sy, sw, sh);
    ctx.globalAlpha = alpha;
    ctx.putImageData(imageData, dx, dy);
    ctx.globalAlpha = 1.0;
  };

  const onResults = (results) => {
    if (!results.multiFaceLandmarks || results.multiFaceLandmarks.length === 0) return;

    const landmarks = results.multiFaceLandmarks[0];
    const width = canvasEl.width;
    const height = canvasEl.height;

    ctx.clearRect(0, 0, width, height);
    ctx.drawImage(videoEl, 0, 0, width, height);

    const gridSize = 5;

    for (let y = 0; y < height; y += gridSize) {
      for (let x = 0; x < width; x += gridSize) {
        const dx = Math.floor(
          x + Math.sin((frameCount + x + y) / speed) * distortion
        );
        const dy = Math.floor(
          y + Math.cos((frameCount + x + y) / speed) * distortion
        );
        const alpha = 1 - Math.sqrt((x - width / 2) ** 2 + (y - height / 2) ** 2) / (width / 1.5);
        drawPart(x, y, gridSize, gridSize, dx, dy, alpha);
      }
    }
  };

  const initCamera = () => {
    started = true;

    faceMesh = new window.FaceMesh({
      locateFile: (file: string) =>
        `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`,
    });

    faceMesh.setOptions({
      maxNumFaces: 1,
      refineLandmarks: true,
      minDetectionConfidence: 0.5,
      minTrackingConfidence: 0.5,
    });

    faceMesh.onResults(onResults);

    navigator.mediaDevices.getUserMedia({ video: true }).then((stream) => {
      videoEl.srcObject = stream;
      videoEl.play();

      const camera = new window.Camera(videoEl, {
        onFrame: async () => {
          await faceMesh.send({ image: videoEl });
          frameCount++;
        },
        width: 640,
        height: 480,
      });

      camera.start();
    });
  };

  onMount(() => {
    ctx = canvasEl.getContext('2d', { willReadFrequently: true });
  });
</script>

<style>
  video {
    display: none;
  }

  canvas {
    width: 100%;
    height: auto;
    aspect-ratio: 4 / 3;
    border: 1px solid #aaa;
  }

  .controls {
    margin-top: 1rem;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  label {
    font-size: 14px;
  }
</style>

{#if !started}
  <button on:click={initCamera}>開始</button>
{/if}

<video
  autoplay
  playsinline
  muted
  width="640"
  height="480"
></video>

<canvas bind:this={canvasEl} width="640" height="480"></canvas>

<div class="controls">
  <label>ズレは: {distortion}</label>
  <input type="range" min="0" max="8" bind:value={distortion} />
  <label>速度: {speed}</label>
  <input type="range" min="7" max="18" bind:value={speed} />
</div>