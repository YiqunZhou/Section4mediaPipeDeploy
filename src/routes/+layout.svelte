<!-- src/routes/+layout.svelte -->
<svelte:head>
  <!-- TFJS + Teachable Machine Pose (UMD) -->
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@1.3.1/dist/tf.min.js" defer></script>
  <script src="https://cdn.jsdelivr.net/npm/@teachablemachine/pose@0.8.6/dist/teachablemachine-pose.min.js" defer></script>
</svelte:head>

{@render children()}

