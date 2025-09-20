<!-- +page.svelte (or your pose page) -->

<script lang="ts" context="module">
  // Disable SSR so window.tmPose exists on this route.
  export const ssr = false;
</script>

<script lang="ts">
  import { onMount, tick } from "svelte";
  import { goto } from "$app/navigation";

  // Stages: "initial", "idle", "pose", "stretch", "leap", "gather"
  let stage: "initial" | "idle" | "pose" | "stretch" | "leap" | "gather" = "initial";

  // Video element references
  let videoElement: HTMLVideoElement;        // Idle video
  let stretchVideoElement: HTMLVideoElement; // Stretch video
  let leapVideoElement: HTMLVideoElement;    // Leap video
  let gatherVideoElement: HTMLVideoElement;  // Gather video

  // Audio element references
  let idleAudioElement: HTMLAudioElement;    // Idle audio (started on click)
  let stretchAudioElement: HTMLAudioElement; // Stretch audio
  let leapAudioElement: HTMLAudioElement;    // Leap audio
  let gatherAudioElement: HTMLAudioElement;  // Gather audio

  // Pose detection variables
  let model: any, webcam: any, maxPredictions: number;
  let detecting = false; // Class 4
  const URL = "https://teachablemachine.withgoogle.com/models/24lwLWBCj/";

  let detectingClass2 = false;
  let webcam2: any;

  let detectingClass1 = false;
  let webcam3: any;

  // For Class 3 detection in the Gather stage
  let detectingClass3 = false;
  let webcam4: any;

  // Overlay flags
  let showStretchOverlay = false;
  let showLeapOverlay = false;
  let showGatherOverlay = false;

  onMount(() => {
    console.log("Component mounted.");
  });

  // ---- helpers for tmPose from CDN ----
  function getTmPose(): any {
    return (window as any).tmPose;
  }
  function waitForTmPose(): Promise<void> {
    return new Promise((resolve) => {
      const check = () => (getTmPose() ? resolve() : setTimeout(check, 50));
      check();
    });
  }

  // ------------- Initial & Idle Stage -------------
  async function handleJoin() {
    stage = "idle";
    console.log("Join clicked, stage set to idle.");
    await tick(); // ensure DOM is updated so refs exist

    try {
      if (videoElement) {
        videoElement.muted = true; // required for reliable autoplay
        await videoElement.play();
      }
      if (idleAudioElement) {
        // start idle audio after gesture; avoids autoplay blocking
        await idleAudioElement.play();
      }
    } catch (err) {
      console.warn("Autoplay/start blocked:", err);
    }
  }

  function handleVideoEnd() {
    console.log("Idle video ended.");
    videoElement.currentTime = Math.max(0, videoElement.duration - 0.1);
    videoElement.pause();
    stage = "pose";
    console.log("Idle video frozen. Showing idle overlay.");
    // After 5 seconds, start Class 4 detection.
    setTimeout(() => {
      detecting = true;
      loadModelForClass4();
    }, 5000);
  }

  // ------------- Class 4 Detection -------------
  async function loadModelForClass4() {
    try {
      console.log("Loading model for Class 4 detection...");
      await waitForTmPose();
      const tmPose = getTmPose();
      if (!tmPose) throw new Error("tmPose not loaded");

      const modelURLFull = URL + "model.json";
      const metadataURLFull = URL + "metadata.json";
      model = await tmPose.load(modelURLFull, metadataURLFull);
      maxPredictions = model.getTotalClasses?.() ?? 0;

      console.log("Model loaded. Setting up webcam for Class 4 detection...");
      webcam = new tmPose.Webcam(200, 200, true);
      await webcam.setup();
      await webcam.play();
      console.log("Webcam started for Class 4.");
      requestAnimationFrame(loopClass4);
    } catch (e) {
      console.error("Failed to init Class 4 detection:", e);
    }
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
    const class4Pred = prediction.find((p: any) => p.className.toLowerCase() === "class 4");
    if (class4Pred && class4Pred.probability > 0.9) {
      console.log("Class 4 detected:", class4Pred.probability);
      detecting = false;
      if (webcam) webcam.stop();
      startStretchStage();
    } else {
      console.log("Class 4 not detected or confidence too low.");
    }
  }

  // ------------- Stretch Stage & Class 2 Detection -------------
  async function startStretchStage() {
    stage = "stretch";
    console.log("Stage set to stretch. Waiting for DOM update...");
    await tick();

    if (stretchVideoElement) {
      try {
        stretchVideoElement.pause();
        stretchVideoElement.currentTime = 0.04;
        console.log("Stretch video set to second frame and paused.");
      } catch (e) {
        console.warn("Stretch video init error:", e);
      }
    }

    if (stretchAudioElement) {
      stretchAudioElement.play().catch((err) => console.error("Error playing stretch audio:", err));
    }

    setTimeout(() => {
      if (stretchVideoElement) {
        stretchVideoElement.play().catch((err) => console.error("Error playing stretch video:", err));
      }
    }, 5000);

    if (stretchVideoElement) {
      stretchVideoElement.onended = () => {
        stretchVideoElement.currentTime = Math.max(0, stretchVideoElement.duration - 0.1);
        stretchVideoElement.pause();
        console.log("Stretch video ended and frozen at last frame.");
        showStretchOverlay = true;
        console.log("Showing stretch overlay.");
        setTimeout(() => {
          startClass2Detection();
        }, 5000);
      };
    }
  }

  async function startClass2Detection() {
    try {
      console.log("Starting detection for Class 2.");
      detectingClass2 = true;
      await waitForTmPose();
      const tmPose = getTmPose();
      if (!tmPose) throw new Error("tmPose not loaded");

      webcam2 = new tmPose.Webcam(200, 200, true);
      await webcam2.setup();
      await webcam2.play();
      console.log("Webcam started for Class 2 detection.");
      requestAnimationFrame(loopClass2);
    } catch (e) {
      console.error("Failed to start Class 2 detection:", e);
    }
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
    const class2Pred = prediction.find((p: any) => p.className.toLowerCase() === "class 2");
    if (class2Pred && class2Pred.probability > 0.9) {
      console.log("Class 2 detected:", class2Pred.probability);
      detectingClass2 = false;
      if (webcam2) webcam2.stop();
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

    if (leapAudioElement) {
      leapAudioElement.play().catch((err) => console.error("Error playing leap audio:", err));
    }
    if (leapVideoElement) {
      leapVideoElement.play().catch((err) => console.error("Error playing leap video:", err));
      leapVideoElement.onended = () => {
        leapVideoElement.currentTime = Math.max(0, leapVideoElement.duration - 0.1);
        leapVideoElement.pause();
        console.log("Leap video ended and frozen at last frame.");
        showLeapOverlay = true;
        console.log("Showing leap overlay.");
        setTimeout(() => {
          startClass1Detection();
        }, 5000);
      };
    }
  }

  async function startClass1Detection() {
    try {
      console.log("Starting detection for Class 1.");
      detectingClass1 = true;
      await waitForTmPose();
      const tmPose = getTmPose();
      if (!tmPose) throw new Error("tmPose not loaded");

      webcam3 = new tmPose.Webcam(200, 200, true);
      await webcam3.setup();
      await webcam3.play();
      console.log("Webcam started for Class 1 detection.");
      requestAnimationFrame(loopClass1);
    } catch (e) {
      console.error("Failed to start Class 1 detection:", e);
    }
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
    const class1Pred = prediction.find((p: any) => p.className.toLowerCase() === "class 1");
    if (class1Pred && class1Pred.probability > 0.9) {
      console.log("Class 1 detected:", class1Pred.probability);
      detectingClass1 = false;
      if (webcam3) webcam3.stop();
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

    if (gatherAudioElement) {
      gatherAudioElement.play().catch((err) => console.error("Error playing gather audio:", err));
    }

    if (gatherVideoElement) {
      try {
        gatherVideoElement.pause();
        gatherVideoElement.currentTime = 0.04;
        console.log("Gather video set to second frame and paused.");
      } catch (e) {
        console.warn("Gather video init error:", e);
      }
      setTimeout(() => {
        gatherVideoElement.play().catch((err) => console.error("Error playing gather video:", err));
      }, 5000);

      gatherVideoElement.onended = () => {
        gatherVideoElement.currentTime = Math.max(0, gatherVideoElement.duration - 0.1);
        gatherVideoElement.pause();
        console.log("Gather video ended and frozen at last frame.");
        showGatherOverlay = true;
        console.log("Showing gather overlay.");
        setTimeout(() => {
          startClass3Detection();
        }, 5000);
      };
    }
  }

  async function startClass3Detection() {
    try {
      console.log("Starting detection for Class 3.");
      detectingClass3 = true;
      await waitForTmPose();
      const tmPose = getTmPose();
      if (!tmPose) throw new Error("tmPose not loaded");

      webcam4 = new tmPose.Webcam(200, 200, true);
      await webcam4.setup();
      await webcam4.play();
      console.log("Webcam started for Class 3 detection.");
      requestAnimationFrame(loopClass3);
    } catch (e) {
      console.error("Failed to start Class 3 detection:", e);
    }
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
    const class3Pred = prediction.find((p: any) => p.className.toLowerCase() === "class 3");
    if (class3Pred && class3Pred.probability > 0.9) {
      console.log("Class 3 detected:", class3Pred.probability);
      detectingClass3 = false;
      if (webcam4) webcam4.stop();
      goto("/listory"); // use absolute path
    } else {
      console.log("Class 3 not detected or confidence too low.");
    }
  }
</script>

<style>
  /* (unchanged styles from your version) */
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
    <video bind:this={videoElement} autoplay playsinline muted on:ended={handleVideoEnd}>
      <source src="https://res.cloudinary.com/dufnjfidt/video/upload/v1740774225/idle.move_ld61z5.mp4" type="video/mp4" />
      Your browser does not support the video tag.
    </video>
    <audio bind:this={idleAudioElement}>
      <source src="https://res.cloudinary.com/dufnjfidt/video/upload/v1740774221/idle_nrvgl6.mp3" type="audio/mp3" />
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
    <audio bind:this={stretchAudioElement} src="https://res.cloudinary.com/dufnjfidt/video/upload/v1740779999/stretch_jrgea3.mp3" autoplay></audio>
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
    <audio bind:this={leapAudioElement} src="https://res.cloudinary.com/dufnjfidt/video/upload/v1740781160/leap_rscjmf.mp3" autoplay></audio>
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
    <audio bind:this={gatherAudioElement} src="https://res.cloudinary.com/dufnjfidt/video/upload/v1740781887/gather_whe7tj.mp3" autoplay></audio>
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
