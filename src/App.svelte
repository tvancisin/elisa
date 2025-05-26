<script>
  import * as d3 from "d3";
  import {
    getCSV,
    createGeoJSONCircle,
    construct_line,
    construct_points,
  } from "./utils.js";
  import { afterUpdate, onMount } from "svelte";
  import mapboxgl from "mapbox-gl";
  import "mapbox-gl/dist/mapbox-gl.css";

  let width = 400;
  let height = 400;

  $: x_scale = d3
    .scaleLinear()
    .domain([0, 10000000])
    .range([100, width - 20]);
  $: y_scale = d3
    .scaleLinear()
    .domain([0, 100])
    .range([height - 20, 20]);

  let blah = createGeoJSONCircle([67.709953, 33.93911], 2000);
  let krava = createGeoJSONCircle([67.709953, 33.93911], 6500);
  console.log(blah);

  let mapContainer;
  mapboxgl.accessToken =
    "pk.eyJ1Ijoic2FzaGFnYXJpYmFsZHkiLCJhIjoiY2xyajRlczBlMDhqMTJpcXF3dHJhdTVsNyJ9.P_6mX_qbcbxLDS1o_SxpFg";

  // load data
  let geo;
  let map;
  let path = ["./elisa_map.csv"];
  let conflict_groups;
  getCSV(path).then((data) => {
    geo = data[0];
    conflict_groups = d3.groups(geo, (d) => d.conflict_country);
    console.log(conflict_groups);
  });

  onMount(() => {
    map = new mapboxgl.Map({
      container: mapContainer,
      style: "mapbox://styles/sashagaribaldy/cm6ktkpxn00m901s2fo0bh1e4",
      projection: "naturalEarth",
      center: [0, 0],
      zoom: 1.5,
    });

    map.on("load", () => {
      if (conflict_groups[0][1]) {
        const lines = construct_line(conflict_groups[0][1]);
        const points = construct_points(conflict_groups[0][1]);
        console.log(blah);

        map.addSource("polygon", {
          type: "geojson",
          data: blah,
        });

        map.addLayer({
          id: "polygon",
          type: "fill",
          source: "polygon",
          paint: {
            "fill-color": "gray",
            "fill-opacity": 0.3,
          },
        });

        map.addSource("polygon3", {
          type: "geojson",
          data: krava,
        });

        map.addLayer({
          id: "polygon3",
          type: "fill",
          source: "polygon3",
          paint: {
            "fill-color": "gray",
            "fill-opacity": 0.3,
          },
        });

        map.addSource("curved-line", {
          type: "geojson",
          data: lines,
        });

        map.addLayer({
          id: "curved-line",
          type: "line",
          source: "curved-line",
          paint: {
            "line-color": "white",
            "line-width": 1,
          },
        });

        map.addSource("points", {
          type: "geojson",
          data: points, // output of your `construct_points()` function
        });

        map.addLayer({
          id: "point-layer",
          type: "circle",
          source: "points",
          paint: {
            "circle-radius": [
              "interpolate",
              ["linear"],
              ["get", "agreements_sum"],
              0,
              0, // agreements_sum = 0 → radius = 0
              1,
              5, // agreements_sum = 1 → radius = 5
              5,
              10, // agreements_sum = 10 → radius = 15
              10,
              20, // agreements_sum = 50 → radius = 25 (adjust scale as needed)
            ],
            "circle-color": "#FF5722",
            "circle-opacity": 0.5,
            "circle-stroke-width": 1,
            "circle-stroke-color": "#ffffff",
          },
        });
      }
    });

    return () => map.remove();
  });

  function updateMapData(conflictData) {
    const entry = conflict_groups.find(([name]) => name === conflictData);
    let update_data = entry[1];

    if (!map || !map.isStyleLoaded()) return;

    const newLines = construct_line(update_data);
    const newPoints = construct_points(update_data);
    let newBlah = createGeoJSONCircle(
      [+update_data[0].longitude_con, +update_data[0].latitude_con],
      2000,
    );
    let newKrava = createGeoJSONCircle(
      [+update_data[0].longitude_con, +update_data[0].latitude_con],
      6500,
    );

    const lineSource = map.getSource("curved-line");
    const pointSource = map.getSource("points");
    const polygonSource = map.getSource("polygon");
    const polygon3Source = map.getSource("polygon3");

    if (lineSource && pointSource && polygonSource) {
      lineSource.setData(newLines);
      pointSource.setData(newPoints);
      polygonSource.setData(newBlah);
      polygon3Source.setData(newKrava);
    } else {
      console.warn("Map sources not found.");
    }
  }

  // code for arcs
  let arcs = [];
  const startX = 50;
  $: if (geo) {
    const parsedData = geo
      .map((d) => {
        const distance = +d.dist_mediation_conflict;
        const agreements = +d.agreements_sum;

        if (!isFinite(distance) || isNaN(distance) || distance <= 0) {
          return null;
        }

        return {
          ...d,
          distance,
          agreements:
            isFinite(agreements) && !isNaN(agreements) ? agreements : 0,
        };
      })
      .filter((d) => d !== null);

    const maxDistance = d3.max(parsedData, (d) => d.distance);
    const maxAgreements = d3.max(parsedData, (d) => d.agreements);

    const xScale = d3
      .scaleLinear()
      .domain([0, maxDistance])
      .range([startX, width - 50]);

    const rScale = d3
      .scaleSqrt() // use square root for visual area scaling
      .domain([0, maxAgreements])
      .range([0, 20]); // adjust min/max radius as needed

    arcs = parsedData.map((d) => {
      const xEnd = xScale(d.distance);
      const radius = (xEnd - startX) / 2;

      const arcPath = `
        M ${startX},${height - 50}
        A ${radius},${radius} 0 0,1 ${xEnd},${height - 50}
      `;

      return {
        d: arcPath,
        x: xEnd,
        y: height - 50,
        r: rScale(d.agreements),
        conflict_country: d.conflict_country,
        distance: d.distance,
        agreements: d.agreements,
      };
    });
  }
