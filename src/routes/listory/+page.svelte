<script>
  import { onMount } from 'svelte';

  // State variables
  let showBody = false;
  let bodyImageSize = '5vw';
  let templeImageSize = '50vw';
  let videoElement;
  let showBlackOverlay = false;
  let showUpdownGif = false;  // Dedicated flag for the up–down GIF during black overlay
  let gooseLeftX = 0, gooseLeftY = 0;
  let gooseRightX = 0, gooseRightY = 0;
  let wipeCanvas;
  let wipeCtx;
  let whiteComplete = false;
  let ozoneLeftX = 0, ozoneLeftY = 0;
  let ozoneRightX = 0, ozoneRightY = 0;
  const videoWidth = 640;
  const videoHeight = 480;
  let mediapipeHands;

  // Audio objects
  let audio;   // leopard1
  let audio2;  // leopard2
  let audio3;  // leopard3
  let audio2Played = false; // ensure audio2 plays only once
  let audio3Played = false; // ensure audio3 plays only once
  let audio1Finished = false; // flag: audio1 must finish before triggering audio2 & overlay
  let audio2Finished = false; // flag: white check only begins after audio2 finishes

  // When user clicks the dialog, switch view and play audio1.
  function handleDialogClick() {
    showBody = true;
    audio.play().catch((error) => {
      console.error("Audio playback failed:", error);
    });
  }

  // Utility: Euclidean distance between two points.
  function computeDistance(x1, y1, x2, y2) {
    return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
  }

  // Convert a normalized x coordinate into a flipped pixel coordinate.
  function convertX(normalizedX) {
    return (1 - normalizedX) * videoWidth;
  }

  // Check if specific hand joints are roughly on the same horizontal line.
  function checkHorizontalAlignment(hand1, hand2) {
    const y1PIP = hand1[10].y * videoHeight;
    const y1TIP = hand1[12].y * videoHeight;
    const y2PIP = hand2[10].y * videoHeight;
    const y2TIP = hand2[12].y * videoHeight;
    const allYs = [y1PIP, y1TIP, y2PIP, y2TIP];
    const maxY = Math.max(...allYs);
    const minY = Math.min(...allYs);
    return (maxY - minY) < 15; // pixels threshold
  }

  // Update goose positions based on hand landmarks.
  function updateGoosePositions(hand1, hand2) {
    const x1PIP = convertX(hand1[10].x);
    const y1PIP = hand1[10].y * videoHeight;
    const x2PIP = convertX(hand2[10].x);
    const y2PIP = hand2[10].y * videoHeight;
    if (x1PIP < x2PIP) {
      gooseLeftX = x1PIP;
      gooseLeftY = y1PIP;
      gooseRightX = x2PIP;
      gooseRightY = y2PIP;
    } else {
      gooseLeftX = x2PIP;
      gooseLeftY = y2PIP;
      gooseRightX = x1PIP;
      gooseRightY = y1PIP;
    }
  }

  // Draw a white circle onto the wipe canvas.
  function drawWipe(x, y, radius) {
    if (!wipeCtx) return;
    wipeCtx.beginPath();
    wipeCtx.arc(x, y, radius, 0, Math.PI * 2);
    wipeCtx.fillStyle = 'white';
    wipeCtx.fill();
    wipeCtx.closePath();
  }

  // Check percentage of the wipe canvas that is white.
  function checkWhiteCoverage() {
    if (!wipeCtx) return 0;
    const imageData = wipeCtx.getImageData(0, 0, wipeCanvas.width, wipeCanvas.height);
    let whitePixels = 0;
    const totalPixels = imageData.data.length / 4;
    for (let i = 0; i < imageData.data.length; i += 4) {
      if (
        imageData.data[i] === 255 &&
        imageData.data[i + 1] === 255 &&
        imageData.data[i + 2] === 255 &&
        imageData.data[i + 3] > 200
      ) {
        whitePixels++;
      }
    }
    return whitePixels / totalPixels;
  }

  // Handle MediaPipe Hands results.
  function handleResults(results) {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length === 2) {
      const hand1 = results.multiHandLandmarks[0];
      const hand2 = results.multiHandLandmarks[1];

      // Scaling calculations for body and temple images:
      const x1_10 = convertX(hand1[10].x);
      const y1_10 = hand1[10].y * videoHeight;
      const x2_10 = convertX(hand2[10].x);
      const y2_10 = hand2[10].y * videoHeight;
      const distance = computeDistance(x1_10, y1_10, x2_10, y2_10);
      
      // Update body image size.
      const bodySizeVW = (distance * 200) / videoWidth;
      bodyImageSize = `${bodySizeVW}vw`;
      
      // Update temple image size.
      const minDist = 0.5 * videoWidth;
      const maxDist = 1.0 * videoWidth;
      let templeSizeVW;
      if (distance <= minDist) {
        templeSizeVW = 50;
      } else if (distance >= maxDist) {
        templeSizeVW = 300;
      } else {
        const ratio = (distance - minDist) / (maxDist - minDist);
        templeSizeVW = 50 + 250 * ratio;
      }
      templeImageSize = `${templeSizeVW}vw`;

      // Only trigger black overlay and audio2 after audio1 is finished.
      if (audio1Finished && showBody && (showBlackOverlay || checkHorizontalAlignment(hand1, hand2))) {
        if (!audio2Played) {
          audio2.play().catch((error) => {
            console.error("Audio2 playback failed:", error);
          });
          audio2Played = true;
        }
        showBlackOverlay = true;
        showUpdownGif = true;
        updateGoosePositions(hand1, hand2);

        const canvasXLeft = (gooseLeftX / videoWidth) * window.innerWidth;
        const canvasYLeft = (gooseLeftY / videoHeight) * window.innerHeight;
        const canvasXRight = (gooseRightX / videoWidth) * window.innerWidth;
        const canvasYRight = (gooseRightY / videoHeight) * window.innerHeight;
        const wipeRadius = 50; // Adjust as desired.
        drawWipe(canvasXLeft, canvasYLeft, wipeRadius);
        drawWipe(canvasXRight, canvasYRight, wipeRadius);

        // Only check white coverage if audio2 has finished playing.
        const coverage = checkWhiteCoverage();
        if (audio2Finished && !whiteComplete && coverage > 0.65) {
          whiteComplete = true;
          wipeCtx.fillStyle = "white";
          wipeCtx.fillRect(0, 0, wipeCanvas.width, wipeCanvas.height);
          ozoneLeftX = gooseLeftX;
          ozoneRightX = gooseRightX;
          ozoneLeftY = gooseLeftY;
          ozoneRightY = gooseRightY;
          // When final images appear, play audio3.
          if (!audio3Played) {
            audio3.play().catch((error) => {
              console.error("Audio3 playback failed:", error);
            });
            audio3Played = true;
          }
        }
      }

      if (whiteComplete) {
        ozoneLeftY = gooseLeftY;
        ozoneRightY = gooseRightY;
      }
    }
  }

  // Main detection loop.
  async function detectHands() {
    if (!videoElement) {
      requestAnimationFrame(detectHands);
      return;
    }
    await mediapipeHands.send({ image: videoElement });
    requestAnimationFrame(detectHands);
  }

  // On mount: set up webcam, MediaPipe, canvas, and initialize audio.
  onMount(async () => {
    // Initialize audio objects.
    audio = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1740500602/leopard1_ceghew.mp3');
    audio2 = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1740501186/leopard2_cuv4mn.mp3');
    audio3 = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1740589054/leopard3_vhxyrh.mp3');

    // When audio1 ends, set flag.
    audio.addEventListener('ended', () => {
      audio1Finished = true;
    });

    // When audio2 ends, set flag so white coverage can be checked.
    audio2.addEventListener('ended', () => {
      audio2Finished = true;
    });

    const stream = await navigator.mediaDevices.getUserMedia({
      video: { width: videoWidth, height: videoHeight }
    });
    videoElement.srcObject = stream;
    await videoElement.play();

    wipeCanvas.width = window.innerWidth;
    wipeCanvas.height = window.innerHeight;
    wipeCtx = wipeCanvas.getContext('2d');

    const { Hands } = await import('@mediapipe/hands');
    mediapipeHands = new Hands({
      locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
    });
    mediapipeHands.setOptions({
      maxNumHands: 2,
      modelComplexity: 1,
      minDetectionConfidence: 0.5,
      minTrackingConfidence: 0.5
    });
    mediapipeHands.onResults(handleResults);
    detectHands();
  });
