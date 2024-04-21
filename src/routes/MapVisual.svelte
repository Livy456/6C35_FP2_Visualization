<script>
import * as d3 from "d3";
    import {
        computePosition,
        autoPlacement,
        offset
    } from '@floating-ui/dom';
    export let binned_data = [];
    export let xScale = d3.scaleBand();
    export let metric = "Family Type";
    export let data = [];
    export let binned_info = "";
    export let bin_type = [];
    // export let yScale = d3.scaleLinear();
</script>

<style>

</style>
