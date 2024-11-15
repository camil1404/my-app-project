<script>
    import "../../src/app.css"
    // You can add any necessary homepage logic here
</script>

<main>
    <h1>Welcome to my project</h1>
    <p>Lets go exploring</p>
    
    <!-- Link to navigate to the map page -->
    <a href="/map">Go to Map</a>
    <a href="/data">Raw Data</a>
</main>

<style>
    main {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        height: 100vh;
        text-align: center;
    }

    a {
        margin-top: 20px;
        padding: 10px 20px;
        background-color: #007acc;
        color: white;
        text-decoration: none;
        border-radius: 4px;
    }

    a:hover {
        background-color: #005ea2;
    }
    h1{
        font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;

    }
    p{
        font-family: Arial, Helvetica, sans-serif;
    }
    a{
        font-family: Arial, Helvetica, sans-serif;
    }
</style>