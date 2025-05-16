<script lang="ts">
  import ScatterPlot from '$lib/components/ScatterPlot.svelte';

</script>

<svelte:head>
  <title>ABE2025 | Wytham Viewer</title>
</svelte:head>

<div class="page-container">
  <div class="plot-wrapper">
    <ScatterPlot />
  </div>
</div>

<style>
  .page-container {
    display: flex;
    flex-direction: column;
    height: 100%;
    width: 100%;
    box-sizing: border-box;
    position: relative; /* Added to properly position the overlay */
  }

  /* new: subtle film-grain overlay */
  .page-container::before {
    content: "";
    pointer-events: none;
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    background: url('/noisy.png') repeat;
    opacity: 0.15;
    mix-blend-mode: screen;
    z-index: 999;
  }



  .plot-wrapper {
    flex-grow: 1; /* Allows ScatterPlot container to fill available space */
    display: flex; /* Ensures child with height: 100% works correctly */
    min-height: 0; /* Fix for flexbox overflow issues with tall children */
  }
  
</style>
