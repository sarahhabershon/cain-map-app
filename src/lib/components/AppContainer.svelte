<script lang="ts">
    import MapCircles from '$lib/components/map/Maplibre_circles.svelte' 
    import geoEurope from '$lib/data/all_events_no_nulls?raw'
    import borders1925 from '$lib/data/all_borders_1925.geojson?raw';
    import countryBounds from '$lib/data/country_bounds?raw'
    import BarChart from '$lib/components/barChart.svelte' 
    import ActorSelector from '$lib/components/filterSelect.svelte' 
    import CountrySelector from '$lib/components/countrySelect.svelte' 
    import ActiveFilters from '$lib/components/ActiveFilters.svelte' 
    import Search from '$lib/components/search.svelte' 
    import Info from '$lib/components/infoBox.svelte' 
    import type { LngLatBoundsLike } from 'maplibre-gl';
    import DownloadButton from '$lib/components/DownloadButton.svelte' 
    const europeGeoJson = JSON.parse(geoEurope)
    const borders = JSON.parse(borders1925)
    const boundingBoxes = JSON.parse(countryBounds)
    const mapStyle = "https://api.maptiler.com/maps/019ba32c-43d2-74ac-bdba-1768cc85c5c2/style.json?key=GDx9s6OzDP05pKKgG4wT"


    let mapZoom = $state(3)
    let toggle = $state(true)
    let actors = $state([])
    let subActors = $state([]) 
    let dates = $state([])
    let country = $state(null)
    let countryTimelineOnly = $state(false);

    let showSuggestions = $state(false);
    let countryOpen = $state(false);
    let filterOpen = $state(false);


    const uniqueCountries= [...new Set(
        europeGeoJson.features.map(f => f.properties.country_coded)
    )];



    const uniqueActors = [
    ...new Map(
        europeGeoJson.features.flatMap(f => [
            { group: f.properties.actor_group_a_reduced, country: f.properties.country_coded },
            { group: f.properties.actor_group_b_reduced, country: f.properties.country_coded }
        ])
        .filter(x => x.group && x.country)                  // remove empty entries
        .map(x => [`${x.group}::${x.country}`, x])         // map for uniqueness
    ).values()
    ];


    let availableGroups = $derived.by(() => {
        return [
            ...new Set(
                uniqueActors
                    .filter(a => !country || a.country === country) 
                    .map(a => a.group)
            )
        ];
    });


    const uniqueSubActors = [
        ...new Map(
            europeGeoJson.features.flatMap(f => [
                { name: f.properties.actor_a, country: f.properties.country_coded, group: f.properties.actor_group_a_reduced },
                { name: f.properties.actor_b, country: f.properties.country_coded, group: f.properties.actor_group_b_reduced },
            ])
            .filter(x => x.name && x.country && x.group)
            .map(x => [`${x.name}::${x.country}`, x])
        ).values()
    ];


    const availableSubActors = $derived.by(() => {
        return uniqueSubActors.filter(sa => {
            const countryOk = !country || sa.country === country;
            const groupOk   = actors.length === 0 || actors.includes(sa.group);
            return countryOk && groupOk;
        });
    });

    const europeBBox: LngLatBoundsLike = [-10.0, 24.5, 31.5, 61.5];

      // bounds is reactive and will update automatically based on country
    const bounds: LngLatBoundsLike = $derived.by(() => {
        if (!country) return europeBBox; // default to Europe

        if (!boundingBoxes) return europeBBox; // safeguard


        const b = boundingBoxes.find(b => b.country_coded === country);
        return b ? [b.min_lon, b.min_lat, b.max_lon, b.max_lat] : europeBBox;

    });



    function filterFeatures() {
    return europeGeoJson.features.filter(f => {

        // 5) COUNTRY FILTER
        const countryMatch =
            country
        ? f.properties.country_coded === country
        : true;


        // 1) SPECIFIC ACTOR–COUNTRY FILTER (takes priority)
        const specificActorMatch =
            subActors.length > 0 &&
            subActors.some(sa =>
                (f.properties.actor_a === sa.name && f.properties.country_coded === sa.country) ||
                (f.properties.actor_b === sa.name && f.properties.country_coded === sa.country)
            );

        // 2) REDUCED GROUP FILTER (used only if no subActors selected)
        const reducedActorMatch =
            subActors.length === 0 && actors.length > 0 &&
            (actors.includes(f.properties.actor_group_a_reduced) ||
             actors.includes(f.properties.actor_group_b_reduced));

        // 3) IF no actor filters active → allow everything
        const actorMatch =
            subActors.length > 0 ? specificActorMatch :     // ✔ priority
            actors.length > 0    ? reducedActorMatch  :     // ✔ fallback
                                  true;                     // ✔ no filter active

        // 4) DATE FILTER remains unchanged
        const dateMatch =
            dates.length === 2
                ? new Date(f.properties.date) >= dates[0] &&
                  new Date(f.properties.date) <= dates[1]
                : true;

        return actorMatch && dateMatch && countryMatch;
    });
}

