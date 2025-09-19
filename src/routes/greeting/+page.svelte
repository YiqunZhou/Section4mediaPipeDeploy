<!-- Why Tween Causes Errors in {#each}
When using a store inside {#each}, Svelte expects subscribe() support.
Tween does NOT implement subscribe() → This is why Svelte throws "store_invalid_shape".
The {#each} loop tries to access jiggleX and jiggleY as stores:
svelte
复制
编辑
style="left: {$jiggleX[index]}%; top: {$jiggleY[index]}%"
Since Tween doesn’t support $storeName syntax, it throws an error. -->

<!-- Best Fix: Using Tween Correctly
If you want to keep using Tween instead of tweened(), you should not use $jiggleX[index] inside {#each}.

Instead, update the character's position manually in a reactive statement ($:) like this:

svelte
复制
编辑
{#each characters as char, index}
    <img class="character"
        src="{char.src}" 
        alt="Character"
        style="left: {jiggleX[index].current}%; top: {jiggleY[index].current}%; width: {char.width}%" />
{/each}
🔥 Why This Fix Works
No $jiggleX[index] subscription needed.
Directly accessing .current like your working example.
Avoids the "store_invalid_shape" error. -->

<!-- Tween is the correct form, it's really annoying about it's storage -->

<script>
    import { onMount } from "svelte";
    import { Tween } from "svelte/motion";
    import { cubicInOut } from "svelte/easing";
  
    let videoElement;
    // Remove TensorFlow/handpose related variables
    let isRunning = true;
  
    // Existing Tween and other state variables…
    let handCursorX = new Tween(0, { duration: 50, easing: cubicInOut });
    let handCursorY = new Tween(0, { duration: 50, easing: cubicInOut });
  
    const images = [
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1739269717/park1_ve6mqi.jpg",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1739269717/park2_snaghz.jpg",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1739269717/park3_wrf6dz.jpg"
    ];
  
    let characters = [
      { src: "https://res.cloudinary.com/dufnjfidt/image/upload/v1739273801/cha2_xyy7uz.png", x: 50, y: 10, width: 20 },
      { src: "https://res.cloudinary.com/dufnjfidt/image/upload/v1739273801/cha3_q5qvmj.png", x: 80, y: 30, width: 25 },
      { src: "https://res.cloudinary.com/dufnjfidt/image/upload/v1739273801/cha4_oe0vbk.png", x: 10, y: 55, width: 25 },
      { src: "https://res.cloudinary.com/dufnjfidt/image/upload/v1739273801/cha1_iuedwe.png", x: 50, y: 65, width: 22 }
    ];
  
    let jiggleX = characters.map(char => new Tween(char.x, { duration: 100, easing: cubicInOut }));
    let jiggleY = characters.map(char => new Tween(char.y, { duration: 100, easing: cubicInOut }));
  
    let handCursorSrc = "https://res.cloudinary.com/dufnjfidt/image/upload/v1739276276/hand_du20n8.png";
  
    // Declare a variable for the MediaPipe Hands instance
    let mediapipeHands;
  
    onMount(async () => {
      // Import and set up MediaPipe Hands
      const { Hands } = await import("@mediapipe/hands");
      mediapipeHands = new Hands({
        locateFile: (file) =>
          `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
      });
      mediapipeHands.setOptions({
        maxNumHands: 2,
        modelComplexity: 1,
        minDetectionConfidence: 0.5,
        minTrackingConfidence: 0.5,
      });
  
      // When MediaPipe returns results, process the landmarks
      mediapipeHands.onResults((results) => {
        if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
          // Use the first detected hand (or loop over all if needed)
          processHand(results.multiHandLandmarks[0]);
        }
      });
  
      await setupCamera();
      detectHands();
    });
  
    async function setupCamera() {
      const stream = await navigator.mediaDevices.getUserMedia({ video: true });
      if (!videoElement) {
        console.error("🚨 videoElement is not defined yet!");
        return;
      }
      videoElement.srcObject = stream;
      await new Promise((resolve) => {
        videoElement.onloadedmetadata = () => {
          videoElement.play();
          resolve();
        };
      });
      console.log("🎥 Camera successfully started!");
    }
  
    async function detectHands() {
      if (!videoElement) {
        requestAnimationFrame(detectHands);
        return;
      }
      // Send the current video frame to MediaPipe Hands
      await mediapipeHands.send({ image: videoElement });
      if (isRunning) {
        requestAnimationFrame(detectHands);
      }
    }
  
    // Updated processHand: converts MediaPipe’s normalized landmarks to pixel coordinates
    function processHand(landmarks) {
      if (!landmarks) return;
      const videoWidth = videoElement.videoWidth;
      const videoHeight = videoElement.videoHeight;
      // Convert each landmark: multiply normalized values by video dimensions and mirror X
      const mirroredLandmarks = landmarks.map(({ x, y, z }) => [
        videoWidth - (x * videoWidth),
        y * videoHeight,
        z
      ]);
  
      // Use landmark 12 as in your original code (e.g. middle finger knuckle)
      const scaleX = window.innerWidth / 640;
      const scaleY = window.innerHeight / 480;
      let newHandX = mirroredLandmarks[12][0] * scaleX;
      let newHandY = mirroredLandmarks[12][1] * scaleY + window.scrollY;
  
      handCursorX.set(newHandX);
      handCursorY.set(newHandY);
  
      checkCollision(newHandX, newHandY);
      // Update gesture detection if needed – passing in the mirrored landmarks
      detectGestureAndScroll({ landmarks: mirroredLandmarks });
    }
  
    // The rest of your functions (detectGestureAndScroll, checkCollision, triggerJiggle) remain unchanged.
    function detectGestureAndScroll(hand) {
      if (!hand.landmarks) return;
      const indexTip = hand.landmarks[8];  
      const indexKnuckle = hand.landmarks[6];  
      const middleKnuckle = hand.landmarks[12]; 
  
      const tipY = indexTip[1];
      const knuckleY = indexKnuckle[1];
      const middleY = middleKnuckle[1];
  
      const scrollSpeed = 3;
      if (tipY < knuckleY && knuckleY < middleY) {
        window.scrollBy({ top: -scrollSpeed });
        console.log("⬆ Gesture: Scrolling Up...");
      } else if (tipY > knuckleY) {
        window.scrollBy({ top: scrollSpeed });
        console.log("⬇ Gesture: Scrolling Down...");
      }
    }
  
    function checkCollision(handX, handY) {
      characters.forEach((char, index) => {
        let charWidth = (char.width / 100) * window.innerWidth;
        let charHeight = charWidth; 
        let charX = (jiggleX[index].current / 100) * window.innerWidth;
        let charY = (jiggleY[index].current / 100) * window.innerHeight + window.scrollY;
        let detectionBoxX = charX - charWidth / 2;
        let detectionBoxY = charY - charHeight / 2;
        let detectionBoxW = charWidth;
        let detectionBoxH = charHeight;
  
        console.log(`🖐 Hand at X=${handX}, Y=${handY}`);
        console.log(`🎯 Character ${index} at X=${charX}, Y=${charY}`);
        console.log(`🟦 Detection Box for Character ${index}: X=${detectionBoxX}, Y=${detectionBoxY}, W=${detectionBoxW}, H=${detectionBoxH}`);
  
        if (
          handX > detectionBoxX &&
          handX < detectionBoxX + detectionBoxW &&
          handY > detectionBoxY &&
          handY < detectionBoxY + detectionBoxH
        ) {
          console.log(`✅ Collision detected with character ${index} (${char.src})`);
          triggerJiggle(index);
        }
      });
    }
  
    async function triggerJiggle(index) {
      console.log(`Jiggling character ${index}`);
      let originalX = characters[index].x;
      let originalY = characters[index].y;
      await jiggleX[index].set(originalX + 1);
      await new Promise(resolve => setTimeout(resolve, 100));
      await jiggleX[index].set(originalX - 1);
      await new Promise(resolve => setTimeout(resolve, 100));
      await jiggleX[index].set(originalX);
      await jiggleY[index].set(originalY + 1);
      await new Promise(resolve => setTimeout(resolve, 100));
      await jiggleY[index].set(originalY - 1);
      await new Promise(resolve => setTimeout(resolve, 100));
      await jiggleY[index].set(originalY);
    }
  </script>
  
  <style>
    html, body {
      margin: 0;
      padding: 0;
      overflow-x: hidden;
    }
  
    /* ✅ Background container with 3 images stacked */
    .container {
      width: 100vw;
      height: 300vh;
      display: flex;
      flex-direction: column;
      position: absolute;
      top: 0;
      left: 0;
    }
  
    /* ✅ Ensures each image section takes the full viewport height */
    .image-wrapper {
      width: 100vw;
      height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
    }
  
    .image {
      max-width: 100vw;
      max-height: 100vh;
      width: auto;
      height: auto;
      display: block;
    }
  
    .character {
      position: absolute;
      height: auto;
    }
  
    .hand-cursor {
      position: absolute;
      width: 50px;
      height: auto;
      pointer-events: none;
    }
  
    .video-container {
      position: fixed; /* ✅ Ensures video moves with scroll */
      width: 100vw;
      height: 100vh; /* Same height as the page */
  }
  
  video {
      position: absolute; /* ✅ Stays inside the scrolling container */
      width: 100%;
      height: 100%;
      opacity: 0.1;
  }
  
  </style>
  
  <!-- ✅ Background images -->
  <div class="container">
    {#each images as image}
      <div class="image-wrapper">
        <img class="image" src="{image}" alt="Park Image" />
      </div>
    {/each}
  
    <!-- ✅ Hand Cursor -->
    <img class="hand-cursor" src="{handCursorSrc}" alt="Hand Cursor"
         style="left: {handCursorX.current}px; top: {handCursorY.current}px;" />
  
    <!-- ✅ Characters -->
    {#each characters as char, index}
        <img class="character"
             src="{char.src}" 
             alt="Character"
             style="left: {jiggleX[index].current}%; top: {jiggleY[index].current}%; width: {char.width}%" />
    {/each}
  </div>
  
  <!-- Hidden video feed for detection -->
  <div class="video-container">
      <video bind:this={videoElement} autoplay></video>
  </div>
  
  <!-- Stop detection button -->
  <button on:click={() => isRunning = false}>Stop Detection</button>
  