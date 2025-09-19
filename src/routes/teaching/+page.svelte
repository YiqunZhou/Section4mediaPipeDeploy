<script>
    import { onMount, tick } from "svelte";
    import { goto } from "$app/navigation";
  
    // Stages: "initial", "idle", "pose", "stretch", "leap", "gather"
    let stage = "initial";
  
    // Video element references
    let videoElement;          // Idle video
    let stretchVideoElement;   // Stretch video
    let leapVideoElement;      // Leap video
    let gatherVideoElement;    // Gather video
  
    // Audio element references
    let stretchAudioElement;   // Stretch audio
    let leapAudioElement;      // Leap audio
    let gatherAudioElement;    // Gather audio
  
    // Pose detection variables for Class 4, Class 2, and Class 1
    let model, webcam, maxPredictions;
    let detecting = false; // For Class 4 detection
    const URL = "https://teachablemachine.withgoogle.com/models/24lwLWBCj/";
  
    let detectingClass2 = false;
    let webcam2;
  
    let detectingClass1 = false;
    let webcam3;
  
    // For Class 3 detection in the Gather stage
    let detectingClass3 = false;
    let webcam4;
  
    // Flags to show overlays
    let showStretchOverlay = false;
    let showLeapOverlay = false;
    let showGatherOverlay = false;
    let devId = undefined
  
    onMount(async () => {
      console.log("Component mounted, video elements should be available.");
      const devices = await navigator.mediaDevices.enumerateDevices();
      console.log(devices);
      const myCam = devices.find(d => d.kind === 'videoinput' && d.label.includes('USB Camera'))
      devId = myCam?.deviceId;
      console.log(devId);
    });
  
    // ------------- Initial & Idle Stage -------------
    function handleJoin() {
      stage = "idle";
      console.log("Join clicked, stage set to idle.");
    }
  
    // When idle video ends, freeze it at its last frame.
    // Also, remove the standup gif overlay and then after 5 seconds start Class 4 detection.
    function handleVideoEnd() {
      console.log("Idle video ended.");
      videoElement.currentTime = videoElement.duration - 0.1;
      videoElement.pause();
      // Change stage to "pose" so that the idle overlay is shown (and standup gif removed).
      stage = "pose";
      console.log("Idle video frozen. Showing idle overlay.");
      // After 5 seconds, start Class 4 detection.
      setTimeout(() => {
        detecting = true;
        loadModelForClass4();
      }, 2000);
    }
  
    // ------------- Class 4 Detection -------------
    async function loadModelForClass4() {
      console.log("Loading model for Class 4 detection...");
      const tmPose = await import("@teachablemachine/pose");
      const modelURLFull = URL + "model.json";
      const metadataURLFull = URL + "metadata.json";
      model = await tmPose.load(modelURLFull, metadataURLFull);
      maxPredictions = model.getTotalClasses();
      console.log("Model loaded. Setting up webcam for Class 4 detection...");
      webcam = new tmPose.Webcam(200, 200, true);
      await webcam.setup({
        deviceId: devId,
      });
      await webcam.play();
      console.log("Webcam started for Class 4.");
      requestAnimationFrame(loopClass4);
    }
  
    async function loopClass4() {
      if (!detecting) return;
      webcam.update();
      await predictClass4();
      if (detecting) requestAnimationFrame(loopClass4);
    }
  
    async function predictClass4() {
      const { posenetOutput } = await model.estimatePose(webcam.canvas);
      const prediction = await model.predict(posenetOutput);
      console.log("Class 4 Prediction:", prediction);
      const class4Pred = prediction.find(
        p => p.className.toLowerCase() === "class 4"
      );
      if (class4Pred && class4Pred.probability > 0.9) {
        console.log("Class 4 detected with probability:", class4Pred.probability);
        detecting = false;
        startStretchStage();
      } else {
        console.log("Class 4 not detected or confidence too low.");
      }
    }
  
    // ------------- Stretch Stage & Class 2 Detection -------------
    async function startStretchStage() {
      if (webcam) {
        webcam.stop();
        console.log("Webcam stopped for Class 4 detection.");
      }
      stage = "stretch";
      console.log("Stage set to stretch. Waiting for DOM update...");
      await tick();
  
      // Pause stretch video at its second frame (small offset)
      if (stretchVideoElement) {
        stretchVideoElement.pause();
        stretchVideoElement.currentTime = 0.04;
        console.log("Stretch video set to second frame and paused.");
      }
  
      // Play stretch audio concurrently.
      if (stretchAudioElement) {
        stretchAudioElement.play().then(() => {
          console.log("Stretch audio playing.");
        }).catch(err => console.error("Error playing stretch audio:", err));
      }
  
      // Hold the paused stretch video for 5 seconds, then play it.
      setTimeout(() => {
        if (stretchVideoElement) {
          stretchVideoElement.play().then(() => {
            console.log("Stretch video started playing after 5-second pause.");
          }).catch(err => console.error("Error playing stretch video:", err));
        }
      }, 2000);
  
      // When stretch video ends, freeze it at its last frame and show overlay.
      if (stretchVideoElement) {
        stretchVideoElement.onended = () => {
          stretchVideoElement.currentTime = stretchVideoElement.duration - 0.1;
          stretchVideoElement.pause();
          console.log("Stretch video ended and frozen at last frame.");
          showStretchOverlay = true;
          console.log("Showing stretch overlay.");
          // After 5 seconds, start detection for Class 2.
          setTimeout(() => {
            startClass2Detection();
          }, 2000);
        };
      }
    }
  
    async function startClass2Detection() {
      console.log("Starting detection for Class 2.");
      detectingClass2 = true;
      const tmPose = await import("@teachablemachine/pose");
      webcam2 = new tmPose.Webcam(200, 200, true);
      await webcam2.setup({
        deviceId: devId,
      });
      await webcam2.play();
      console.log("Webcam started for Class 2 detection.");
      requestAnimationFrame(loopClass2);
    }
  
    async function loopClass2() {
      if (!detectingClass2) return;
      webcam2.update();
      await predictClass2();
      if (detectingClass2) requestAnimationFrame(loopClass2);
    }
  
    async function predictClass2() {
      const { posenetOutput } = await model.estimatePose(webcam2.canvas);
      const prediction = await model.predict(posenetOutput);
      console.log("Class 2 Prediction:", prediction);
      const class2Pred = prediction.find(
        p => p.className.toLowerCase() === "class 2"
      );
      if (class2Pred && class2Pred.probability > 0.9) {
        console.log("Class 2 detected with probability:", class2Pred.probability);
        detectingClass2 = false;
        webcam2.stop();
        startLeapStage();
      } else {
        console.log("Class 2 not detected or confidence too low.");
      }
    }
  
    // ------------- Leap Stage & Class 1 Detection -------------
    async function startLeapStage() {
      stage = "leap";
      console.log("Stage set to leap. Waiting for DOM update...");
      await tick();
  
      // Play leap audio concurrently.
      if (leapAudioElement) {
        leapAudioElement.play().then(() => {
          console.log("Leap audio playing.");
        }).catch(err => console.error("Error playing leap audio:", err));
      }
  
      // Play leap video normally.
      if (leapVideoElement) {
        leapVideoElement.play().then(() => {
          console.log("Leap video playing.");
        }).catch(err => console.error("Error playing leap video:", err));
      }
  
      // When leap video ends, freeze it at its last frame and show the leap overlay.
      if (leapVideoElement) {
        leapVideoElement.onended = () => {
          leapVideoElement.currentTime = leapVideoElement.duration - 0.1;
          leapVideoElement.pause();
          console.log("Leap video ended and frozen at last frame.");
          showLeapOverlay = true;
          console.log("Showing leap overlay.");
          // After 5 seconds, start detection for Class 1.
          setTimeout(() => {
            startClass1Detection();
          }, 2000);
        };
      }
    }
  
    async function startClass1Detection() {
      console.log("Starting detection for Class 1.");
      detectingClass1 = true;
      const tmPose = await import("@teachablemachine/pose");
      webcam3 = new tmPose.Webcam(200, 200, true);
      await webcam3.setup({
        deviceId: devId,
      });
      await webcam3.play();
      console.log("Webcam started for Class 1 detection.");
      requestAnimationFrame(loopClass1);
    }
  
    async function loopClass1() {
      if (!detectingClass1) return;
      webcam3.update();
      await predictClass1();
      if (detectingClass1) requestAnimationFrame(loopClass1);
    }
  
    async function predictClass1() {
      const { posenetOutput } = await model.estimatePose(webcam3.canvas);
      const prediction = await model.predict(posenetOutput);
      console.log("Class 1 Prediction:", prediction);
      const class1Pred = prediction.find(
        p => p.className.toLowerCase() === "class 1"
      );
      if (class1Pred && class1Pred.probability > 0.9) {
        console.log("Class 1 detected with probability:", class1Pred.probability);
        detectingClass1 = false;
        webcam3.stop();
        startGatherStage();
      } else {
        console.log("Class 1 not detected or confidence too low.");
      }
    }
  
    // ------------- Gather Stage & Class 3 Detection -------------
    async function startGatherStage() {
      stage = "gather";
      console.log("Stage set to gather. Waiting for DOM update...");
      await tick();
  
      // Play gather audio concurrently.
      if (gatherAudioElement) {
        gatherAudioElement.play().then(() => {
          console.log("Gather audio playing.");
        }).catch(err => console.error("Error playing gather audio:", err));
      }
  
      // Pause gather video at its second frame.
      if (gatherVideoElement) {
        gatherVideoElement.pause();
        gatherVideoElement.currentTime = 0.04;
        console.log("Gather video set to second frame and paused.");
      }
  
      // After 5 seconds, play gather video.
      setTimeout(() => {
        if (gatherVideoElement) {
          gatherVideoElement.play().then(() => {
            console.log("Gather video started playing after 5-second pause.");
          }).catch(err => console.error("Error playing gather video:", err));
        }
      }, 2000);
  
      // When gather video ends, freeze it at its last frame and show the gather overlay.
      if (gatherVideoElement) {
        gatherVideoElement.onended = () => {
          gatherVideoElement.currentTime = gatherVideoElement.duration - 0.1;
          gatherVideoElement.pause();
          console.log("Gather video ended and frozen at last frame.");
          showGatherOverlay = true;
          console.log("Showing gather overlay.");
          // After 5 seconds, start detection for Class 3.
          setTimeout(() => {
            startClass3Detection();
          }, 2000);
        };
      }
    }
  
    async function startClass3Detection() {
      console.log("Starting detection for Class 3.");
      detectingClass3 = true;
      const tmPose = await import("@teachablemachine/pose");
      webcam4 = new tmPose.Webcam(200, 200, true);
      await webcam4.setup({
        deviceId: devId,
      });
      await webcam4.play();
      console.log("Webcam started for Class 3 detection.");
      requestAnimationFrame(loopClass3);
    }
  
    async function loopClass3() {
      if (!detectingClass3) return;
      webcam4.update();
      await predictClass3();
      if (detectingClass3) requestAnimationFrame(loopClass3);
    }
  
    async function predictClass3() {
      const { posenetOutput } = await model.estimatePose(webcam4.canvas);
      const prediction = await model.predict(posenetOutput);
      console.log("Class 3 Prediction:", prediction);
      const class3Pred = prediction.find(
        p => p.className.toLowerCase() === "class 3"
      );
      if (class3Pred && class3Pred.probability > 0.9) {
        console.log("Class 3 detected with probability:", class3Pred.probability);
        detectingClass3 = false;
        webcam4.stop();
        goto("/litalk");
      } else {
        console.log("Class 3 not detected or confidence too low.");
      }
    }
  </script>
  
  <style>
    /* General styling */
    .container {
      position: fixed;
      top: 0;
      left: 0;
      min-width: 100vw;
      min-height: 100vh;
      overflow-x: hidden;
    }
    .imgbackground {
      width: 100%;
      height: 100%;
      object-fit: cover;
      position: absolute;
      top: 0;
      left: 0;
      z-index: -1;
    }
    .dialogue-box {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      background: transparent;
      color: #000;
      padding: 1rem 2rem;
      border-radius: 8px;
      text-align: center;
      transition: background 0.3s, color 0.3s;
      border: 2px solid rgba(0, 0, 0, 0.5);
    }
    .dialogue-box button {
      margin-top: 1rem;
      padding: 0.5rem 1rem;
      border: 1px solid rgba(0, 0, 0, 0.5);
      border-radius: 4px;
      background: transparent;
      color: #000;
      cursor: pointer;
      transition: background 0.3s, color 0.3s;
    }
    .dialogue-box:hover {
      background: rgba(50, 50, 50, 0.8);
      color: #fff;
    }
    .dialogue-box:hover button {
      background: rgba(255, 255, 0, 0.8);
      color: #fff;
    }
    .click-gif {
      position: absolute;
      bottom: 0;
      left: 0;
      width: 7vw;
      height: auto;
    }
    .video-container {
      position: fixed;
      top: 0;
      left: 0;
      width: 100vw;
      height: 100vh;
      overflow: hidden;
      z-index: 1;
    }
    .video-container video {
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
    .standup-gif {
      position: absolute;
      bottom: 20px;
      left: 5px;
      width: 15vw;
    }
    /* Idle overlay: shown only when idle video has ended (stage "pose") */
    .idle-overlay {
      position: absolute;
      top: 0;
      left: 0;
      width: 100vw;
      height: 100vh;
      z-index: 2;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      pointer-events: none;
    }
    .idle-square {
      width: 80vh;
      height: 80vh;
      border: 4px solid dodgerblue;
      box-sizing: border-box;
    }
    .idle-text {
      margin-top: 1rem;
      color: #020101;
      font-size: 3rem;
    }
    /* Overlays for stretch, leap, and gather stages */
    .stretch-overlay,
    .leap-overlay,
    .gather-overlay {
      position: absolute;
      top: 0;
      left: 0;
      width: 100vw;
      height: 100vh;
      z-index: 3;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
    }
    .overlay-box {
      background: rgba(0, 0, 0, 0.5);
      padding: 1rem 2rem;
      border-radius: 8px;
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    .overlay-square {
      width: 60vh;
      height: 60vh;
      border: 4px solid dodgerblue;
      margin-bottom: 1rem;
      box-sizing: border-box;
    }
    .overlay-text {
      color: #fff;
      font-size: 3rem;
      text-align: center;
    }
  </style>
  
  {#if stage === 'initial'}
    <!-- Initial screen -->
    <div class="container">
      <img class="imgbackground" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740690919/teachstart_jqyb66.jpg" alt="Background Image" />
      <div class="dialogue-box">
        They are gathering around now. The practice is going to start.
        <button on:click={handleJoin}>Click to join</button>
      </div>
      <img class="click-gif" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740497105/click1_zpl2dp.gif" alt="Click gif" />
    </div>
  
  {:else if stage === 'idle' || stage === 'pose'}
    <!-- Idle video stage -->
    <div class="video-container">
      <video bind:this={videoElement} autoplay playsinline on:ended={handleVideoEnd}>
        <source src="https://res.cloudinary.com/dufnjfidt/video/upload/v1740774225/idle.move_ld61z5.mp4" type="video/mp4" />
        Your browser does not support the video tag.
      </video>
      <audio autoplay>
        <source src="https://res.cloudinary.com/dufnjfidt/video/upload/v1743087216/idlelow_rn3hau.mp3" type="audio/mp3" />
        Your browser does not support the audio element.
      </audio>
      {#if stage === 'idle'}
        <img class="standup-gif" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1740775006/standup_by42gs.gif" alt="Stand Up Gif" />
      {/if}
      {#if stage === 'pose'}
        <div class="idle-overlay">
          <div class="idle-square"></div>
          <div class="idle-text">
            Do the pose in the square to show you are ready.
          </div>
        </div>
      {/if}
    </div>
  
  {:else if stage === 'stretch'}
    <!-- Stretch stage -->
    <div class="video-container">
      <video bind:this={stretchVideoElement} playsinline preload="auto" poster="https://res.cloudinary.com/dufnjfidt/video/upload/v1740776160/stretch.move_wl3tpx.mp4">
        <source src="https://res.cloudinary.com/dufnjfidt/video/upload/v1740776160/stretch.move_wl3tpx.mp4" type="video/mp4" />
        Your browser does not support the video tag.
      </video>
      <audio bind:this={stretchAudioElement} src="https://res.cloudinary.com/dufnjfidt/video/upload/v1743087211/sketchlow_c1sume.mp3" autoplay></audio>
      {#if showStretchOverlay}
        <div class="stretch-overlay">
          <div class="overlay-box">
            <div class="overlay-square"></div>
            <div class="overlay-text">
              Do this stretch pose to unlock next move
            </div>
          </div>
        </div>
      {/if}
    </div>
  
  {:else if stage === 'leap'}
    <!-- Leap stage -->
    <div class="video-container">
      <video bind:this={leapVideoElement} playsinline preload="auto">
        <source src="https://res.cloudinary.com/dufnjfidt/video/upload/v1740779995/leap.move_j4nru6.mp4" type="video/mp4" />
        Your browser does not support the video tag.
      </video>
      <audio bind:this={leapAudioElement} src="https://res.cloudinary.com/dufnjfidt/video/upload/v1743087214/leaplow_mwermz.mp3" autoplay></audio>
      {#if showLeapOverlay}
        <div class="leap-overlay">
          <div class="overlay-box">
            <div class="overlay-square"></div>
            <div class="overlay-text">
              Do this Leap pose to unlock next pose
            </div>
          </div>
        </div>
      {/if}
    </div>
  
  {:else if stage === 'gather'}
    <!-- Gather stage -->
    <div class="video-container">
      <video bind:this={gatherVideoElement} playsinline preload="auto" poster="https://res.cloudinary.com/dufnjfidt/video/upload/v1740781200/gather.move_d7u3lj.mp4">
        <source src="https://res.cloudinary.com/dufnjfidt/video/upload/v1740781200/gather.move_d7u3lj.mp4" type="video/mp4" />
        Your browser does not support the video tag.
      </video>
      <audio bind:this={gatherAudioElement} src="https://res.cloudinary.com/dufnjfidt/video/upload/v1743087213/gatherlow_hyftum.mp3" autoplay></audio>
      {#if showGatherOverlay}
        <div class="gather-overlay">
          <div class="overlay-box">
            <div class="overlay-square"></div>
            <div class="overlay-text">
              Do this pose Gather to end the session
            </div>
          </div>
        </div>
      {/if}
    </div>
  {/if}
  