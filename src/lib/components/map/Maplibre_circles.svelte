<script lang="ts">
  import { MapLibre, CircleLayer, LineLayer, GeoJSON, Popup} from 'svelte-maplibre';
  import type { LngLatBoundsLike } from 'maplibre-gl';
  import MapInstanceCapture from './MapInstanceCapture.svelte';

  let {zoom = $bindable(), filteredData, borders, country = $bindable(), boundingBoxes} = $props()


  // let bounds: LngLatBoundsLike = $state([-10.0, 24.5, 31.5, 61.5]);

  const europeBBox: LngLatBoundsLike = [-10.0, 24.5, 31.5, 61.5];

  // bounds is reactive and will update automatically based on country
 let bounds: LngLatBoundsLike = $derived.by(() => {
  if (!country) return europeBBox; // default to Europe

  if (!boundingBoxes) return europeBBox; // safeguard

  const b = boundingBoxes.find(b => b.country_coded === country);
  return b ? [b.min_lon, b.min_lat, b.max_lon, b.max_lat] : europeBBox;
});

</script>



 <!-- "https://api.maptiler.com/maps/019ba32c-43d2-74ac-bdba-1768cc85c5c2/style.json?key=GDx9s6OzDP05pKKgG4wT" -->


<MapLibre  
  style = "https://api.maptiler.com/maps/019ba32c-43d2-74ac-bdba-1768cc85c5c2/style.json?key=GDx9s6OzDP05pKKgG4wT"
  bind:zoom={zoom}
  bind:bounds={bounds}
  preserveDrawingBuffer={true}
>

<MapInstanceCapture />

  <GeoJSON
    id="borders-1925"
    data={borders}
  >
    <LineLayer
      id="borders-1925-line"
      paint={{
        'line-color': '#d87355',
        'line-width': [
          'interpolate',
          ['linear'],
          ['zoom'],
          0, 0.7,
          5, 1.2,
          10, 2
        ],
        'line-opacity': [
          'interpolate',
          ['linear'],
          ['zoom'],
          0, 0.1,
          5, 0.2,
          10, 0.3
        ]
      }}
    />
  </GeoJSON>

  
  <GeoJSON 
      id="europeMap" 
      data={filteredData}
      cluster = {{
          radius: 50,
          maxZoom: 12,
          properties: {
            event_count: ['+', ['get', 'point_count']]
          }
      }}
  >
    
  <CircleLayer
      id="clusters"
      source="europeMap"

      applyToClusters
      manageHoverState
      paint={{
        'circle-radius': [
            'interpolate',
            ['linear'],
            ['zoom'],
            5, [
              'interpolate', ['linear'], ['get', 'point_count'],
              1, 7,
              500, 35
            ],
            10, [
              'interpolate', ['linear'], ['get', 'point_count'],
              1, 10,
              500, 50
            ]
          ],
        'circle-color': [
            'interpolate',
            ['linear'],
            ['get', 'point_count'],
            2, '#fdae2a',
            20, '#f6973d',
            40, '#e9844b',
            80, '#d87355',
            200, '#d87355',
            420, '#d87355'
          ]
        }}
        filter={['has', 'point_count']}  
  />


<CircleLayer
  id="unclustered-points"
  source="europeMap"
  hoverCursor="pointer"
  paint={{
    'circle-radius': 5,
    'circle-color': '#fec604',
    'circle-stroke-width' : 1.5
  }}
  filter={['!', ['has', 'point_count']]}   
>
  
  
    <Popup openOn="click">
      {#snippet children({ data })}
        {#if data?.properties}

          <div class="map-popup">

            <div class="meta">
              <span>{data.properties.date}</span>
              <span>{data.properties.country}</span>
            </div>


            <div class="actors">
              <p>Event involving:</p>
              <div class="actor">{data.properties.actor_a}</div>
              <div class="actor">{data.properties.actor_b}</div>
            </div>


            {#if data.properties.deaths}
              <div class="deaths">Deaths: {data.properties.deaths}</div>
            {/if}

          </div> 

        {:else}
          <div class="map-popup">No data available</div>
        {/if}
      {/snippet}
    </Popup>

  </CircleLayer>
  </GeoJSON>
</MapLibre>


<style>


  /* Outer popup wrapper */
  .map-popup {
    font-family: var(--e-global-typography-text-font-family, "Roboto condensed"), sans-serif;
    padding: 10px 14px;
    display: flex;
    flex-direction: column;
    /* gap: 2px; */
    max-width: 230px;
  }

  /* Date + Country small metadata */
  .meta {
    display: flex;
    justify-content: space-between;
    font-size: 0.8rem;
    color: #555;
    opacity: 0.8;
    font-weight: 500;
  }

  /* Main actor names */
  .actors {
    display: flex;
    flex-direction: column;
    gap: 0px;
  }

  .actor {
    font-size: 0.85rem;
    font-weight: 500;
    color: #383C42;  
  }

  /* Death count */
  .deaths {
    font-size: 0.85rem;
    color: #D87355; 
    margin-top: 2px;
    font-weight: 500;
  }


</style>