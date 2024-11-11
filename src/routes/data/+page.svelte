<script>
  import { onMount } from "svelte";

  let flightData = [];
  let errorMessage = "";

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
    } catch (error) {
      console.error("Error fetching flight data:", error);
      errorMessage = "Error fetching flight data.";
    }
  });
</script>

<style>
#flight-data {
  margin-top: 20px;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 15px;
}

.flight-item {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    cursor: pointer;
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  padding: 20px;
  transition: transform 0.2s;
}

.flight-item:hover {
  transform: translateY(-5px);
}

.flight-item div {
  margin-bottom: 10px;
}

.flight-item strong {
  color: #333;
}

</style>

<div id="flight-data">
  {#if errorMessage}
    <p>{errorMessage}</p>
  {:else if flightData.length === 0}
    <p>No flight data available.</p>
  {:else}
    {#each flightData as flight}
      <div class="flight-item">
        <div><strong>Flight Number:</strong> {flight.mainFlight || "N/A"}</div>
        <div><strong>Airline:</strong> {flight.prefixICAO || "N/A"}</div>
        <div><strong>Aircraft Type:</strong> {flight.aircraftType?.iataMain || "N/A"}</div>
        <div><strong>Schedule Date:</strong> {flight.scheduleDate || "N/A"}</div>
        <div><strong>Schedule Time:</strong> {flight.scheduleTime || "N/A"}</div>
        <div><strong>Destination :</strong>{flight.route?.destinations[0]}</div>
      </div>
    {/each}
  {/if}
</div>