let filteredData = $derived({ ...europeGeoJson, features: filterFeatures() });






</script>


<div class="container">
    <div class="toolbar-wrapper">
        <div id="toolbar">
            <div class="country-selector"><CountrySelector uniqueCountries={uniqueCountries} 
                bind:country={country}
                bind:countryOpen={countryOpen}
                bind:filterOpen={filterOpen}
                bind:showSuggestions={showSuggestions}/></div>
            <div class="actor-selector"><ActorSelector availableGroups={availableGroups} 
                bind:actors={actors} 
                bind:countryOpen={countryOpen}
                bind:filterOpen={filterOpen}
                bind:showSuggestions={showSuggestions}/></div>
            <div class="search-wrapper"><Search uniqueSubActors={availableSubActors}
                bind:subActors={subActors}
                bind:showSuggestions={showSuggestions}
                bind:countryOpen={countryOpen}
                bind:filterOpen={filterOpen} /></div>
        </div>
        <div id="filters-overlay">
            <ActiveFilters  bind:actors={actors} 
                bind:subActors={subActors} 
                bind:dates={dates}
                bind:country={country}
                bind:countryTimelineOnly={countryTimelineOnly}/>
        </div>
    </div>

    <DownloadButton
        actors={actors}
        subActors={subActors}
        dates={dates}
        country={country}
        filteredData={filteredData}
        borders={borders}
        bounds={bounds}
        />

    <MapCircles bind:zoom={mapZoom} 
        filteredData={filteredData}
        borders={borders}
        bounds = {bounds}
        mapStyle = {mapStyle}/> 
</div>


<div class="timeline-shell">
    <div class="info-button-wrapper">
        <Info bind:countryOpen={countryOpen}
            bind:filterOpen={filterOpen}
            bind:showSuggestions={showSuggestions}/>
        <DownloadButton
            actors={actors}
            subActors={subActors}
            dates={dates}
            country={country}
            filteredData={filteredData}
            borders={borders}
            bounds={bounds}
        />
    </div>

    <div id="timeline">
        <BarChart filteredData={filteredData} bind:dates = {dates} countryTimelineOnly={countryTimelineOnly} country={country}/>
    </div>
</div>



    


  <style>


    :global(html, body, #svelte) {
    height: 100%;
    margin: 0;
    padding: 0;
    font-family: 'Roboto Condensed', sans-serif;
    }

    .container {
        position: relative;
        width: 100%;
        height: 100vh;
        overflow: hidden;
        }

    .container :global(.maplibre-map) {
        position: absolute !important;
        inset: 0;               
        width: 100% !important;
        height: 100% !important;
        }


    /*Toolbar and filters layer */


    .toolbar-wrapper {
        position: relative;
        z-index: 30;
        overflow: visible;

        /* Center horizontally */
        /* left: 50%;
        transform: translateX(-50%); */
        margin-left: auto;
        margin-right: auto;
        top: 1rem;

        display: flex;
        flex-direction: column;
        gap: 0.5rem;

        /* width: 90%; */
        width: calc(100% - 2rem);
        max-width: 700px;
        }

    /* Toolbar itself */

    #toolbar {
        display: flex;
        width: 100%;
        flex-wrap: wrap;            
        align-items: center;
        gap: 0.5rem;
        overflow: visible;

        background: white;
        padding: 0.75rem;
        border-radius: 10px;
        box-shadow: 0 3px 12px rgba(0,0,0,0.18);

        box-sizing: border-box;
    }

    /* Country selector */
    #toolbar > :global(.country-selector) {
        flex: 0 0 40%;
    }

    /* Actor selector */
    #toolbar > :global(.actor-selector) {
        flex: 0 0 55%;
    }

    /* Search bar */
    #toolbar > :global(.search-wrapper) {
        flex: 0 0 100%;
        margin-top: 0.25rem;
    }


    @media (min-width: 641px) {
        #toolbar {
            flex-wrap: nowrap;
        }

        #toolbar > :global(.country-selector),
        #toolbar > :global(.actor-selector),
        #toolbar > :global(.search-wrapper) {
            flex: 1 1 auto;
            margin-top: 0;
        }
    }


    /*timeline chart */

    .timeline-shell {
        position: relative;
        bottom: 8rem;
        margin: 0 auto;
        z-index: 500;

        width: 90%;
        max-width: 1000px;
    }

    /* .timeline-shell {
        position: relative;
        bottom: 8rem;
        z-index: 25;

        width: 90%;
        max-width: 1000px;
        margin: 0 auto;
    } */

    

    #timeline {
        position: relative;
        z-index: 25;

        bottom: 2rem;
        /* left: 50%;
        transform: translateX(-50%); */
        margin: 0 auto;
        width: 90%;
        max-width: 1000px;

        background: white;
        padding: 0.5rem 1rem;
        border-radius: 10px;
        box-shadow: 0 -3px 12px rgba(0,0,0,0.2);
        }

    /*INFO*/

    .info-button-wrapper {
    position: relative;
    top: -2.5rem;
    z-index: 50;
}

</style>