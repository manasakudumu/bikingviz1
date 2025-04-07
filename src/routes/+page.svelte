<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import {onMount} from "svelte";
    import * as d3 from "d3";

    mapboxgl.accessToken = "pk.eyJ1IjoibWFuYXNha3VkdW11IiwiYSI6ImNtOTJ5bnV6ZjBjZGIyam9scWxld2c5NmIifQ.90koBSX6cIpION_rK2vvLg";
    let stations=[];
    let trips=[];
    let map;
    let mapViewChanged = 0;
    let timeFilter =-1;
    const departuresByMinute = Array.from({length: 1440}, () => []);
    const arrivalsByMinute = Array.from({length: 1440}, () => []);


    $: map?.on("move", evt => mapViewChanged++);
    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});


    function getCoords(station){
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }
    

    async function initMap(){
        map = new mapboxgl.Map({
            container: "map",
            style: "mapbox://styles/mapbox/streets-v12",
            zoom:12,
            center:[-71.09185737389886,42.360116874639395],
        });

        await new Promise(resolve => map.on("load", resolve));
        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });
        map.addLayer({
            id: "boston", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "boston_route", // The id we specified in `addSource()`
            paint: {
                "line-color": "green",
                "line-width": 3,
                "line-opacity": 0.4
                // paint params, e.g. colors, thickness, etc.
            },
        });


        // Cambridge routes
        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson", 
        });
        map.addLayer({
            id: "cambridge", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "cambridge_route", // The id we specified in `addSource()`
            paint: {
                "line-color": "green",
                "line-width": 3,
                "line-opacity": 0.4
                // paint params, e.g. colors, thickness, etc.
            },
        });


        map.on("move", () => {
			stations = [...stations]; // trigger reactivity
		});
    }

    let departures = new Map();
    let arrivals = new Map();


    onMount(async() => {
        initMap();
        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at)
                trip.ended_at = new Date(trip.ended_at)
                
                let startedMinutes = minutesSinceMidnight(trip.started_at);
                departuresByMinute[startedMinutes].push(trip);

                let finishedMinutes = minutesSinceMidnight(trip.ended_at);
                arrivalsByMinute[finishedMinutes].push(trip);
            }
            return trips;
        });

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);


        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures; 
            return station;
        }); 

        

// TODO: Same for arrivals


    });
   


    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    // $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
    //     let startedMinutes = minutesSinceMidnight(trip.started_at);
    //     let endedMinutes = minutesSinceMidnight(trip.ended_at);
    //     return Math.abs(startedMinutes - timeFilter) <= 60
    //         || Math.abs(endedMinutes - timeFilter) <= 60;
    // });

    function filterByMinute (tripsByMinute, minute) {
        // Normalize both to the [0, 1439] range
        // % is the remainder operator: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Remainder
        let minMinute = (minute - 60 + 1440) % 1440;
        let maxMinute = (minute + 60) % 1440;

        if (minMinute > maxMinute) {
            let beforeMidnight = tripsByMinute.slice(minMinute);
            let afterMidnight = tripsByMinute.slice(0, maxMinute);
            return beforeMidnight.concat(afterMidnight).flat();
        }
        else {
            return tripsByMinute.slice(minMinute, maxMinute).flat();
        }
    }

    $: filteredDepartures = d3.rollup(filterByMinute(departuresByMinute, timeFilter), v => v.length, d => d.start_station_id);
    $: filteredArrivals = d3.rollup(filterByMinute(arrivalsByMinute, timeFilter), v => v.length, d => d.end_station_id);



    $: filteredStations = stations.map(station => {
        const id = station.Number;
        const arr = filteredArrivals.get(id) ?? 0;
        const dep = filteredDepartures.get(id) ?? 0;
        return {
            ...station,
            arrivals: arr,
            departures: dep,
            totalTraffic: arr + dep
        };
    });

    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(filteredStations, d => d.totalTraffic) || 0])
        .range([0, 25]);

    //Make circle color depend on traffic flow
    let stationFlow = d3.scaleQuantize()
        .domain([0, 1])
        .range([0, 0.5, 1]);


    //Adding isochrones
    const urlBase = 'https://api.mapbox.com/isochrone/v1/mapbox/';
    const profile = 'cycling';
    const minutes = [5, 10, 15, 20];
    const contourColors = [
        "03045e",
        "0077b6",
        "00b4d8",
        "90e0ef"
    ]
    let isochrone = null;

    async function getIso(lon, lat) {
        const base = `${urlBase}${profile}/${lon},${lat}`;
        const params = new URLSearchParams({
            contours_minutes: minutes.join(','),
            contours_colors: contourColors.join(','),
            polygons: 'true',
            access_token: mapboxgl.accessToken
        });
        const url = `${base}?${params.toString()}`;

        const query = await fetch(url, { method: 'GET' });
        isochrone = await query.json();
    }
    getIso(-71.09415, 42.36027);


    //selecting a station
    let selectedStation = null;

    //loading isochrones
    function geoJSONPolygonToPath(feature) {
	const path = d3.path();
	const rings = feature.geometry.coordinates;

	for (const ring of rings) {
		for (let i = 0; i < ring.length; i++) {
			const [lng, lat] = ring[i];
			const { x, y } = map.project([lng, lat]);
			if (i === 0) path.moveTo(x, y);
			else path.lineTo(x, y);
		}
		path.closePath();
	}
	return path.toString();
}








</script>

<header>
    <h1>BikeWatching 🚴🏽‍♀️</h1>
    <label>
        Filter by time:
        <input type="range" min="-1" max="1440" bind:value={timeFilter} />
        {#if timeFilter !== -1}
            <time style="display: block">
                {timeFilterLabel}
            </time>
        {:else}
            <em style="display: block">(any time)</em>
        {/if}
    </label>
</header>


<p>immersive, interactive map visualization of bike traffic in the Boston area during different times of the day.</p>

<div id="map">
    <svg>
        {#key mapViewChanged}
            {#if isochrone}
                {#each isochrone.features as feature}
                    <path
                            d={geoJSONPolygonToPath(feature)}
                            fill={feature.properties.fillColor}
                            fill-opacity="0.2"
                            stroke="#000000"
                            stroke-opacity="0.5"
                            stroke-width="1"
                    >
                        <title>{feature.properties.contour} minutes of biking</title>
                    </path>
                {/each}
            {/if}
            {#each filteredStations as station}
            <circle { ...getCoords(station) } r={radiusScale(station.totalTraffic)} fill="steelblue" fill-opacity=0.6 stroke="white" style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }" class={station?.Number === selectedStation?.Number ? "selected" : ""}
            on:mousedown={() => selectedStation = selectedStation?.Number !== station?.Number ? station : null}>
                <title>{station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)</title>
            </circle>
            {/each}
            {/key}
    </svg>
</div>

<!-- adding a legend -->
<div class="legend">
	<span class="legend-title">LEGEND:</span>
	<div class="legend-item" style="--departure-ratio: 1">
		<span class="swatch"></span>
		<span class="label">More departures</span>
	</div>

	<div class="legend-item" style="--departure-ratio: 0.5">
		<span class="swatch"></span>
		<span class="label">Balanced</span>
	</div>

	<div class="legend-item" style="--departure-ratio: 0">
		<span class="swatch"></span>
		<span class="label">More arrivals</span>
	</div>
</div>




<style>
    @import url("$lib/global.css");
</style>