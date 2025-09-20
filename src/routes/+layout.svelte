<script lang="ts">
	import '../app.css';
	let { children } = $props();
</script>
<!-- src/routes/+layout.svelte -->
<svelte:head>
  <!-- TFJS 1.3.1 + Teachable Machine Pose UMD builds -->
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@1.3.1/dist/tf.min.js" defer></script>
  <script src="https://cdn.jsdelivr.net/npm/@teachablemachine/pose@0.8.6/dist/teachablemachine-pose.min.js" defer></script>
</svelte:head>

<slot />


{@render children()}
