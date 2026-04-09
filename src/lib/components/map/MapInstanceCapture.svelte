<!-- <script lang="ts">
  import { getMapContext } from 'svelte-maplibre';

  console.log('MapInstanceCapture mounting...');
  const mapContext = getMapContext();
  console.log('mapContext in capture:', mapContext);

  $effect(() => {
    if (mapContext?.map) {
      (window as any).__map = mapContext.map;
      console.log('Map instance stored:', (window as any).__map);
    }
  });
</script> -->