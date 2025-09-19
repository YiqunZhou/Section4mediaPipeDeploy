<script>
    import { onMount, onDestroy } from "svelte";
    import { Tween } from "svelte/motion";
    import { cubicInOut } from "svelte/easing";
    import { goto } from "$app/navigation";
  
    /***************** FIRST PAGE VARIABLES & FUNCTIONS *****************/
    let currentImage = 0;
    let interval;
    let bgHeight = "100vh";
    let bgWidth = "100vw";
  
    let audio;
    let isPlaying = false;
  
    // Mouse move effect variables
    let x = 0;
    let y = 0;
    export let dataspeed = 0.5;
  
    // Moving text animation
    let yPosition = new Tween(0, { duration: 600, easing: cubicInOut });
  
    function domousemove(e) {
      x = (window.innerWidth - e.pageX * dataspeed) / 100;
      y = (window.innerHeight - e.pageY * dataspeed) / 100;
      checkCollision();
    }
  
    function startAudio() {
      if (audio && !isPlaying) {
        audio
          .play()
          .then(() => {
            isPlaying = true;
            console.log("✅ Audio started!");
          })
          .catch((error) => {
            console.error("❌ Autoplay blocked, waiting for interaction...", error);
          });
      }
    }
  
    onMount(() => {
      interval = setInterval(() => {
        currentImage = (currentImage + 1) % 3;
      }, 800);
      startTextAnimation();
    });
  
    onDestroy(() => {
      clearInterval(interval);
    });
  
    async function startTextAnimation() {
      while (true) {
        await yPosition.set(-20);
        await yPosition.set(0);
        await yPosition.set(20);
        await yPosition.set(0);
      }
    }
  
    // Draggable Bigman and collision detection
    let bigmanX = 65;
    let isDragging = false;
    let offsetX = 0;
    let bigmanEl;
    let mousemoveEl;
    let showGreeting = false;
  
    function startDrag(event) {
      event.preventDefault();
      isDragging = true;
      const container = document.querySelector(".bigman-container");
      const containerRect = container.getBoundingClientRect();
      offsetX =
        event.clientX - (containerRect.left + (bigmanX / 100) * containerRect.width);
      window.addEventListener("mousemove", onDrag);
      window.addEventListener("mouseup", stopDrag);
    }
  
    function onDrag(event) {
      if (isDragging) {
        const container = document.querySelector(".bigman-container");
        const containerRect = container.getBoundingClientRect();
        bigmanX =
          ((event.clientX - containerRect.left - offsetX) / containerRect.width) * 100;
        bigmanX = Math.max(0, Math.min(100, bigmanX));
        checkCollision();
      }
    }
  
    function stopDrag() {
      isDragging = false;
      window.removeEventListener("mousemove", onDrag);
      window.removeEventListener("mouseup", stopDrag);
    }
  
    function calculateScale() {
      const minX = 20;
      const maxX = 80;
      const normalizedX = 1 - Math.max(0, Math.min(1, (bigmanX - minX) / (maxX - minX)));
      return 0.93 - normalizedX * 0.7;
    }
  
    function calculateHeight() {
      const minX = 20;
      const maxX = 80;
      const normalizedX = 1 - Math.max(0, Math.min(1, (bigmanX - minX) / (maxX - minX)));
      return normalizedX * -35 - 60;
    }
  
    function checkCollision() {
      if (!bigmanEl || !mousemoveEl) return;
      const bigmanRect = bigmanEl.getBoundingClientRect();
      const mouseRect = mousemoveEl.getBoundingClientRect();
      if (
        bigmanRect.left < mouseRect.right &&
        bigmanRect.right > mouseRect.left &&
        bigmanRect.top < mouseRect.bottom &&
        bigmanRect.bottom > mouseRect.top
      ) {
        showGreeting = true;
      } else {
        showGreeting = false;
      }
    }
  
    // Page switching: when false, show first page; when true, show second page.
    let showGreetingPage = false;
    function greet() {
      showGreetingPage = true;
    }
  
    /***************** SECOND PAGE VARIABLES & FUNCTIONS *****************/
    let videoElement;
    let isRunning = true;
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
  
    let mediapipeHands;
  
    // Create a flag and mapping for character audio
    let isCharacterAudioPlaying = false;
    const characterAudioUrls = [
      "https://res.cloudinary.com/dufnjfidt/video/upload/v1743088251/davelow_ofvzyc.mp3",    // for cha2_xyy7uz.png (index 0)
      "https://res.cloudinary.com/dufnjfidt/video/upload/v1743088562/jessicahigh_isd5mh.mp3", // for cha3_q5qvmj.png (index 1)
      "https://res.cloudinary.com/dufnjfidt/video/upload/v1743089761/amy_aca405.mp3",    // for cha4_oe0vbk.png (index 2)
      "https://res.cloudinary.com/dufnjfidt/video/upload/v1740784365/Li_acauxj.mp3"        // for cha1_iuedwe.png (index 3)
    ];
  
    function playCharacterAudio(index) {
      if (isCharacterAudioPlaying) return;
      const audioEl = new Audio(characterAudioUrls[index]);
      isCharacterAudioPlaying = true;
      audioEl
        .play()
        .then(() => {
          console.log(`Playing audio for character ${index}`);
        })
        .catch((error) => {
          console.error("Error playing character audio", error);
          isCharacterAudioPlaying = false;
        });
      audioEl.addEventListener("ended", () => {
        isCharacterAudioPlaying = false;
      });
    }
  
    // Set up MediaPipe Hands on mount (but don’t call setupCamera yet)
    onMount(async () => {
      const { Hands } = await import("@mediapipe/hands");
      mediapipeHands = new Hands({
        locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
      });
      mediapipeHands.setOptions({
        maxNumHands: 2,
        modelComplexity: 1,
        minDetectionConfidence: 0.5,
        minTrackingConfidence: 0.5
      });
      mediapipeHands.onResults((results) => {
        if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
          processHand(results.multiHandLandmarks[0]);
        }
      });
    });
  
    // Reactive block: when the greeting page is shown and videoElement exists, set up the camera.
    let cameraSetupDone = false;
    $: if (showGreetingPage && videoElement && !cameraSetupDone) {
      cameraSetupDone = true;
      setupCamera().then(() => {
        detectHands();
      });
    }
  
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
      await mediapipeHands.send({ image: videoElement });
      if (isRunning) {
        requestAnimationFrame(detectHands);
      }
    }
  
    function processHand(landmarks) {
      if (!landmarks) return;
      const videoWidth = videoElement.videoWidth;
      const videoHeight = videoElement.videoHeight;
      const mirroredLandmarks = landmarks.map(({ x, y, z }) => [
        videoWidth - x * videoWidth,
        y * videoHeight,
        z
      ]);
      const scaleX = window.innerWidth / 640;
      const scaleY = window.innerHeight / 480;
      let newHandX = mirroredLandmarks[12][0] * scaleX;
      let newHandY = mirroredLandmarks[12][1] * scaleY + window.scrollY;
      handCursorX.set(newHandX);
      handCursorY.set(newHandY);
      checkCharacterCollision(newHandX, newHandY);
      detectGestureAndScroll({ landmarks: mirroredLandmarks });
    }
  
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
  
    function checkCharacterCollision(handX, handY) {
      characters.forEach((char, index) => {
        let charWidth = (char.width / 100) * window.innerWidth;
        let charHeight = charWidth;
        let charX = (jiggleX[index].current / 100) * window.innerWidth;
        let charY = (jiggleY[index].current / 100) * window.innerHeight + window.scrollY;
        let detectionBoxX = charX - charWidth / 2;
        let detectionBoxY = charY - charHeight / 2;
        let detectionBoxW = charWidth;
        let detectionBoxH = charHeight;
        if (
          handX > detectionBoxX &&
          handX < detectionBoxX + detectionBoxW &&
          handY > detectionBoxY &&
          handY < detectionBoxY + detectionBoxH
        ) {
          console.log(`✅ Collision detected with character ${index}`);
          triggerJiggle(index);
        }
      });
    }
  
    async function triggerJiggle(index) {
      // Play the associated audio for this character if no other is playing
      playCharacterAudio(index);
  
      let originalX = characters[index].x;
      let originalY = characters[index].y;
      await jiggleX[index].set(originalX + 1);
      await new Promise((resolve) => setTimeout(resolve, 100));
      await jiggleX[index].set(originalX - 1);
      await new Promise((resolve) => setTimeout(resolve, 100));
      await jiggleX[index].set(originalX);
      await jiggleY[index].set(originalY + 1);
      await new Promise((resolve) => setTimeout(resolve, 100));
      await jiggleY[index].set(originalY - 1);
      await new Promise((resolve) => setTimeout(resolve, 100));
      await jiggleY[index].set(originalY);
    }
  </script>
  
  <svelte:window on:mousemove={(e) => {
    if (!showGreetingPage) {
      startAudio();
      domousemove(e);
    }
  }} />
  
  <style>
    /* FIRST PAGE STYLES */
    .background {
      position: relative;
      width: 100vw;
      height: auto;
      display: block;
    }
    .background-overlay {
      position: absolute;
      top: 0;
      left: 0;
      width: 100vw;
      height: var(--bg-height, auto);
      background-image: url("https://res.cloudinary.com/dufnjfidt/image/upload/v1738788454/humanback_uonpdt.png");
      background-size: contain;
      background-repeat: no-repeat;
      background-position: center;
      pointer-events: none;
      z-index: 1;
    }
    .overlay-container {
      position: absolute;
      top: 0;
      left: 0;
      width: 100vw;
      height: var(--bg-height, auto);
      display: flex;
      justify-content: center;
      align-items: center;
      z-index: 2;
    }
    .human {
      position: absolute;
      opacity: 0;
      transition: opacity 0.2s ease-in-out;
    }
    .visible {
      opacity: 1;
    }
    .mousemove-image {
      width: 120px;
      height: auto;
      position: absolute;
      top: 85%;
      left: 34%;
      transform: translate(-50%, -50%);
      transition: transform 0.1s linear;
      pointer-events: none;
      z-index: 3;
    }
    .bigman-container {
      position: absolute;
      top: 0;
      left: 0;
      width: 100vw;
      height: 100vh;
      pointer-events: none;
      overflow: hidden;
      z-index: 4;
    }
    .bigman {
      position: absolute;
      cursor: grab;
      transition: transform 0.2s linear, top 0.2s ease-in-out;
      user-select: none;
      pointer-events: auto;
    }
    .bigman:active {
      cursor: grabbing;
    }
    .moving-text {
      position: absolute;
      top: 5%;
      left: 70%;
      transform: translate(-50%, 0);
      font-size: 1rem;
      color: #de513f;
      background-color: rgba(192, 214, 173, 0.8);
      padding: 10px;
      border-radius: 5px;
      box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
      z-index: 5;
    }
    .greeting-box {
      position: absolute;
      top: 60%;
      left: 50%;
      transform: translate(-50%, -50%);
      font-size: 1rem;
      color: #de513f;
      background-color: rgba(192, 214, 173, 0.8);
      padding: 10px;
      border-radius: 5px;
      box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
      z-index: 6;
    }
    .greeting-btn {
      display: inline-block;
      background-color: #6b8a88;
      color: white;
      border: none;
      border-radius: 4px;
      padding: 10px 16px;
      font-size: 1rem;
      cursor: pointer;
      margin-top: 10px;
    }
    .greeting-btn:hover {
      background-color: #578f66;
    }
    /* FIRST PAGE: Click Icon (if needed) */
    .click-icon {
      position: fixed;
      bottom: 10px;
      left: 10px;
      z-index: 1000;
    }
    .click-icon img {
      width: 7vw;
      height: auto;
    }
  
    /* SECOND PAGE STYLES */
    html,
    body {
      margin: 0;
      padding: 0;
      overflow-x: hidden;
    }
    .container {
      width: 100vw;
      height: 300vh;
      display: flex;
      flex-direction: column;
      position: absolute;
      top: 0;
      left: 0;
    }
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
      pointer-events: none;
    }
    .video-container {
      position: fixed;
      width: 100vw;
      height: 100vh;
    }
    video {
      
      position: absolute;
      width: 100%;
      height: 100%;
      opacity: 0;
    }
    /* SECOND PAGE: Icons at bottom left */
    .second-page-icons {
      position: fixed;
      bottom: 10px;
      left: 10px;
      z-index: 1000;
      display: flex;
      flex-direction: column;
    }
    .second-page-icons img {
      width: 7vw;
      height: auto;
      margin-bottom: 5px;
    }
  
    .character-wrapper {
      position: absolute;
      text-align: center;
    }
  
    .character-text {
      font-size: 1.5rem;
      color: #de513f;
      text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
      margin-top: 5px;
    }
  
    /* Start Practice Button Styles */
    .start-practice-container {
      position: absolute;
  bottom: 50px;
  width: 100%;
  left: 50%;
  text-align: center;
  z-index: 1100;
    }
    .start-practice {
      background-color: #6b8a88;
      color: white;
      border: none;
      border-radius: 4px;
      padding: 10px 16px;
      font-size: 1rem;
      cursor: pointer;
    }
    .start-practice:hover {
      background-color: #578f66;
    }
  </style>
  
  {#if !showGreetingPage}
    <!-- FIRST PAGE CONTENT -->
    <img
      class="background"
      src="https://res.cloudinary.com/dufnjfidt/image/upload/v1738787371/Section4-1_background_mytehd.jpg"
      alt="Background"
      on:load={(event) => {
        bgHeight = event.target.clientHeight + "px";
        bgWidth = event.target.clientWidth + "px";
      }}
      style="--bg-height: {bgHeight};"
    />
  
    <div class="background-overlay" style="height: {bgHeight};"></div>
  
    <div class="overlay-container" style="height: {bgHeight};">
      <img class="human {currentImage === 0 ? 'visible' : ''}" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1738788835/human1_k8n7fg.png" alt="Human 1" />
      <img class="human {currentImage === 1 ? 'visible' : ''}" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1738788836/human2_eoipmq.png" alt="Human 2" />
      <img class="human {currentImage === 2 ? 'visible' : ''}" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1738788836/human3_kd7lke.png" alt="Human 3" />
    </div>
  
    <div class="bigman-container">
      <img
        class="bigman"
        src="https://res.cloudinary.com/dufnjfidt/image/upload/v1738799208/Bigman_hwf3lk.png"
        alt="Bigman"
        on:mousedown={startDrag}
        style="left: {bigmanX}%; top: {calculateHeight()}%; transform: scale({calculateScale()});"
        bind:this={bigmanEl}
      />
    </div>
  
    <audio bind:this={audio} loop>
      <source src="https://res.cloudinary.com/dufnjfidt/video/upload/v1738790949/156480__teokgaming__park-with-traffic-and-chatter_sxychj.mp3" type="audio/mp3" />
    </audio>
  
    <img
      class="mousemove-image"
      src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740086098/feet_ell6mg.png"
      alt="Feet Image"
      style="transform: translateX({x}px) translateY({y}px);"
      bind:this={mousemoveEl}
    />
  
    <div class="moving-text" style="transform: translateY({yPosition.current}px);">
      Drag yourself closer to the group, onto the feet spot.
    </div>
  
    {#if showGreeting}
      <div class="greeting-box">
        They seem like a bunch of nice people, but I am a bit nervous.
        <button class="greeting-btn" on:click={greet}>
          Greet the group
        </button>
      </div>
    {/if}
  
    <!-- FIRST PAGE: Click Icon (if needed) -->
    <div class="click-icon">
      <img src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740497105/click1_zpl2dp.gif" alt="Click Icon" />
    </div>
  {:else}
    <!-- SECOND PAGE CONTENT (GREETING PAGE) -->
    <div class="container">
      {#each images as image}
        <div class="image-wrapper">
          <img class="image" src={image} alt="Park Image" />
        </div>
      {/each}
  
      <img
        class="hand-cursor"
        src={handCursorSrc}
        alt="Hand Cursor"
        style="left: {handCursorX.current}px; top: {handCursorY.current}px;"
      />
  
      {#each characters as char, index}
        <div
          class="character-wrapper"
          style="left: {jiggleX[index].current}%; top: {jiggleY[index].current}%; width: {char.width}%"
        >
          <img
            class="character"
            src={char.src}
            alt="Character"
            style="width: 100%;"
          />
          <div class="character-text">wave to say Hi at them</div>
        </div>
      {/each}

    <!-- New Start Practice Button -->
    <div class="start-practice-container">
      <button class="start-practice" on:click={() => goto('/teaching')}>
        start practice
      </button>
    </div>
    </div>
  
    <div class="video-container">
      <video bind:this={videoElement} autoplay></video>
    </div>
  
    <button on:click={() => (isRunning = false)}>Stop Detection</button>
  
    <!-- SECOND PAGE: Icons at bottom left corner -->
    <div class="second-page-icons">
      <img src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740498036/pointupdown_bwc6bg.gif" alt="Point Up Down" />
      <img src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740498037/wave_ekofea.gif" alt="Wave" />
    </div>
  
  {/if}
  
  