</script>

<style>
  /* Container and background */
  .container {
    position: fixed;
    top: 0;
    left: 0;
    min-width: 100vw;
    min-height: 100vh;
    overflow-x: hidden;
  }
  .background-image {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
  .dialog {
    position: absolute;
    top: 65%;
    left: 35%;
    transform: translateX(-50%);
    background-color: rgba(255, 255, 255, 0.8);
    padding: 20px;
    border-radius: 8px;
    cursor: pointer;
    transition: background-color 0.3s ease;
  }
  .dialog:hover {
    background-color: rgba(200, 200, 200, 0.8);
  }
  /* Body view styles */
  .body-container {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: white;
    overflow: visible;
  }
  .back-image {
    position: absolute;
    width: 100vw;
    height: 100vh;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 1;
    max-width: none;
  }
  .body-image {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 1;
    max-width: none;
  }
  .temple-image {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 2;
    pointer-events: none;
    max-width: none;
  }
  /* Overlays and canvas */
  .black-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background-color: black;
    z-index: 10;
  }
  .wipe-canvas {
    position: fixed;
    top: 0;
    left: 0;
    z-index: 11;
    pointer-events: none;
  }
  /* Goose and final images */
  .goose-left,
  .goose-right {
    position: fixed;
    z-index: 12;
    width: 30vw;
    height: auto;
    transform: translate(-50%, -50%);
  }
  .leopardbackground-image {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 13;
    object-fit: cover;
  }
  .leopardmiddle-image {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 15;
    object-fit: cover;
  }
  .ozone{
    position: fixed;
    z-index: 14;
    width: 100vw;
    height: auto;
    transform: translate(-50%, -50%);
  }
  .leopard {
    position: fixed;
    z-index: 16;
    width: 100vw;
    height: auto;
    transform: translate(-50%, -50%);
  }
  /* GIFs */
  .click-gif {
    position: absolute;
    bottom: 10px;
    left: 10px;
    width: 7vw;
  }
  .bottom-left-gifs {
    position: absolute;
    bottom: 10px;
    left: 10px;
    gap: 1vw;
    z-index: 15;
  }
  .bottom-left-gifs img {
    width: 10vw;
  }
  /* Up–down GIF styles */
  /* During black overlay phase */
  .updown-gif {
    position: fixed;
    bottom: 10px;
    left: 10px;
    width: 10vw;
    z-index: 14;
    pointer-events: none;
  }
  /* Hide video element */
  video {
    display: none;
  }
