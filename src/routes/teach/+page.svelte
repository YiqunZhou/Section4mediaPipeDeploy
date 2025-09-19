<script>
    import { onMount } from "svelte";
    let video1, video2;
    let model, webcam, maxPredictions;

    const video1URL = "https://res.cloudinary.com/dufnjfidt/video/upload/v1739290840/move1_gj4a9h.mp4";
    const video2URL = "https://res.cloudinary.com/dufnjfidt/video/upload/v1739290839/move2_hy1o0l.mp4";
    const modelURL = "https://teachablemachine.withgoogle.com/models/9e75LYecs/model.json";
    const metadataURL = "https://teachablemachine.withgoogle.com/models/9e75LYecs/metadata.json";

    let detecting = false;

    async function loadModel() {
        const tmPose = await import("@teachablemachine/pose");
        model = await tmPose.load(modelURL, metadataURL);
        maxPredictions = model.getTotalClasses();

        webcam = new tmPose.Webcam(200, 200, true);
        await webcam.setup();
        await webcam.play();
        requestAnimationFrame(loop);
    }

    async function loop() {
        if (!detecting) return;
        webcam.update();
        const { pose, posenetOutput } = await model.estimatePose(webcam.canvas);
        const prediction = await model.predict(posenetOutput);

        const class1Confidence = prediction[0].probability; // Class 1 (Target pose)
        const class2Confidence = prediction[1].probability; // Class 2 (Neutral/Background)

        console.log(`Class 1: ${class1Confidence}, Class 2: ${class2Confidence}`);

        // Lower Class 2 confidence threshold to 50% (detect when Class 2 > 50%)
        if (class1Confidence > 0.8 || class2Confidence > 0.5) {  
            detecting = false; 
            startSecondVideo();
        } else {
            requestAnimationFrame(loop);
        }
    }

    function startSecondVideo() {
        video1.style.display = "none";
        video2.style.display = "block";
        video2.play();
    }

    function onVideo1Ended() {
        video1.currentTime = video1.duration - 0.1; // Stop at last frame
        video1.pause();
        detecting = true;
        loadModel();
    }

    onMount(() => {
        video1.addEventListener("ended", onVideo1Ended);
    });
</script>

<video bind:this={video1} src={video1URL} autoplay muted playsinline></video>
<video bind:this={video2} src={video2URL} muted style="display: none;" playsinline></video>
