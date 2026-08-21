<script lang="ts">
  import ReportGenerator from '$lib/components/report/ReportGenerator.svelte';

  let { actors = [], subActors = [], dates = [], country = null, filteredData, borders, bounds } = $props();
</script>

{#if country}
  <div class="download-wrapper">
    <ReportGenerator
      {actors}
      {subActors}
      {dates}
      {country}
      {filteredData}
      {borders}
      {bounds}
    />
    <span class="tooltip">Download report</span>
  </div>
{/if}

<style>
  .download-wrapper {
    position: absolute;
    bottom: 20px;
    right: 80px;
    display: flex;
    align-items: center;
  }

  .tooltip {
    position: absolute;
    bottom: 110%;
    left: 50%;
    transform: translateX(-50%);
    background: rgba(0, 0, 0, 0.75);
    color: white;
    font-family: 'Roboto Condensed', sans-serif;
    font-size: 0.75rem;
    padding: 4px 8px;
    border-radius: 4px;
    white-space: nowrap;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.2s ease;
  }

  .download-wrapper:hover .tooltip {
    opacity: 1;
  }
</style>