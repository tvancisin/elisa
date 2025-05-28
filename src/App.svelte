<script>
  import * as d3 from "d3";
  import {
    getCSV,
    getGeoMultiple,
    createGeoJSONCircle,
    construct_line,
    construct_points,
  } from "./utils.js";
  import { onMount } from "svelte";
  import mapboxgl from "mapbox-gl";
  import "mapbox-gl/dist/mapbox-gl.css";

  let width = 400;
  let height = 400;
  let isOverlayVisible = true;
  let enabled = false;

  // remove initial div
  function removeOverlay() {
    isOverlayVisible = false;
    enabled = true;
  }

  $: x_scale = d3
    .scaleLinear()
    .domain([0, 17000000])
    .range([80, width - 20]);

  $: y_scale = d3
    .scaleLinear()
    .domain([0, 25000])
    .range([height - 50, 10]);

  let blah = createGeoJSONCircle([67.709953, 33.93911], 1500);
  let krava = createGeoJSONCircle([67.709953, 33.93911], 5000);

  let mapContainer;
  mapboxgl.accessToken =
    "pk.eyJ1Ijoic2FzaGFnYXJpYmFsZHkiLCJhIjoiY2xyajRlczBlMDhqMTJpcXF3dHJhdTVsNyJ9.P_6mX_qbcbxLDS1o_SxpFg";

  // load data
  let geo;
  let geo_data;
  let map;
  let path = ["./elisa_map.csv"];
  let conflict_groups;
  let cleaned_geo = [];
  getCSV(path).then((data) => {
    geo = data[0];
    cleaned_geo = geo
      .map((d) => {
        // Reject rows where any relevant field is "NA"
        if (
          d.dist_mediation_conflict === "NA" ||
          d.fatalities_best === "NA" ||
          d.latitude_con === "NA" ||
          d.longitude_con === "NA" ||
          d.iso3c === "NA"
        ) {
          return null;
        }

        const distance = +d.dist_mediation_conflict;
        const deaths = +d.fatalities_best;

        if (!isFinite(distance) || distance <= 0) {
          return null;
        }

        return {
          ...d,
          distance,
          deaths: isFinite(deaths) ? deaths : 0,
        };
      })
      .filter(Boolean);

    conflict_groups = d3.groups(geo, (d) => d.conflict_country);
  });

  const json_path = ["geojson.json"];
  getGeoMultiple(json_path).then((geo) => {
    geo_data = geo[0];
  });

  onMount(() => {
    // Load GEOJSON

    map = new mapboxgl.Map({
      container: mapContainer,
      style: "mapbox://styles/sashagaribaldy/cm6ktkpxn00m901s2fo0bh1e4",
      projection: "naturalEarth",
      center: [10, 10],
      zoom: 1.2,
    });

    map.on("load", () => {
      if (conflict_groups[0][1] && geo_data) {
        const lines = construct_line(conflict_groups[0][1]);
        const points = construct_points(conflict_groups[0][1]);
        console.log(geo_data);

        map.addSource("countries", {
          type: "geojson",
          data: geo_data,
          generateId: true, // Ensures all features have unique IDs
        });

        // Find the first symbol layer (typically a label layer)
        const labelLayerId = map
          .getStyle()
          .layers.find(
            (layer) => layer.type === "symbol" && layer.layout?.["text-field"],
          )?.id;

        // Add fill layer with conditional color
        map.addLayer(
          {
            id: "countries_fill",
            type: "fill",
            source: "countries",
            paint: {
              "fill-color": "steelblue",
              "fill-opacity": 0.8,
            },
            filter: ["in", "ADMIN", "Afghanistan"],
          },
          labelLayerId,
        );

        map.addSource("polygon", {
          type: "geojson",
          data: blah,
        });

        map.addLayer({
          id: "polygon",
          type: "fill",
          source: "polygon",
          paint: {
            "fill-color": "steelblue",
            "fill-opacity": 0.2,
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
            "fill-color": "steelblue",
            "fill-opacity": 0.2,
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
            "line-opacity": 0.5,
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
      1500,
    );
    let newKrava = createGeoJSONCircle(
      [+update_data[0].longitude_con, +update_data[0].latitude_con],
      5000,
    );

    const lineSource = map.getSource("curved-line");
    const pointSource = map.getSource("points");
    const polygonSource = map.getSource("polygon");
    const polygon3Source = map.getSource("polygon3");

    if (lineSource && pointSource && polygonSource) {
      map.setFilter("countries_fill", ["==", "ADMIN", conflictData]);
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
  const startX = 75;
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
      .domain([0, 17000000])
      .range([80, width - 20]);

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

  let x_axis_grp;
  let x_axis_grp1;
  let y_axis_grp;
  $: if (x_axis_grp && y_axis_grp) {
    let xAxis = d3.axisBottom(x_scale);
    d3.select(x_axis_grp).call(xAxis);
    let xAxis1 = d3.axisBottom(x_scale);
    d3.select(x_axis_grp1).call(xAxis1);
    let yAxis = d3.axisLeft(y_scale);
    d3.select(y_axis_grp).call(yAxis);
  }
</script>

<main>
  <h1>
    Bridging the Distance: New Insights on Geography in Conflict Mediation
  </h1>
  <div class="blog_text">
    <p>
      Until recently, we haven’t been able to answer basic questions about the
      logistics of conflict mediation and how they shape outcomes, largely due
      to a lack of systematic data. As a result, factors like where mediation
      takes place have remained understudied. With new data now available,
      researchers are beginning to examine these overlooked dimensions,
      including recent work on the role of location in the mediation process.
    </p>
    <p>
      A recent study, “The Geography of Conflict Mediation: Proximity and
      Success in Armed Conflict Resolutions,” looks at how the distance between
      conflict zones and mediation sites affects the chances of reaching a
      peaceful outcome. Drawing on newly available geospatial data, the study
      finds that location plays a meaningful role in shaping mediation success.
    </p>
    <h2>The Paradox of Distance: Agreements vs. De-escalation</h2>
    <p>The research highlights a fascinating duality in mediation success:</p>

    <ol>
      <li>
        Distant Mediation Fosters Agreements: The study finds that the further a
        mediation event is from a conflict area, the higher the likelihood of a
        formal peace agreement being signed. This suggests that a greater
        geographical distance can confer perceived neutrality, reduce immediate
        local political pressures, and allow parties to negotiate without the
        intense emotional and security concerns that often plague discussions
        held closer to the conflict. The data indicates that for every
        additional “far away” mediation event, the odds of a peace agreement
        being signed increase by approximately 14.2%. This underscores the
        strategic value of neutral ground when aiming for formal settlements.
      </li>
    </ol>
  </div>
  <div id="buttons">
    <button disabled={!enabled} on:click={() => updateMapData("Afghanistan")}>
      Afghanistan
    </button>
    <button disabled={!enabled} on:click={() => updateMapData("Israel")}
      >Israel</button
    >
    <button disabled={!enabled} on:click={() => updateMapData("Libya")}
      >Libya</button
    >
    <button disabled={!enabled} on:click={() => updateMapData("Sudan")}
      >Sudan</button
    >
    <button disabled={!enabled} on:click={() => updateMapData("Syria")}
      >Syria</button
    >
    <button disabled={!enabled} on:click={() => updateMapData("Yemen")}
      >Yemen</button
    >
  </div>
  <div class="map-wrapper">
    {#if isOverlayVisible}
      <div class="overlay">
        <button class="remove-overlay" on:click={removeOverlay}>
          Click to Explore
        </button>
      </div>
    {/if}
    <div id="map" bind:this={mapContainer}></div>
  </div>

  <div class="blog_text">
    <ol start="2">
      <li>
        Closer Proximity Reduces Violence: Conversely, the research demonstrates
        that closer proximity between a mediation event and the conflict zone is
        directly linked to a greater reduction in violent events and fatalities.
        This is because localized mediation can foster trust, directly engage
        local actors, address specific grievances, and enable quicker responses
        to escalating violence on the ground. Such immediacy, driven by close
        proximity, acts as a powerful catalyst for de-escalation. Specifically,
        for every additional “close by” mediation event, the expected number of
        battle deaths in the following month decreases by approximately 4.7%.
      </li>
    </ol>
  </div>
  <div id="chart1">
    <svg {width} {height}>
      <g bind:this={x_axis_grp} transform={`translate(0, ${height - 40})`} />
      <g bind:this={y_axis_grp} transform={`translate(75, 0)`} />
      <text x={width / 2} y={height} fill="white" font-size="14px"
        >Distance from Conflict</text
      >
      <text
        x={20}
        y={height / 2}
        fill="white"
        font-size="14px"
        transform={`rotate(-90, 20, ${height / 2})`}
      >
        Number of Fatalities
      </text>
      {#if cleaned_geo}
        {#each cleaned_geo as g}
          <circle
            cx={x_scale(g.distance)}
            cy={y_scale(g.deaths)}
            r="3"
            fill="steelblue"
            fill-opacity="0.4"
            stroke="none"
          >
          </circle>
        {/each}
      {/if}
    </svg>
  </div>
  <div class="blog_text">
    <h2>Implications for Peacemakers</h2>
    <p>
      This groundbreaking work signals a vital redefinition of “conflict
      resolution” for peacemakers. It suggests that success isn’t a singular
      metric. Formal agreements may indeed require the neutral, removed spaces
      offered by distant mediation, but the necessary work of reducing violence
      on the ground often happens closer to the epicenter of conflict through
      local engagement and direct intervention.
    </p>
    <p>
      For policymakers and peace practitioners, these findings prompt a deeper
      reflection on priorities and strategies:
    </p>
    <ul>
      <li>
        Tailor the Approach to the Goal: If the primary objective is a formal
        peace agreement, seeking geographically distant and neutral mediation
        venues appears to be a more effective strategy.
      </li>
      <li>
        Invest in Localized Efforts: For immediate de-escalation and mitigation
        of fatalities, greater investment in and support for localized mediation
        efforts are essential. These efforts leverage intimate knowledge of the
        conflict context and facilitate trust-building among directly affected
        parties.
      </li>
      <li>
        A Holistic Strategy: True conflict resolution requires a holistic
        strategy that values both formal peace agreements and on-the-ground
        violence reduction. This may necessitate a multi-layered approach,
        combining distant high-level negotiations with embedded, proximate
        mediation initiatives.
      </li>
    </ul>
  </div>
  <div id="chart" bind:clientWidth={width} bind:clientHeight={height}>
    <svg {width} {height}>
      <g bind:this={x_axis_grp1} transform={`translate(0, ${height - 40})`} />
      <text x={width / 2} y={height} fill="white" font-size="14px"
        >Distance from Conflict</text
      >
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
</main>

<style>
  .map-wrapper {
    position: relative;
    width: 100%;
    height: 80vh; /* or whatever height you need */
  }

  .overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    width: 80%;
    height: 80vh;
    margin: 0px auto;
    background-color: rgba(255, 255, 255, 0);
    display: flex;
    border-radius: 5px;
    align-items: center;
    justify-content: center;
    z-index: 10;
  }

  .remove-overlay {
    color: black;
    width: 150px;
    height: 150px;
    border-radius: 50%;
    border: none;
    background-color: rgba(255, 255, 255, 0.8);
    font-family: "Montserrat", sans-serif;
    font-size: 20px;
    font-optical-sizing: auto;
    font-weight: 400;
    font-style: normal;
  }

  .remove-overlay:hover {
    cursor: pointer;
    background-color: red;
    color: white;
  }

  button {
    background-color: #04aa6d; /* Green */
    border: none;
    color: black;
    padding: 10px 30px;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    border-radius: 3px;
    cursor: pointer;
  }

  button:hover {
    color: white;
  }

  button:disabled {
    background-color: #ccc;
    cursor: not-allowed;
  }

  main {
    display: flex;
    flex-direction: column;
    padding: 20px;
    max-width: 100%;
    box-sizing: border-box;
    color: rgb(246, 246, 234);
  }

  h1 {
    width: 80%;
    margin: 50px auto;
    text-align: center;
  }

  li {
    padding: 5px;
  }

  .blog_text,
  #buttons {
    width: 65%;
    margin: 50px auto;
  }

  @media (max-width: 768px) {
    .blog_text,
    #buttons {
      width: 95%;
    }
  }

  #buttons {
    text-align: center;
    margin: 10px auto;
  }

  #map,
  #chart,
  #chart1 {
    width: 80%;
    height: 80vh;
    margin: auto;
  }
</style>
