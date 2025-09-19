<script>
    import { onMount, onDestroy, tick } from 'svelte';
    import { cubicInOut } from 'svelte/easing';
    import { goto } from "$app/navigation";
  
    // Four images for the LI slideshow
    const slideshowImages = [
      'https://res.cloudinary.com/dufnjfidt/image/upload/v1741040018/li1_wskxm0.png',
      'https://res.cloudinary.com/dufnjfidt/image/upload/v1741040014/li2_pk0dbr.png',
      'https://res.cloudinary.com/dufnjfidt/image/upload/v1741040010/li4_xjiprv.png',
      'https://res.cloudinary.com/dufnjfidt/image/upload/v1741040007/li3_amffya.png'
    ];
  
    // Slideshow logic for LI images
    let currentIndex = 0;
    const displayInterval = 500;
    const transitionDuration = 300;
    let intervalId;
  
    // Declare audio variables (initialized in onMount)
    let dialogueAudio;
    let convoAudio;
    let convo2Audio;
    let convo3Audio;
  
    // Page states: "slideshow", "convo1", or "convo3"
    let pageState = "slideshow";
    let buttonVisible = true;
  
    // New state: controls when to switch to background1 with pose1
    let showPose1 = false;
  
    // State variables for the stretch/detection phase
    let showStretchPose = false; // used during pose1 stage ("Stretch Pose")
    let showLeapSquare = false;  // used during Class2 detection ("Leap Pose")
    let detectedClass2 = false;  // set when Class2 is detected
    let showPose3 = false;       // set when Class1 is detected
  
    // Variable for square text (will be "Stretch Pose", "Leap Pose", or later "Gather Pose")
    let squareText = "Stretch Pose";
  
    // New audio for the stretch pose (pose1 audio)
    let pose1Audio;
    // New audio for the leap pose (pose2 audio)
    let pose2Audio;
    // NEW: Gather audio (for the "Gather Pose" stage) and final audio for Class3
    let gatherAudio;
    let finalAudio;
  
    // Flags for the new gather detection stage
    let gatherStarted = false;  // to ensure gatherAudio plays only once
    let finalDetected = false;  // set when Class3 is detected
  
    // Container for the webcam canvas (if you want to see it)
    let webcamContainer;
  
    // Variables for pose detection using Teachable Machine
    let model, webcam, maxPredictions;
    const TM_URL = "https://teachablemachine.withgoogle.com/models/24lwLWBCj/";
  
    // Control variable to pause detection while audio is playing
    let detectionPaused = false;
    // Variable to switch detection mode: "class2" for pose1 detection; "class1" for after pose2 audio; "class3" for gather stage.
    let detectionMode = "class2";
    let devId = undefined;
  
    onMount(async () => {
      intervalId = setInterval(() => {
        currentIndex = (currentIndex + 1) % slideshowImages.length;
      }, displayInterval);

      const devices = await navigator.mediaDevices.enumerateDevices();
      console.log(devices);
      const myCam = devices.find(d => d.kind === 'videoinput' && d.label.includes('USB Camera'))
      devId = myCam?.deviceId;
      console.log(devId);
  
      // Initialize audio objects (client-side only)
      dialogueAudio = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1741761022/1_jcvhie.mp3');
      convoAudio = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1741761029/2_cgfjuh.mp3');
      convo2Audio = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1741761025/3_eq0rze.mp3');
      convo2Audio.crossOrigin = "anonymous";
      convo3Audio = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1741761033/4_qszfuw.mp3');
      convo3Audio.crossOrigin = "anonymous";
      pose1Audio = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1741761040/pose1_zv90wx.mp3');
      pose2Audio = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1741761044/pose2_usbauy.mp3');
      // New audios:
      gatherAudio = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1741761048/pose3_q1muea.mp3');
      finalAudio = new Audio('https://res.cloudinary.com/dufnjfidt/video/upload/v1741761037/5_amcwcy.mp3');
    });
  
    onDestroy(() => {
      clearInterval(intervalId);
      if (convo2Interval) clearInterval(convo2Interval);
    });
  
    // Custom transitions
    function fadeIn(node, { delay = 0, duration = 300, easing = cubicInOut } = {}) {
      return { delay, duration, easing, css: t => `opacity: ${0.7 + 0.3 * t}` };
    }
    function fadeOutToHalf(node, { delay = 0, duration = 300, easing = cubicInOut } = {}) {
      return { delay, duration, easing, css: t => `opacity: ${1 - 0.1 * t}` };
    }
  
    // --- Slideshow ---
    function handleClick() {
      buttonVisible = false;
      dialogueAudio.play();
      dialogueAudio.onended = () => {
        pageState = "convo1";
      };
    }
  
    // --- Convo1 ---
    let convoAudioStarted = false;
    let convoButtonVisible = false;
    let waitButtonVisible = false;
    let convoOverlay = "https://res.cloudinary.com/dufnjfidt/image/upload/v1741762369/litalk1_ohl24c.png";
  
    $: if (pageState === 'convo1' && !convoAudioStarted && convoAudio) {
      convoAudioStarted = true;
      convoAudio.play();
      convoAudio.onended = () => {
        convoButtonVisible = true;
      };
    }
  
    // --- Convo2 Animation Setup (within Convo1 view) ---
    const convo2Images = [
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741763182/groove1_rgtvb6.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791072/groove2_nxon3g.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741763182/groove1_rgtvb6.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791072/groove2_nxon3g.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741763182/groove1_rgtvb6.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791072/groove2_nxon3g.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791078/crouch1_mo6qog.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791084/crouch2_zpojqn.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791078/crouch1_mo6qog.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791084/crouch2_zpojqn.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791078/crouch1_mo6qog.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791084/crouch2_zpojqn.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791096/down1_fvj32u.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791090/down2_e2a4af.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791096/down1_fvj32u.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791090/down2_e2a4af.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791096/down1_fvj32u.png",
      "https://res.cloudinary.com/dufnjfidt/image/upload/v1741791090/down2_e2a4af.png"
    ];
    let showConvo2 = false;
    let convo2Index = 0;
    let convo2Interval;
  
    function handleConvoClick() {
      convoButtonVisible = false;
      showConvo2 = true;
      convo2Interval = setInterval(() => {
        convo2Index = (convo2Index + 1) % convo2Images.length;
      }, displayInterval);
      setTimeout(() => {
        waitButtonVisible = true;
      }, 3000);
    }
  
    // --- Convo3 Setup ---
    let convo3Overlay = "https://res.cloudinary.com/dufnjfidt/image/upload/v1743100036/label1logo_uvuq4e.png";
    let showActuallyButton = false;
    let showLabel2 = false;
    const label2Overlay = "https://res.cloudinary.com/dufnjfidt/image/upload/v1741808295/label2_hf7xav.png";
  
    function handleWaitClick() {
      pageState = "convo3";
      showLabel2 = false;
      showPose1 = false;
      const playPromise = convo2Audio.play();
      if (playPromise !== undefined) {
        playPromise.then(() => {
          convo2Audio.onended = () => {
            showActuallyButton = true;
          };
        }).catch(error => {
          console.error("Error playing convo2 audio:", error);
        });
      } else {
        showActuallyButton = true;
      }
    }
  
    // The Actually button handler:
    function handleActuallyClick() {
      showLabel2 = true;
      showActuallyButton = false;
      setTimeout(() => {
        showPose1 = true;
        convo3Audio.play();
        convo3Audio.onended = () => {
          // When convo3Audio finishes, show the square overlay ("Stretch Pose") and play pose1Audio.
          showStretchPose = true;
          pose1Audio.play();
          // Start detection for Class2 only after pose1Audio fully plays.
          pose1Audio.onended = () => {
            initDetectionClass2();
          };
        };
      }, 4500);
    }
  
    /* ------------------- Updated Detection Logic with Separate Canvas ------------------- */
    async function initDetectionClass2() {
      const tmPose = await import("@teachablemachine/pose");
      const modelURLFull = TM_URL + "model.json";
      const metadataURLFull = TM_URL + "metadata.json";
      model = await tmPose.load(modelURLFull, metadataURLFull);
      maxPredictions = model.getTotalClasses();
  
      webcam = new tmPose.Webcam(400, 400, true);
      await webcam.setup({
        deviceId: devId,
      });
      await webcam.play();
  
      if (webcamContainer.firstChild) {
        webcamContainer.removeChild(webcamContainer.firstChild);
      }
      webcamContainer.appendChild(webcam.canvas);
  
      detectionMode = "class2";
      detectionPaused = false;
      window.requestAnimationFrame(loop);
    }
  
    async function initDetectionClass1() {
      const tmPose = await import("@teachablemachine/pose");
      const modelURLFull = TM_URL + "model.json";
      const metadataURLFull = TM_URL + "metadata.json";
      model = await tmPose.load(modelURLFull, metadataURLFull);
      maxPredictions = model.getTotalClasses();
  
      webcam = new tmPose.Webcam(400, 400, true);
      await webcam.setup({
        deviceId: devId,
      });
      await webcam.play();
  
      if (webcamContainer.firstChild) {
        webcamContainer.removeChild(webcamContainer.firstChild);
      }
      webcamContainer.appendChild(webcam.canvas);
  
      detectionMode = "class1";
      detectionPaused = false;
      window.requestAnimationFrame(loop);
    }
  
    async function initDetectionClass3() {
      const tmPose = await import("@teachablemachine/pose");
      const modelURLFull = TM_URL + "model.json";
      const metadataURLFull = TM_URL + "metadata.json";
      model = await tmPose.load(modelURLFull, metadataURLFull);
      maxPredictions = model.getTotalClasses();
  
      webcam = new tmPose.Webcam(400, 400, true);
      await webcam.setup({
        deviceId: devId,
      });
      await webcam.play();
  
      if (webcamContainer.firstChild) {
        webcamContainer.removeChild(webcamContainer.firstChild);
      }
      webcamContainer.appendChild(webcam.canvas);
  
      detectionMode = "class3";
      detectionPaused = false;
      window.requestAnimationFrame(loop);
    }
  
    async function loop() {
      if (detectionPaused) return;
      webcam.update();
      await predict();
      window.requestAnimationFrame(loop);
    }
  
    async function predict() {
      const { posenetOutput } = await model.estimatePose(webcam.canvas);
      const predictions = await model.predict(posenetOutput);
      console.log("Predictions:", predictions);
      for (let i = 0; i < predictions.length; i++) {
        const cls = predictions[i].className.toLowerCase();
        const prob = predictions[i].probability;
        console.log(`Prediction ${i}: ${cls} - ${prob.toFixed(2)}`);
        if (detectionMode === "class2") {
          if (cls === "class 2" && prob > 0.9 && !detectionPaused) {
            console.log("Class 2 detected with probability:", prob);
            detectedClass2 = true;
            showLeapSquare = true;
            squareText = "Leap Pose";
            detectionPaused = true;
            playPose2Audio();
            break;
          }
        } else if (detectionMode === "class1") {
          if (cls === "class 1" && prob > 0.9 && !detectionPaused) {
            console.log("Class 1 detected with probability:", prob);
            showPose3 = true;
            // Clear Class2 flag so UI switches from Pose2 to Pose3 (if needed)
            detectedClass2 = false;
            detectionPaused = true;
            break;
          }
        } else if (detectionMode === "class3") {
          if (cls === "class 3" && prob > 0.9 && !detectionPaused) {
            console.log("Class 3 detected with probability:", prob);
            finalDetected = true;
            detectionPaused = true;
            playFinalAudio();
            break;
          }
        }
      }
    }
  
    function playPose2Audio() {
      pose2Audio.play();
      pose2Audio.onended = () => {
        // Do not clear detectedClass2 so the UI remains in Pose2 mode.
        // Instead, switch detection mode to class1 and resume detection.
        detectionPaused = false;
        initDetectionClass1();
      };
    }
  
    function playFinalAudio() {
      finalAudio.play();
      finalAudio.onended = () => {
    goto("/listory");
  };
    }
  
  
    $: if (showPose3 && !gatherStarted) {
      gatherStarted = true;
      // When Pose3 overlay appears, play gather audio and show a square with "Gather Pose"
      // Update square text to "Gather Pose"
      squareText = "Gather Pose";
      // Play gather audio and after it ends, start detection for Class3.
      gatherAudio.play();
      gatherAudio.onended = ()  => {
        initDetectionClass3();
      };
    }
  </script>
  
  <style>
    :global(html, body) {
      margin: 0;
      padding: 0;
      width: 100vw;
      height: 100vh;
      overflow: hidden;
      box-sizing: border-box;
    }
    /* Slideshow styles */
    .scrolling-wrapper-flexbox {
      display: flex;
      flex-wrap: nowrap;
      overflow-x: auto;
      overflow-y: hidden;
      width: 100vw;
      height: 100vh;
    }
    .card {
      flex: 0 0 auto;
      width: 100vw;
      height: 100vh;
      position: relative;
    }
    .bg {
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
    .overlay {
      position: absolute;
    }
    .jessDave {
      width: 50%;
      height: auto;
      left: 20%;
      top: 20%;
    }
    .emilySteve {
      width: 50%;
      height: auto;
      left: 10%;
      top: 30%;
    }
    .slideshow {
      position: absolute;
      top: 50%;
      right: 0;
      transform: translateY(-50%);
      width: 100vw;
      height: 100vh;
      overflow: hidden;
    }
    .slide-img {
      position: absolute;
      top: 0;
      right: 0;
      width: 100%;
      height: 100%;
      object-fit: contain;
    }
    .dialogue-btn {
      position: absolute;
      top: 70vh;
      left: 65vw;
      padding: 0.5em 1em;
      background: rgba(255,255,255,0.8);
      border: 1px solid #ccc;
      border-radius: 4px;
      cursor: pointer;
      font-size: 1rem;
    }
    .click-btn {
      position: absolute;
      top: 10px;
      right: 10px;
      height: 10vh;
      width: 10vh;
      cursor: pointer;
    }
    /* Convo1 styles */
    .convo1 {
      width: 100vw;
      height: 100vh;
      position: relative;
    }
    .convo1 .bg {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
    .convo1 .overlay {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
    .convo-btn {
      position: absolute;
      top: 70vh;
      left: 40vw;
      padding: 0.5em 1em;
      background: rgba(255,255,255,0.8);
      border: 1px solid #ccc;
      border-radius: 4px;
      cursor: pointer;
      font-size: 1rem;
    }
    .wait-btn {
      left: 80vw;
    }
    /* Convo2 animated container */
    .convo2 {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
    }
    /* Convo3 styles */
    .convo3 {
      width: 100vw;
      height: 100vh;
      position: relative;
    }
    .convo3 .bg {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
    .convo3 .overlay {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
    .stretch-container {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      text-align: center;
      z-index: 10;
    }
    .stretch-square {
      width: 80vh;
      height: 80vh;
      border: 5px solid rgb(66, 67, 50);
      background: transparent;
      margin: 0 auto;
    }
    .stretch-text {
      color: rgb(69, 63, 42);
      font-size: 2rem;
      margin-top: 1rem;
    }
  </style>
  
  {#if pageState === 'slideshow'}
    <div class="scrolling-wrapper-flexbox">
      <div class="card">
        <img class="bg" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740989643/background2_cwzqs6.jpg" alt="Background 1" />
        <img class="overlay jessDave" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1741033315/jessdave_cxe1lj.png" alt="Jess & Dave" />
      </div>
      <div class="card">
        <img class="bg" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740989647/background1_cjdsfp.jpg" alt="Background 2" />
        <img class="overlay emilySteve" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1741033312/emilysteve_kfvsp1.png" alt="Emily & Steve" />
        <div class="slideshow">
          {#each slideshowImages as img, i}
            {#if i === currentIndex}
              <img class="slide-img" src={img} alt="LI slideshow" in:fadeIn={{ duration: transitionDuration }} out:fadeOutToHalf={{ duration: transitionDuration }} />
            {/if}
          {/each}
        </div>
        {#if buttonVisible}
          <button class="dialogue-btn" on:click={handleClick}>Ugh... Hello?</button>
        {/if}
        <div class="click-btn">
          <img src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740497105/click1_zpl2dp.gif" alt="Click animation" style="height:100%;" />
        </div>
      </div>
    </div>
  {:else if pageState === 'convo1'}
    <div class="convo1">
      <img class="bg" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740989647/background1_cjdsfp.jpg" alt="New Background" />
      {#if showConvo2}
        <div class="convo2">
          <img src={convo2Images[convo2Index]} alt="Convo Animation" in:fadeIn={{ duration: transitionDuration }} out:fadeOutToHalf={{ duration: transitionDuration }} style="width:100%; height:100%; object-fit:cover;" />
        </div>
      {:else}
        <img class="overlay" src={convoOverlay} alt="Convo Overlay" />
      {/if}
      {#if convoButtonVisible && !showConvo2}
        <button class="convo-btn" on:click={handleConvoClick}>I guess yes?</button>
      {/if}
      {#if waitButtonVisible}
        <button class="convo-btn wait-btn" on:click={handleWaitClick}>Wait...</button>
      {/if}
    </div>
  {:else if pageState === 'convo3'}
    <div class="convo3">
      {#if showPose3}
        {#if finalDetected}
          <!-- Final stage: if Class3 is detected -->
          <img class="bg" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740989647/background1_cjdsfp.jpg" alt="Background 1" />
          <img class="overlay" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1742318792/final_fuslpm.png" alt="Final Pose" />
        {:else}
          <!-- Gather stage: show Pose3 overlay, plus draw square with text "Gather Pose" -->
          <img class="bg" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740989647/background1_cjdsfp.jpg" alt="Background 1" />
          <img class="overlay" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1742274085/pose3_tnurzm.png" alt="Pose3" />
          <div class="stretch-container">
            <div class="stretch-square"></div>
            <div class="stretch-text">Gather Pose</div>
          </div>
        {/if}
      {:else if detectedClass2}
        <img class="bg" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740989647/background1_cjdsfp.jpg" alt="Background 1" />
        <img class="overlay" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1742274030/pose2_kww8zu.png" alt="Pose2" />
        {#if showLeapSquare}
          <div class="stretch-container">
            <div class="stretch-square"></div>
            <div class="stretch-text">{squareText}</div>
          </div>
        {/if}
      {:else if showPose1}
        <img class="bg" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740989647/background1_cjdsfp.jpg" alt="Background 1" />
        <img class="overlay" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1742274024/pose1_nmgawy.png" alt="Pose 1" />
        {#if showStretchPose}
          <div class="stretch-container">
            <div class="stretch-square"></div>
            <div class="stretch-text">Stretch Pose</div>
          </div>
        {/if}
      {:else if showLabel2}
        <img class="overlay" src={label2Overlay} alt="Label2" />
      {:else}
        <img class="overlay" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1743100036/label1logo_uvuq4e.png" alt="Label1" />
      {/if}
      {#if showActuallyButton}
        <button class="convo-btn" on:click={handleActuallyClick}>
          Actually, you have to turn around. I won't be able to see the movement when your back is facing me...
        </button>
      {/if}
    </div>
  {/if}
  
  <!-- Hidden container for the webcam canvas -->
  <div bind:this={webcamContainer} style="display: none;"></div>
