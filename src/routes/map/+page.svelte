<script>
    import { onMount } from "svelte";
    import * as d3 from "d3";

    let width, height;
    let geoData = null;
    let hoveredCountry = null;  // To store the country being hovered

    onMount(async () => {
        // Fetch the GeoJSON data
        const response = await fetch("/custom.geo.json");
        geoData = await response.json();
    });

    // Define a D3 projection and path generator
    let projection = d3.geoMercator().scale(100).translate([width / 2, height / 2]);
    let pathGenerator = d3.geoPath().projection(projection);

    // Update projection when dimensions change
    $: if (width && height) {
        projection.translate([width / 2, height / 2]);
        projection.scale(width / 5); // Adjust scale as needed
    }

    // Handle hover to show country name
    function handleMouseOver(event, feature) {
        hoveredCountry = feature.properties.name;  // Assuming the country name is stored in `properties.name`
    }

    function handleMouseOut() {
        hoveredCountry = null;
    }
</script>

<main bind:clientWidth={width} bind:clientHeight={height}>
    {#if geoData}
        <svg width={width} height={height}>
            <!-- oceaan blauw maken -->
                        <rect width="100%" height="100%" fill="lightblue" />
            {#each geoData.features as feature}
                <path
                    d={pathGenerator(feature)}
                    fill="lightgray"
                    stroke="black"
                    on:mouseover={(e) => handleMouseOver(e, feature)}
                    on:mouseout={handleMouseOut}
                    style="transition: fill 0.3s ease;"
                    class="country-path"
                />
            {/each}
            
            <!-- Tooltip display -->
            {#if hoveredCountry}
                <text x={width - 150} y={50} class="tooltip">
                    {hoveredCountry}
                </text>
            {/if}
        </svg>
    {/if}
</main>

<style>
    main {
        width: 100vw;
        height: 100vh;
        overflow: hidden;
    }

    .country-path:hover {
        fill: red; 
    }

    .tooltip {
        font-size: 20px;
        font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        font-weight: bold;
        fill: black;
        text-anchor: end;
        pointer-events: none;
    }
</style>