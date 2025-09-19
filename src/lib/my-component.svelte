<script>
    import { onMount, onDestroy } from "svelte";
    import { Tween } from 'svelte/motion';
    import { cubicInOut } from 'svelte/easing';
    import { goto } from '$app/navigation'; // Import the `goto` function

    let currentImage = 0;
    let interval;
    let bgHeight = "100vh";  
    let bgWidth = "100vw";

    let audio;
    let isPlaying = false;

    // Mouse Move Effect
    let x = 0;
    let y = 0;
    export let dataspeed = 0.5;

    // Moving text animation
    let yPosition = new Tween(0, { duration: 600, easing: cubicInOut });

    const domousemove = (e) => {
        x = (window.innerWidth - e.pageX * dataspeed) / 100;
        y = (window.innerHeight - e.pageY * dataspeed) / 100;

        checkCollision(); // ADDED: Check collision on mouse move
    };

    function startAudio() {
        if (audio && !isPlaying) {
            audio.play().then(() => {
                isPlaying = true;
                console.log("✅ Audio started!");
            }).catch(error => {
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

    // Draggable Bigman
    let bigmanX = 65;
    let isDragging = false;
    let offsetX = 0;

    // ADDED: references to DOM elements for collision
    let bigmanEl;
    let mousemoveEl;
    let showGreeting = false; // ADDED: track greeting box visibility

    const startDrag = (event) => {
        event.preventDefault();
        isDragging = true;
        const container = document.querySelector(".bigman-container");
        const containerRect = container.getBoundingClientRect();
        offsetX = event.clientX - (containerRect.left + (bigmanX / 100) * containerRect.width);
        window.addEventListener("mousemove", onDrag);
        window.addEventListener("mouseup", stopDrag);
    };

    const onDrag = (event) => {
        if (isDragging) {
            const container = document.querySelector(".bigman-container");
            const containerRect = container.getBoundingClientRect();
            bigmanX = ((event.clientX - containerRect.left - offsetX) / containerRect.width) * 100;
            bigmanX = Math.max(0, Math.min(100, bigmanX));

            checkCollision(); // ADDED: Check collision on drag
        }
    };

    const stopDrag = () => {
        isDragging = false;
        window.removeEventListener("mousemove", onDrag);
        window.removeEventListener("mouseup", stopDrag);
    };

    const calculateScale = () => {
        const minX = 20;
        const maxX = 80;
        const normalizedX = 1 - Math.max(0, Math.min(1, (bigmanX - minX) / (maxX - minX)));
        return 0.93 - (normalizedX * 0.7);
    };

    const calculateHeight = () => {
        const minX = 20;
        const maxX = 80;
        const normalizedX = 1 - Math.max(0, Math.min(1, (bigmanX - minX) / (maxX - minX)));
        return (normalizedX * -35) - 60;
    };

    // ADDED: Collision-checking function
    function checkCollision() {
        if (!bigmanEl || !mousemoveEl) return;

        const bigmanRect = bigmanEl.getBoundingClientRect();
        const mouseRect = mousemoveEl.getBoundingClientRect();

        // Basic bounding-box overlap check
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

   // ADDED: function to go to the "greeting" page
   function greet() {
      goto('greeting', { replaceState: true, noScroll: true });
   }
</script>

<style>
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
    }

    .bigman-container {
        position: absolute;
        top: 0;
        left: 0;
        width: 100vw;
        height: 100vh;
        pointer-events: none;
        overflow: hidden;
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

    /* Moving text */
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
    }

    /* ADDED: Greeting box styling, similar to moving text but not moving */
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
        z-index: 999; /* Ensure it's on top */
    }

        /* Style the button to look more clickable */
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
</style>

<!-- Do not remove anything below, only add: -->

<svelte:window on:mousemove={startAudio} on:mousemove={domousemove} />

<img
    class="background"
    src="https://res.cloudinary.com/dufnjfidt/image/upload/v1738787371/Section4-1_background_mytehd.jpg"
    alt="Background"
    on:load={() => {
      bgHeight = event.target.clientHeight + 'px';
      bgWidth = event.target.clientWidth + 'px';
    }}
    style="--bg-height: {bgHeight};"
/>

<div class="background-overlay" style="height: {bgHeight};"></div>

<div class="overlay-container" style="height: {bgHeight};">
    <img class="human {currentImage === 0 ? 'visible' : ''}" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1738788835/human1_k8n7fg.png" alt="Human 1"/>
    <img class="human {currentImage === 1 ? 'visible' : ''}" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1738788836/human2_eoipmq.png" alt="Human 2"/>
    <img class="human {currentImage === 2 ? 'visible' : ''}" src="https://res.cloudinary.com/dufnjfidt/image/upload/v1738788836/human3_kd7lke.png" alt="Human 3"/>
</div>

 <!-- ADDED: reference for bounding box -->
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
    <source src="https://res.cloudinary.com/dufnjfidt/video/upload/v1738790949/156480__teokgaming__park-with-traffic-and-chatter_sxychj.mp3" type="audio/mp3"/>
</audio>

<!-- ADDED: reference for bounding box -->
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

<!-- ADDED: Greeting box, only appears if showGreeting is true -->
{#if showGreeting}
    <div class="greeting-box">
        They seems like a bunch of nice people, but I am a bit nervous.
        <button class="greeting-btn" on:click={greet}>
            Greet the group
        </button>
    </div>
{/if}

