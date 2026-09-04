<script lang="ts">
  import { MapLibre, GeoJSON, CircleLayer, LineLayer } from 'svelte-maplibre';
  import MapInstanceCapture from './MapInstanceCapture.svelte';
  import type { LngLatBoundsLike } from 'maplibre-gl';

  let { filteredData, borders, bounds, oncapture, width, height} = $props();
  
</script>

<!-- <div style="position: fixed; top: 0; left: 0; width: 1200px; height: 800px; z-index: 99999; border: 5px solid red;"> -->

<div style="position: absolute; top: -9999px; left: -9999px; width: {width}px; height: {height}px;">
  <MapLibre
    style="https://api.maptiler.com/maps/019ba32c-43d2-74ac-bdba-1768cc85c5c2/style.json?key=GDx9s6OzDP05pKKgG4wT"
    bind:bounds={bounds}
    preserveDrawingBuffer={true}
  >
    <MapInstanceCapture oncapture={oncapture} />

    <GeoJSON id="borders-1925-offscreen" data={borders}>
      <LineLayer
        id="borders-1925-line-offscreen"
        paint={{
          'line-color': '#d87355',
          'line-width': [
            'interpolate', ['linear'], ['zoom'],
            0, 1.5,
            5, 2.5,
            10, 4
          ],
          'line-opacity': [
            'interpolate', ['linear'], ['zoom'],
            0, 0.4,
            5, 0.6,
            10, 0.8
          ]
        }}
      />
    </GeoJSON>

    <GeoJSON
      id="europeMap-offscreen"
      data={filteredData}
      cluster={{
        radius: 50,
        maxZoom: 12,
        properties: { event_count: ['+', ['get', 'point_count']] }
      }}
    >
      <CircleLayer
        id="clusters-offscreen"
        source="europeMap-offscreen"
        applyToClusters
        paint={{
          'circle-radius': [
            'interpolate', ['linear'], ['zoom'],
            5, ['interpolate', ['linear'], ['get', 'point_count'], 1, 7, 500, 35],
            10, ['interpolate', ['linear'], ['get', 'point_count'], 1, 10, 500, 50]
          ],
          'circle-color': [
            'interpolate', ['linear'], ['get', 'point_count'],
            2, '#fdae2a', 20, '#f6973d', 40, '#e9844b', 80, '#d87355', 420, '#d87355'
          ]
        }}
        filter={['has', 'point_count']}
      />

      <CircleLayer
        id="unclustered-points-offscreen"
        source="europeMap-offscreen"
        paint={{
          'circle-radius': 5,
          'circle-color': '#fec604',
          'circle-stroke-width': 1.5
        }}
        filter={['!', ['has', 'point_count']]}
      />
    </GeoJSON>
  </MapLibre>
</div>