</script>

<div id="buttons">
  <button on:click={() => updateMapData("Afghanistan")}> Afghanistan </button>
  <button on:click={() => updateMapData("Israel")}>Israel</button>
  <button on:click={() => updateMapData("Libya")}>Libya</button>
  <button on:click={() => updateMapData("Sudan")}>Sudan</button>
  <button on:click={() => updateMapData("Syria")}>Syria</button>
  <button on:click={() => updateMapData("Yemen")}>Yemen</button>
</div>
<div id="map" bind:this={mapContainer}></div>
<div id="chart" bind:clientWidth={width} bind:clientHeight={height}>
  <svg {width} {height}>
    <!-- {#if geo}
      {#each geo as g}
        <circle
          cx={x_scale(g.dist_mediation_conflict)}
          cy={y_scale(g.fatalities_best)}
          r="3"
          fill="white"
        >
        </circle>
      {/each}
    {/if} -->
    {#each arcs as arc}
      <path
        d={arc.d}
        fill="none"
        stroke="steelblue"
        stroke-width="1"
        stroke-opacity="0.3"
      >
        <title
          >{arc.conflict_country}: {arc.distance.toLocaleString()} meters</title
        >
      </path>
    {/each}
    {#each arcs as arc}
      <circle
        cx={arc.x}
        cy={arc.y}
        r={arc.r}
        fill="tomato"
        fill-opacity="0.5"
        stroke="black"
      >
        <title>
          {arc.conflict_country} — Agreements: {arc.agreements}
        </title>
      </circle>
    {/each}
  </svg>
</div>

<style>
  #map,
  #chart {
    width: 100%;
    height: 100vh;
  }

  #buttons {
    position: absolute;
    top: 0px;
    z-index: 999;
  }
</style>
