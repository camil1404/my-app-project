<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";
    // import "d3-transition";
    import "../../../src/app.css";

    let width, height;
    let geoData = null;
    let airportData = null;
    let hoveredCountry = null;
    let countryAirports = [];
    let tooltipX = 0; // Tooltip X position
    let tooltipY = 0; // Tooltip Y position
    let flightData = [];
    let destinationCounts = {}; // To store flight counts for each country

    const schipholCoords = [4.764377, 52.308932]; // Schiphol coordinates

    let projection = d3.geoMercator().scale(100).translate([width / 2, height / 2]);
    let pathGenerator = d3.geoPath().projection(projection);

    let lineElements = []; // Store lines for later updates
    let currentTransform = d3.zoomIdentity; // Store the current zoom/pan transform
    //fetching data
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

            // Once flight data is loaded, call countFlightsToCountries
            countFlightsToCountries();
        } catch (error) {
            console.error("Error fetching flight data:", error);
            errorMessage = "Error fetching flight data.";
        }

        // Load the geographical data and airport data
        const geoResponse = await fetch("/custom.geo.json");
        geoData = await geoResponse.json();

        const airportResponse = await fetch("/airport.geojson");
        airportData = await airportResponse.json();

        // Once airportData is loaded, we can call countFlightsToCountries again
        if (airportData) {
            countFlightsToCountries();  // Re-run the counting after airport data is loaded
        }

        const svg = d3.select("svg");

        // Enable zoom and handle zoom/pan events
        svg.call(d3.zoom().scaleExtent([1, 8]).on("zoom", (event) => {
            currentTransform = event.transform;
            d3.select(".map-group").attr("transform", event.transform);
            updateLines(); // Update lines based on the current zoom/pan transform
        }));

        drawSimplePaths(svg); // Draw lines initially
    });

    $: if (width && height) {
        projection.translate([width / 2, height / 2]);
        projection.scale(width / 5); 
    }

    // Handle mouse over/out for displaying country-specific airports in a tooltip
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


    // Count flights based on destination IATA codes
    function countFlightsToCountries() {
        if (!airportData) return;  // Ensure airportData is loaded before proceeding

        flightData.forEach(flight => {
            const destinations = flight.route?.destinations || [];
            if (destinations.length > 0) {
                const destinationIATA = destinations[0]; // First destination IATA code
                airportData.features.forEach(airport => {
                    if (airport.properties.iata === destinationIATA) {
                        const country = airport.properties.country;
                        if (!destinationCounts[country]) {
                            destinationCounts[country] = 0;
                        }
                        destinationCounts[country]++;
                    }
                });
            }
        });
    }
    

    // Function to draw flight paths
function drawSimplePaths(svg) {
    if (!airportData || !flightData) return;
//punt waar de lijnen vandaan moeten komen 
    const schipholProjected = projection(schipholCoords);

    // Step 1: Aggregate flights by destination IATA
    const destinationCounts = {};
    flightData.forEach(flight => {
        const destinations = flight.route?.destinations || [];
        if (destinations.length > 0) {
            const destinationIATA = destinations[0];
            if (!destinationCounts[destinationIATA]) {
                destinationCounts[destinationIATA] = 0;
            }
            destinationCounts[destinationIATA]++;
        }
    });

    // Step 2: Define a scale for line width
    const lineWidthScale = d3.scaleLinear()
        .domain([1, d3.max(Object.values(destinationCounts))]) // From 1 flight to the max count
        .range([0.5, 5]); // Adjust range for line thickness as needed

    // Step 3: Draw lines
    Object.entries(destinationCounts).forEach(([destinationIATA, count]) => {
        const destinationAirport = airportData.features.find(airport => airport.properties.iata === destinationIATA);

        if (destinationAirport) {
            const destinationCoords = destinationAirport.geometry.coordinates;
            const destinationProjected = projection(destinationCoords);

            if (schipholProjected && destinationProjected) {
                const line = svg.append("line")
                    .attr("x1", schipholProjected[0])
                    .attr("y1", schipholProjected[1])
                    .attr("x2", schipholProjected[0]) // Start x2 and y2 at Schiphol
                    .attr("y2", schipholProjected[1]) // Start x2 and y2 at Schiphol
                    .attr("stroke", "orange")
                    .attr("stroke-width", lineWidthScale(count)) // Use the scaled width
                    .attr("opacity", 0.7);

                // Animate line drawing
                line.transition()
                    .duration(2000)
                    .ease(d3.easeLinear)
                    .attr("x2", destinationProjected[0])
                    .attr("y2", destinationProjected[1]);

                // Store line info for updates
                lineElements.push({
                    line,
                    schiphol: schipholProjected,
                    destination: destinationProjected
                });
            }
        }
    });
}

    // Update line positions based on the current zoom and pan
function updateLines() {
    lineElements.forEach(({ line, schiphol, destination }) => {
        const transformedSchiphol = currentTransform.apply([schiphol[0], schiphol[1]]);
        const transformedDestination = currentTransform.apply([destination[0], destination[1]]);


        line.attr("x1", transformedSchiphol[0])
            .attr("y1", transformedSchiphol[1])
            .attr("x2", transformedDestination[0])
            .attr("y2", transformedDestination[1]);
    });
}
   
</script>

<main bind:clientWidth={width} bind:clientHeight={height}>
    {#if geoData}
        <svg width={width} height={height}>
            <rect width="100%" height="100%" fill="#121212." />

            <g class="map-group">
                {#each geoData.features as feature}
                    <path
                        d={pathGenerator(feature)}
                        fill={feature.properties.name === "Netherlands" ? "#03DAC5" : "#1F1B24"}
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
                            r="1"
                            fill="#bb86fc"
                            stroke="black"
                             stroke-width="0.3"
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
            <p class="airport-count">Total Flights: {destinationCounts[hoveredCountry] || 0}</p>
        </div>
    {/if}
    <div class="flight-count">
    Total Flights: {flightData.length}
</div>

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
    .flight-count {
    position: absolute;
    top: 10px;
    right: 10px;
    background-color: rgba(0, 0, 0, 0.7); /* Semi-transparent background */
    color: white;
    padding: 8px 12px;
    border-radius: 8px;
    font-size: 14px;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.2); /* Optional shadow for better visibility */
}
</style>