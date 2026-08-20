<script lang="ts">
  import ReportGenerator from '$lib/components/report/ReportGenerator.svelte';

  let { actors = [], subActors = [], dates = [], country = null, filteredData, borders, bounds } = $props();
</script>

<ReportGenerator
  {actors}
  {subActors}
  {dates}
  {country}
  {filteredData}
  {borders}
  {bounds}
/>