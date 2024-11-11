<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";

    let width, height;
    let geoData = null;
    let airportData = null;
    let hoveredCountry = null;
    let countryAirports = [];
    let tooltipX = 0; // Tooltip X position
    let tooltipY = 0; // Tooltip Y position
    let flightData = [];
    let errorMessage = "";
    let destinationCounts = {}; // To store flight counts for each country

    onMount(async () => {
        console.log("Fetching flight data...");

        try {
            const response = await fetch("http://localhost:3000/api/flights");

            console.log("Response status:", response.status);

            if (!response.ok) {
                throw new Error(`HTTP error! Status: ${response.status}`);
            }

            const data = await response.json();
            console.log("Flight data received:", data);

            flightData = Array.isArray(data) && data.length > 0 ? data : [];

            // Count flights based on destination IATA codes
            countFlightsToCountries();
        } catch (error) {
            console.error("Error fetching flight data:", error);
            errorMessage = "Error fetching flight data.";
        }
    });

    // Count flights based on destination IATA codes
    function countFlightsToCountries() {
        flightData.forEach(flight => {
            const destinations = flight.route?.destinations || [];
            if (destinations.length > 0) {
                const destinationIATA = destinations[0]; // First destination IATA code
                // Find matching airport in airportData
                airportData.features.forEach(airport => {
                    if (airport.properties.iata === destinationIATA) {
                        const country = airport.properties.country;
                        // Increment the count for the corresponding country
                        if (!destinationCounts[country]) {
                            destinationCounts[country] = 0;
                        }
                        destinationCounts[country]++;
                    }
                });
            }
        });
    }

    onMount(async () => {
        const geoResponse = await fetch("/custom.geo.json");
        geoData = await geoResponse.json();

        const airportResponse = await fetch("/airport.geojson");
        airportData = await airportResponse.json();

        const svg = d3.select("svg");
        svg.call(d3.zoom().scaleExtent([1, 8]).on("zoom", (event) => {
            d3.select(".map-group").attr("transform", event.transform);
        }));
    });

    let projection = d3.geoMercator().scale(100).translate([width / 2, height / 2]);
    let pathGenerator = d3.geoPath().projection(projection);

    $: if (width && height) {
        projection.translate([width / 2, height / 2]);
        projection.scale(width / 5); 
    }

    function handleMouseOver(event, feature) {
        hoveredCountry = feature.properties.name;
        tooltipX = event.pageX + 10;
        tooltipY = event.pageY + 10;

        countryAirports = airportData.features.filter(airport =>
            airport.properties.country === hoveredCountry
        );
    }

    function handleMouseOut() {
        hoveredCountry = null;
        countryAirports = [];
    }
</script>

<main bind:clientWidth={width} bind:clientHeight={height}>
    {#if geoData}
        <svg width={width} height={height}>
            <rect width="100%" height="100%" fill="lightblue" />

            <g class="map-group">
                {#each geoData.features as feature}
                    <path
                        d={pathGenerator(feature)}
                        fill={feature.properties.name === "Netherlands" ? "blue" : "lightgray"}
                        stroke="black"
                        on:mouseover={(e) => handleMouseOver(e, feature)}
                        on:mousemove={(e) => { tooltipX = e.pageX + 10; tooltipY = e.pageY + 10; }}
                        on:mouseout={handleMouseOut}
                        class="country-path"
                    />
                {/each}

                {#if airportData}
                    {#each airportData.features as airport}
                        <circle
                            cx={projection(airport.geometry.coordinates)[0]}
                            cy={projection(airport.geometry.coordinates)[1]}
                            r="3"
                            fill="red"
                            stroke="black"
                            on:mouseover={() => handleMouseOver({ pageX: projection(airport.geometry.coordinates)[0], pageY: projection(airport.geometry.coordinates)[1] }, airport)}
                            on:mouseout={handleMouseOut}
                        />
                    {/each}
                {/if}
            </g>
        </svg>
    {/if}

    <!-- Tooltip div positioned absolutely outside of the SVG -->
    {#if hoveredCountry}
        <div
            class="tooltip-box"
            style="top: {tooltipY}px; left: {tooltipX}px;"
        >
            <h3 class="tooltip-title">{hoveredCountry}</h3>
            
            <!-- Display the total count of flights to this country -->
            <p class="airport-count">Total Flights: {destinationCounts[hoveredCountry] || 0}</p>
            
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
        fill: orange;
    }

    .tooltip-box {
        position: absolute;
        padding: 10px;
        background-color: rgba(255, 255, 255, 0.9);
        border-radius: 8px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
        font-family: Arial, sans-serif;
        color: #333;
        font-size: 14px;
        text-align: left;
        max-width: 300px;
        max-height: 800px; /* Max height for long lists */
        overflow-y: auto;
        z-index: 1000;
        pointer-events: none;
    }

    .tooltip-title {
        font-size: 16px;
        font-weight: bold;
        margin-bottom: 5px;
    }

    .tooltip-content {
        font-size: 14px;
        line-height: 1.6;
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

    .airport-count {
        font-size: 14px;
        font-weight: bold;
        color: #555;
        margin-bottom: 8px;
    }
</style>