</style>

{#if !showBody}
  <div class="container">
    <img
      src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740189975/zhoustart_uyakvw.jpg"
      alt="Zhou"
      class="background-image"
    />
    <!-- Click GIF on background view -->
    <img
      src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740497105/click1_zpl2dp.gif"
      alt="Click Animation"
      class="click-gif"
    />
    <div class="dialog" on:click={handleDialogClick}>
      Actually... I can't really understand the feeling. Can you describe it for me?
    </div>
  </div>
{:else}
  <!-- Body view -->
  <div class="body-container">
    <img
    src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743031676/bodybackground_efnneo.jpg"
    alt="Body"
    class="back-image"
    
  />
    
    <img
      src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743031673/bodyfinishtransparent_okfibt.png"
      alt="Body"
      class="body-image"
      style="width: {bodyImageSize}; height: auto;"
    />
    <img
      src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743031176/bodyinsidefinish_rgvyvx.png"
      alt="Temple"
      class="temple-image"
      style="width: {templeImageSize}; height: auto;"
    />
    <!-- Bottom-left GIFs in body view -->
    <div class="bottom-left-gifs">
      <img
        src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740500992/outin_xsjhma.gif"
        alt="Out In Animation"
      />
      <img
        src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740501359/fold_pudeo2.gif"
        alt="Fold Animation"
      />
    </div>
  </div>
{/if}

<!-- Black overlay -->
{#if showBlackOverlay && !whiteComplete}
  <img 
    src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743036102/gooseback_toevht.jpg" 
    alt="Goose Back" 
    class="background-image" 

  />
    <!-- Text overlay at the bottom right -->
    <div 
    style="
      position: fixed; 
      bottom: 20px; 
      right: 20px; 
      z-index: 11; 
      color: white; 
      font-size: 1.5rem; 
      background: rgba(0, 0, 0, 0.5); 
      padding: 10px; 
      border-radius: 5px;
    "
  >
    Goose leaves no trace, try to wipe the screen blank with the goose.
  </div>
{/if}

<!-- White wipe canvas -->
<canvas bind:this={wipeCanvas} class="wipe-canvas" style="display: block;"></canvas>

<!-- Up–down GIF during black overlay phase -->
{#if showUpdownGif && !whiteComplete}
  <img
    src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740501674/updown_m3psqs.gif"
    alt="Up Down Animation"
    class="updown-gif"
  />
{/if}

<!-- Up–down GIF on final image page -->
{#if whiteComplete}
  <img
    src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740501674/updown_m3psqs.gif"
    alt="Up Down Animation"
    class="updown-gif"
  />
{/if}

<!-- Goose images -->

<img
  src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743036034/gooseleftfinish_ehaeqc.png"
  alt="Goose Left"
  class="goose-left"
  style="display: {showBlackOverlay && !whiteComplete ? 'block' : 'none'};
         left: {(gooseLeftX / videoWidth) * 100}vw;
         top: {(gooseLeftY / videoHeight) * 100}vh;"
/>
<img
  src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743036036/goosefinishright_k3saei.png"
  alt="Goose Right"
  class="goose-right"
  style="display: {showBlackOverlay && !whiteComplete ? 'block' : 'none'};
         left: {(gooseRightX / videoWidth) * 100}vw;
         top: {(gooseRightY / videoHeight) * 100}vh;"
/>

<!-- Final images (ozone and leopard) -->
<img
src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743041495/leopardback_hgtqs7.jpg"
alt="Leopardback"
class="leopardbackground-image"
  style="display: {whiteComplete ? 'block' : 'none'};"

/>

<img
  src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743041325/person_dlxiuv.png"
  alt="Ozone"
  class="ozone"
   style="display: {whiteComplete ? 'block' : 'none'};
         left: 50vw;
         top: {(ozoneLeftY / videoHeight) * 100}vh;"
/>
<img
src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743041653/earth_juqdzy.png"
alt="Leopardback"
class="leopardmiddle-image"
style="display: {whiteComplete ? 'block' : 'none'};"
/>
<img
  src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743041326/leopardfinish_far0be.png"
  alt="Leopard"
  class="leopard"
  style="display: {whiteComplete ? 'block' : 'none'};
         left: 50vw;
         top: {(ozoneRightY / videoHeight) * 100}vh;"
/>

<!-- Hidden video element -->
<video bind:this={videoElement} width="{videoWidth}" height="{videoHeight}" style="display: none;"></video>
