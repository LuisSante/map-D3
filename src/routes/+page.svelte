<script>
	import { onMount } from "svelte";
	import * as d3 from "d3";
	import mapboxgl from "mapbox-gl";

	const TOKEN_API = import.meta.env.VITE_MAPBOX_TOKEN;
	mapboxgl.accessToken = TOKEN_API

	let map;
	let stations = [];
	let trips = [];
	let arrivals = new Map();
	let departures = new Map();
	let mapViewChanged = 0;
	let radiusScale;
	let timeFilter = -1;
	
	const departuresByMinute = Array.from({length: 1440}, () => []);
	const arrivalsByMinute = Array.from({length: 1440}, () => []);

	$: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});
	
	$: filteredDepartures = timeFilter === -1 
		? departures 
		: d3.rollup(filterByMinute(departuresByMinute, timeFilter), v => v.length, d => d.start_station_id);
	
	$: filteredArrivals = timeFilter === -1 
		? arrivals 
		: d3.rollup(filterByMinute(arrivalsByMinute, timeFilter), v => v.length, d => d.end_station_id);
	
	$: filteredStations = stations.map(station => {
		const id = station.id;
		const arr = filteredArrivals.get(id) ?? 0;
		const dep = filteredDepartures.get(id) ?? 0;
		return {
			...station,
			arrivals: arr,
			departures: dep,
			totalTraffic: arr + dep
		};
	});
	
	$: filteredRadiusScale = d3
		.scaleSqrt()
		.domain([0, d3.max(filteredStations, d => d.totalTraffic) || 0])
		.range(timeFilter === -1 ? [0, 25] : [3, 30]);

	function getCoords(station) {
		if (!map) return { cx: 0, cy: 0 };
		let point = new mapboxgl.LngLat(station.Long, station.Lat);
		let { x, y } = map.project(point);
		return { cx: x, cy: y };
	}
	
	function minutesSinceMidnight(date) {
		return date.getHours() * 60 + date.getMinutes();
	}
	
	function filterByMinute(tripsByMinute, minute) {
		if (minute === -1) return trips;
		
		// Normalize both to the [0, 1439] range
		let minMinute = (minute - 60 + 1440) % 1440;
		let maxMinute = (minute + 60) % 1440;

		if (minMinute > maxMinute) {
			let beforeMidnight = tripsByMinute.slice(minMinute);
			let afterMidnight = tripsByMinute.slice(0, maxMinute + 1);
			return beforeMidnight.concat(afterMidnight).flat();
		} else {
			return tripsByMinute.slice(minMinute, maxMinute + 1).flat();
		}
	}

	async function initMap() {
		map = new mapboxgl.Map({
			container: "map",
			center: [-71.09415, 42.36027], // Boston
			zoom: 12,
			style: "mapbox://styles/mapbox/streets-v12",
		});

		await new Promise((resolve) => map.on("load", resolve));

		map.addSource("boston_route", {
			type: "geojson",
			data: "/data/Existing_Bike_Network_2022.geojson",
		});

		map.addLayer({
			id: "boston_bikelanes",
			type: "line",
			source: "boston_route",
			paint: {
				"line-color": "rgba(0, 128, 0, 0.4)", // verde translúcido 40%
				"line-width": 3,
				"line-opacity": 1,
			},
		});

		map.on("move", () => {
			mapViewChanged++;
		});
	}

	async function loadStationData() {
		try {
			const csvUrl = "/data/bluebikes-stations.csv";
			const data = await d3.csv(csvUrl);

			stations = data.map((station) => ({
				id: station.Number,
				name: station.NAME,
				Lat: +station.Lat,
				Long: +station.Long,
				arrivals: 0,
				departures: 0,
				totalTraffic: 0,
			}));
		} catch (error) {
			console.error("Error loading station data:", error);
		}
	}

	async function loadStationDemand() {
		try {
			const csvUrl = "/data/bluebikes-traffic-2024-03.csv";
			const data = await d3.csv(csvUrl);

			trips = await d3.csv(csvUrl).then(trips => {
				for (let trip of trips) {
					trip.started_at = new Date(trip.started_at);
					trip.ended_at = new Date(trip.ended_at);
					
					// Fill the departure and arrival buckets
					let startedMinutes = minutesSinceMidnight(trip.started_at);
					departuresByMinute[startedMinutes].push(trip);
					
					let endedMinutes = minutesSinceMidnight(trip.ended_at);
					arrivalsByMinute[endedMinutes].push(trip);
				}
				return trips;
			});

			departures = d3.rollup(
				trips,
				(v) => v.length,
				(d) => d.start_station_id,
			);
			arrivals = d3.rollup(
				trips,
				(v) => v.length,
				(d) => d.end_station_id,
			);

			stations = stations.map((station) => {
				let id = station.id;
				station.arrivals = arrivals.get(id) ?? 0;
				station.departures = departures.get(id) ?? 0;
				station.totalTraffic = station.arrivals + station.departures;
				return station;
			});

			radiusScale = d3
				.scaleSqrt()
				.domain([0, d3.max(stations, (d) => d.totalTraffic) || 0])
				.range([0, 25]);
		} catch (error) {
			console.error("Error loading station demand:", error);
		}
	}

	onMount(async () => {
		await initMap();
		await loadStationData();
		await loadStationDemand();

		// Re-render stations al mover el mapa
		map.on("move", () => {
			mapViewChanged++;
		});
	});
</script>

<header>
    <h1>🚲 BikeWatch</h1>
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

<div id="map">
	<svg>
		{#key mapViewChanged}
			{#each filteredStations as station}
				<circle
					{...getCoords(station)}
					r={filteredRadiusScale ? filteredRadiusScale(station.totalTraffic) : 0}
				/>
			{/each}
		{/key}
	</svg>
</div>

<style>
	/* Reset default margin and padding on body to prevent overflow */
	:global(body) {
		margin: 0;
		padding: 0;
		overflow-x: hidden;
	}
	
	header {
		position: absolute;
		top: 10px;
		left: 10px;
		right: 10px;
		z-index: 10;
		background: rgba(255, 255, 255, 0.8);
		padding: 10px;
		border-radius: 5px;
		display: flex;
		gap: 1em;
		align-items: baseline;
	}
	
	header label {
		margin-left: auto;
	}
	
	header input {
		width: 200px;
	}
	
	header em {
		color: #777;
		font-style: italic;
	}
	
	#map {
		position: relative;
		width: 100%;  /* Changed from 100vw to 100% */
		height: 100vh;
		left: 0;      /* Explicitly set left to 0 */
		right: 0;     /* Explicitly set right to 0 */
		margin: 0;    /* Remove any margin */
	}

	#map svg {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
		z-index: 1;
	}

	svg circle {
		fill: steelblue;
		fill-opacity: 0.6;
		stroke: white;
		stroke-width: 1;
	}
	
	h1 {
		margin: 0;
	}
</style>