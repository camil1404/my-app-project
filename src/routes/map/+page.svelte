<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";

    let width, height;
    let geoData = null;
    let airportData = null;
    let hoveredCountry = null;  
    let countryAirports = [];    

    onMount(async () => {
        const geoResponse = await fetch("/custom.geo.json");
        geoData = await geoResponse.json();

        const airportResponse = await fetch("/airport.geojson");
        airportData = await airportResponse.json();

        // Initialize zoom behavior
        const svg = d3.select("svg");
        svg.call(d3.zoom().scaleExtent([1, 8]).on("zoom", (event) => {
            d3.select(".map-group").attr("transform", event.transform);
        }));
    });

    // D3 projection and path generator for countries
    let projection = d3.geoMercator().scale(100).translate([width / 2, height / 2]);
    let pathGenerator = d3.geoPath().projection(projection);

    // Update projection when dimensions change
    $: if (width && height) {
        projection.translate([width / 2, height / 2]);
        projection.scale(width / 5); 
    }

    // Handle hover to show country name and airports
    function handleMouseOver(event, feature) {
        hoveredCountry = feature.properties.name;

        // Filter the airports within this country
        countryAirports = airportData.features.filter(airport =>
            airport.properties.country === hoveredCountry
        );
    }

    function handleAirportMouseOver(airport) {
        hoveredCountry = airport.properties.name; // Show the airport's name in the tooltip
        countryAirports = [];
    }

    function handleMouseOut() {
        hoveredCountry = null;
        countryAirports = [];
    }
</script>

<main bind:clientWidth={width} bind:clientHeight={height}>
    {#if geoData}
        <svg width={width} height={height}>
            <!-- Ocean background -->
            <rect width="100%" height="100%" fill="lightblue" />

            <!-- Zoomable map group -->
            <g class="map-group">
                <!-- Render country paths -->
                {#each geoData.features as feature}
                    <path
                        d={pathGenerator(feature)}
                        fill={feature.properties.name === "Netherlands" ? "blue" : "lightgray"}
                        stroke="black"
                        on:mouseover={(e) => handleMouseOver(e, feature)}
                        on:mouseout={handleMouseOut}
                        style="transition: fill 0.3s ease;"
                        class="country-path"
                    />
                {/each}

                <!-- Render airport points -->
                {#if airportData}
                    {#each airportData.features as airport}
                        <circle
                            cx={projection(airport.geometry.coordinates)[0]}
                            cy={projection(airport.geometry.coordinates)[1]}
                            r="3"
                            fill="red"
                            stroke="black"
                            on:mouseover={() => handleAirportMouseOver(airport)}
                            on:mouseout={() => hoveredCountry = null}
                        />
                    {/each}
                {/if}
            </g>

            <!-- Tooltip display in top-left box -->
            {#if hoveredCountry}
                <foreignObject x="10" y="10" width="300" height={countryAirports.length > 5 ? 300 : 150}>
                    <div class="tooltip-box">
                        <h3 class="tooltip-title">{hoveredCountry}</h3>
                        {#if countryAirports.length > 0}
                            <div class="tooltip-content">
                                <p>Airports:</p>
                                <ul>
                                    {#each countryAirports as airport}
                                        <li>{airport.properties.iata}</li>
                                    {/each}
                                </ul>
                            </div>
                        {/if}
                    </div>
                </foreignObject>
            {/if}
        </svg>
    {/if}
</main>

<style>
main {
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    position: relative;
}

.country-path:hover {
    fill: orange; /* Color change on hover */
}

.tooltip-box {
    padding: 10px;
    background-color: rgba(255, 255, 255, 0.9);
    border-radius: 8px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
    font-family: Arial, sans-serif;
    color: #333;
    font-size: 14px;
    text-align: left;
    transition: transform 0.3s ease-in-out;
    max-width: 300px;
}

.tooltip-title {
    font-size: 16px;
    font-weight: bold;
    margin-bottom: 5px;
}

.tooltip-content {
    font-size: 14px;
    line-height: 1.6;
    overflow-y: auto;
    max-height: 400px; /* Higher max height for countries with many airports */
    padding-right: 10px; /* Add padding to avoid overlap with scrollbar */
}

.tooltip-content ul {
    padding-left: 20px;
    list-style-type: disc;
    margin: 0;
}

.tooltip-content li {
    font-size: 13px;
    margin-bottom: 2px;
}
</style>