<script lang="ts">
  // Svelte 5: declare the children snippet
  import type { Snippet } from 'svelte';
  export let children: Snippet;
</script>

<svelte:head>
  <!-- TFJS + Teachable Machine Pose (UMD) -->
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@1.3.1/dist/tf.min.js" defer></script>
  <script src="https://cdn.jsdelivr.net/npm/@teachablemachine/pose@0.8.6/dist/teachablemachine-pose.min.js" defer></script>
</svelte:head>

{@render children()}

