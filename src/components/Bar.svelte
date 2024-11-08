<script>
    import * as d3 from "d3";
    import { onMount } from 'svelte';
 
    onMount(() => {
        d3.select('p')
        .text('Hello world!')
    });
</script>
 
<p></p>
 
<style>
    p {
        font-size: 2em;
        color: blue;
    }
</style> 