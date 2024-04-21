<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css" 
    import { onMount } from "svelte";
    import * as d3 from 'd3';

    let bike_styling = {line_width: 2, line_color: "green", line_opacity: 0.4}
    let stations = [];
    let trips = [];
    let radiusScale;
    let stationFlow; 
    let departures;
    let arrivals;
    // let timeFilter = -1;
    // let filteredTrips;
    // let filteredStations;
    // let filteredArrivals;
    // let filteredDepartures;
    // let timeFilterLabel;
    let map;
    let mapViewChanged = 0;
    
    mapboxgl.accessToken = "pk.eyJ1Ijoib2xpdmlhOTg3IiwiYSI6ImNsdmEyb3A4bzEzd2cybG54YTUxMDh6cm8ifQ.hb_oq1MPCOODXago8629uw";
    stationFlow = d3.scaleQuantize().domain( [0, 1] ).range( [0, 0.5, 1] );
    $: map?.on("move", evt => mapViewChanged++);
    $: radiusScale = d3.scaleSqrt().domain([0, d3.max(stations, d => d.totalTraffic) ]).range([0, 35]);
    // $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {timeStyle: "short"});
    // $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
    //     let startedMinutes = minutesSinceMidnight(trip.started_at);
    //     let endedMinutes = minutesSinceMidnight(trip.ended_at);
    //     return Math.abs(startedMinutes - timeFilter) <= 60
    //             || Math.abs(endedMinutes - timeFilter) <= 60;
    // }); 
    // $: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
    // $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);
    // $: filteredStations = stations.map(station => {
    //         station = {...station};
    //         let id = station.Number;
    //         station.arrivals = filteredArrivals.get(id) ?? 0;
    //         station.departures = filteredDepartures.get(id) ?? 0;
    //         station.totalTraffic = station.arrivals + station.departures;

    //         return station;
    // });

    onMount( async() => {
        trips = await d3.csv("bluebikes-traffic-2024-03.csv").then(trips => {
            for (let trip of trips)
            {
                trip.started_at = new Date(trip.started_at);
                trip.ended_at = new Date(trip.ended_at);
            }

            return trips;
        });

        stations = await d3.csv("bluebikes-stations.csv", row=>({
                ...row,
                Lat: Number(row.Lat),
                Long: Number(row.Long),
                total_docks: Number(row.Total_Docks),
            })
        );

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;

            return station;
        });

        map = new mapboxgl.Map({
            container: 'map',
            style: 'mapbox://styles/olivia987/cluuu60oe00q101qs4jrxedq9',
            center: [ -71.0961891, 42.3591965 ],
            zoom: 12,
        });

        await new Promise(resolve => map.on("load", resolve));
        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "Cambridge_Bike_Lanes.geojson",
        })

        map.addLayer({
            id:'boston_bike_layer',
            type: 'line',
            source: "boston_route",
            paint: {
                'line-color': bike_styling.line_color,
                'line-width': bike_styling.line_width,
                'line-opacity': bike_styling.line_opacity,
            }
        });

        map.addLayer({
            id: "cambridge_bike_layer",
            type: "line",
            source: "cambridge_route",
            paint: {
                'line-color': bike_styling.line_color,
                'line-width': bike_styling.line_width,
                'line-opacity': bike_styling.line_opacity,
            }
        });
    });

    // $: console.log(map);

    function getCoords (station)
    {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);

        return {cx: x, cy: y};
    }

    function minutesSinceMidnight (date)
    {   
        return date.getHours() * 60 + date.getMinutes(); // converts time of day to minutes
    }
</script>

<style>
    /* @import url("$lib/global.css"); */
    /* #map{
        flex: 1;
    } */

    #map svg{
        /* position:absolute; */
        z-index: 1; 
        /* z-index being 1, Puts the svg element in front of the map */
        /* to use z-index you need to you position property */
        width: 300px;
        height: 300px;
        pointer-events: none;

        circle{
            fill-opacity: 60%;
            stroke: white;
            pointer-events: auto;
            --color-departures: steelblue;
            --color-arrivals: darkorange;
            --color: color-mix(
                in oklch,
                var(--color-departures) calc(100% * var(--departure-ratio)),
                var(--color-arrivals)
            );
            fill: var(--color);
        }
    }

    header{
        display: flex;
        gap: 1em;
        align-items: baseline;
        margin-bottom: 0px;
        /* margin-left: auto; */

        h1{
            margin-right: auto;
        }

        input{
            width: 500px;
        }
    
    }

    .slider{
        display: grid;
        grid-template-rows: 5fr 5fr;
        margin-left: auto;
        margin-top: 1px;
        margin-bottom: 15px;
        /* margin-right: auto; */
    }

    em{
        font-style: italic;
        color:lightslategrey;
        margin-right:0px;
    }
    
    time{
        transition: 500ms;
    }

    .legend{
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        border: 10px solid var(--color);
        background-color: var(--color);
        display: grid;
        grid-template-columns: 10fr 10fr 10fr;
        margin-block: 10px;
        margin: 10px;

        .balance{
            background-color: rgb(196, 116, 196);
            column-gap:10px;
            color: white;
            font-style: bold;
            text-align: center;
            
        }
        .arrive{
            background-color: darkorange;
            column-gap: 10px;
            color:white;
            font-style: bold;
            text-align: center;
            margin-left: 5px;
        }
        .depart{
            background-color: steelblue;
            margin-right: 5px;
            font-style: bold;
            text-align: center;
            color: white;
        }
    }

</style>

<header>
    <h1> 🚴🏼‍♀️ Bike Watching</h1>
    <!-- <div class="slider">
        <label>
            <strong> Filter by time: </strong> 
            <input type="range" min= -1 max= 1440 bind:value={timeFilter}> 
        </label>
    
        {#if timeFilter === -1}
            <em>(any time)</em>
        {/if}
        
        {#if timeFilter !== -1 }
            <time>{timeFilterLabel}</time>
        {/if}
    </div> -->
    
</header>

<div id='map'>
    <svg>
        {#key mapViewChanged}
            {#each stations as station}
            <!-- {#each filteredStations as station} -->
                <circle 
                    { ...getCoords(station) } r={ radiusScale(station.totalTraffic) } fill="steelblue"
                    style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }"
                >
                    <title class="tooltip">{station.totalTraffic} trips ({station.arrivals} arrivals, {station.departures} departures)</title>
                </circle>
                <!-- {console.log("inside for loop: ", station)} -->
            {/each}
        {/key}
    </svg>
</div>
<div class="legend">
    <div class="depart" style="--departure-ratio: {1}">More Departures</div>
    <div class="balance" style="--departure-ratio: {0.5}">Balanced</div>
    <div class="arrive" style="--departure-ratio: {0}">More Arrivals</div>
</div>
