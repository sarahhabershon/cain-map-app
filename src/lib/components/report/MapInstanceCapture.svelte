<script lang="ts">
  import { getMapContext } from 'svelte-maplibre';

  let { oncapture } = $props();

  const mapContext = getMapContext();
  console.log('MapInstanceCapture init, mapContext:', mapContext);

  $effect(() => {
    console.log('effect ran, mapContext?.map:', mapContext?.map);
    if (mapContext?.map) {
      mapContext.map.once('idle', () => {
        const canvas = mapContext.map.getCanvas();
        const dataUrl = canvas.toDataURL('image/png');
        oncapture?.(dataUrl);
      });
    }
  });
</script>