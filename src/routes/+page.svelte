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

    $: map?.on("move", evt => mapViewChanged++);

    function getCoords (station) {
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
        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv");

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);// TODO arrivals


        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;// TODO departures
            station.totalTraffic = station.arrivals + station.departures; // TODO totalTraffic
            return station;
        });
    });

    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic) || 0])
        .range([0, 25]);

   
    
</script>

<h1>BikeWatching 🚴🏽‍♀️</h1>
<p>immersive, interactive map visualization of bike traffic in the Boston area during different times of the day.</p>

<div id="map" >
    <svg>
        {#key mapViewChanged}
        {#each stations as station}
            <circle { ...getCoords(station) } r="{radiusScale(station.totalTraffic)}" fill="steelblue" />
        {/each}
        {/key}
    </svg>
</div>


<style>
    @import url("$lib/global.css");
</style>