<script>
    import * as d3 from "d3";
    import { onMount } from "svelte";
    import {
        computePosition,
        autoPlacement,
        offset
    } from '@floating-ui/dom';

    let data = [];
    let width = 800, height = 400; // changed the height of the graph from 600 to 450
    let yScale = d3.scaleLinear();
    let xScale = d3.scaleBand();
    let rScale = d3.scaleSqrt();
    let xAxis, yAxis;
    let yAxisGridlines;
    let hoveredIndex = -1;
    $: hoveredEviction = data[hoveredIndex]?? {};
    let tooltipPosition = {x:0, y:0};
    let evictionTooltip;
    let svg;
    let brushedSelection;
    let selectedEvictions;
    let hasSelection;
    let selectedLines;
    let boxWidth = 100;
    let family_bins = ["mixed", "non-family", "family"];
    let data_mixed;
    let data_family;
    let data_non_family;
    let box_plot_stats = {mixed: {}, non_family: {}, family: {}}
    let metric_to_graph = "Type of Household";

    let languageBreakdown;
    let languageBreakdownArray;
    
    const format = d3.format(".1~%");
        
    // defining axes
    let margin = {top: 10, right: 10, bottom: 30, left:20};
    let usableArea = {
        top: margin.top,
        bottom: height - margin.bottom,
        left: margin.left,
        right: width - margin.right
    };
    usableArea.width = usableArea.right - usableArea.left;
    usableArea.height = usableArea.bottom - usableArea.top;

    // creating axes on the page
    $: {
        d3.select(xAxis).call(d3.axisBottom(xScale));
        d3.select(yAxis).call(d3.axisLeft(yScale));
        d3.select(yAxisGridlines).call(d3.axisLeft(yScale).tickSize(-usableArea.width));
    }

    onMount(async() => {
        data = await d3.csv("binned_data.csv", row=> ({
            ...row,
            mhi: Number(row.mhi),
            eviction_rate: Number(row.eviction_rate),
            family_bins: String(row.family_bins)
        }));
    });
    $: data = data.filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1).filter((d) => d.mhi > 0);

    // family type, race, elderly or not
    // [y_min - (y_min*0.05), y_max + (y_max * 0.05)]
    $: yScale = yScale.domain([0, 0.2]).range([usableArea.bottom, usableArea.top]); // might need to switch values currently domain = [0, height], range = [0, 24] 
    $: xScale = xScale.domain(["mixed", "non-family", "family"]).range( [usableArea.left, usableArea.right] ).padding(0.2);    
    $: {
        d3.select(svg).call(d3.brush().on("start brush end", brushed));
        d3.select(svg).selectAll(".dots, .overlay ~ *").raise();
    }

    $: selectedEvictions = brushedSelection ? data.filter(isDataSelected) : [];    
    $: hasSelection = brushedSelection && selectedEvictions.length > 0;
    $: selectedLines = (hasSelection ? selectedEviction: data).flatMap(d => d.mhi);
    $: data_mixed = data.filter((d) => d.family_bins === "mixed");
    $: data_family = data.filter((d) => d.family_bins === "family");
    $: data_non_family = data.filter((d) => d.family_bins === "non-family");
    $: box_plot_stats.family = calculate_box_plot(data_family);
    $: box_plot_stats.non_family = calculate_box_plot(data_non_family);
    $: box_plot_stats.mixed = calculate_box_plot(data_mixed);

    async function dotInteraction (index, evt){
        let hoveredDot = evt.target;
        
        if (evt.type === "mouseenter" || evt.type === "focus")
        {
            hoveredIndex = index;
            tooltipPosition = await computePosition(hoveredDot, evictionTooltip, {
                strategy: "fixed",
                middleware: [
                    offset(20),
                    autoPlacement()
                ],
            });
        }

        else if (evt.type === "mouseleave" || evt.type === "blur")
        {
            hoveredIndex = -1;
        }
    }

    function calculate_box_plot(binned_data)
    {   
        let eviction_rate_array = [];

        for (let d of binned_data)
        {
            eviction_rate_array.push(d.eviction_rate);
        }

        let quantile1 = d3.quantile(eviction_rate_array, 0.25);
        let quantile2 = d3.quantile(eviction_rate_array, 0.5);
        let quantile3 = d3.quantile(eviction_rate_array, 0.75);
        let innerQuantileRange = quantile3 - quantile1;
        let minimum = quantile1 - .5*innerQuantileRange; // MIGHT NEED TO RECALCULATE THIS, ORIGINALLY -1.5*innerQuartileRange
        let maximum = quantile3 + 1.5*innerQuantileRange;
        let summary_stats = {q1: quantile1, q2: quantile2, q3:quantile3, innerQuantile:innerQuantileRange, min: minimum, max:maximum }
        
        return summary_stats
    } 

    function brushed (evt)
    {
        brushedSelection = evt.selection;
    }

    function isDataSelected(dot)
    {
        if(!brushedSelection)
        {
            return false;
        }

        // get points of brushed selection box
        let top_left = {x: brushedSelection[0][0], y: brushedSelection[0][1]};
        let bottom_right = {x: brushedSelection[1][0], y: brushedSelection[1][1]};

        // gets coordinate of commit data point
        let data_x_coord = xScale(dot.family_bins);
        let data_y_coord = yScale(dot.eviction_rate)

        // checks if commit data point is within selected brush region
        if(data_x_coord >= top_left.x && data_x_coord <= bottom_right.x && 
        data_y_coord >= top_left.y && data_y_coord <= bottom_right.y)
        {
            // console.log("I'm a brushed dot!!");
            return true;
        }

        // console.log("I haven't been brushed!!!");

        return false;
    }
    
</script>

<style>

@keyframes marching-ants {
        to{
            stroke-dashoffset: -8;
        }
    }

    svg :global(.selection){
        /* overflow: visible;
        margin:50px; */
        fill-opacity: 10%;
        stroke: black;
        stroke-opacity: 70%;
        stroke-dasharray: 5 3;
        animation: marching-ants 2s linear infinite;
    }

    svg{
        margin:5 0px;
    }

    dl.info
    {
        display: grid;
        grid-template-columns: auto 1fr;
        background-color: oklch(100% 0% 0 / 80%);
        box-shadow: 5px 5px 5px lightslategrey;
        border-radius: 5%;
        backdrop-filter: blur(10px);
        padding:10px;

        /* transition-duration: 50ms; */
        transition-property: opacity, visibility;

        &[hidden]:not(:hover, :focus-within){
            opacity: 0;
            visibility: hidden;
        }
    }

    .tooltip{
        position: fixed;
        margin:15px;
        /* top: 1em; */
    }


    h2.meta{
        font-size: 50px;
        font-family: 'Segoe UI';
    }

    dl.stats{
        display: grid;
        grid-template-columns: auto;
        grid-template-rows: auto;
        font-family: 'Segoe UI';

        dt{
            grid-row: 1;
            font-size: 20px;
            color: grey;
        }

        dd{
            grid-row: 2;
            font-size: 35px;
            text-align: left;
            padding: 0px;
            margin: 0px;
        }
    }

    dl.date_summary{
        display: grid;
        grid-template-columns: auto;
        grid-template-rows: auto;
        font-family: 'Segoe UI';
    }

    .gridlines{
        stroke-opacity: 0.2;
    }
    
    .dots{

        circle.selected{
            fill: rgb(228, 24, 24)
        }

        circle{
            transition: 200ms;
            transform-origin: center;
            transform-box: fill-box;
            &:hover
            {
                transform: scale(1.5);
                
            }
        }
    }

    .meta_legend{
        display: grid;


        dd{
            grid-row: 1;
            margin: 0px;
            font-size: 15px;
            text-transform: uppercase;
            color: rgb(99, 99, 99);
            font-family: 'Segoe UI';
        }

        dt{
            grid-row: 2;
        }
    }
    
</style>

<h2 class="meta">Summary</h2>
<div class="metric_selection">
    <input type="button" value="Race">
    <input type="button" value="Elderly">
    <input type="button" value="Household Type">
</div>




<h2 style="margin-top: 3rem">Number of Evictions vs {metric_to_graph}</h2>
    
<dl id="eviction-tooltip" class="info tooltip" 
    hidden={hoveredIndex === -1}
    bind:this={evictionTooltip}
    style="top:{tooltipPosition.y}px; left:{tooltipPosition.x}px">
    <dt>Eviction Number</dt>
    <dd> { format(hoveredEviction.eviction_rate, ".3f") } </dd>

    <dt>Median Household Income</dt>
    <dd>{ hoveredEviction.mhi }</dd>

    <dt>Race</dt>
    <dd> { hoveredEviction.majority_race } </dd>

    <dt>House Type</dt>
    <dd> { hoveredEviction.family_bins } </dd>

    <dt>Elderly in House</dt>
    <dd> { hoveredEviction.elder_bins } </dd>


</dl>
<svg viewBox="0 0 {width} {height}" bind:this={svg}>
    <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
    <g class="gridlines" transform="translate({usableArea.left}, 0)" bind:this={yAxisGridlines} />
    <g transform="translate({usableArea.top})" bind:this={yAxis}/>
    
    <g class="dots">
    <!-- MAKE SURE TO SHIFT ALL X VALUES BY + xScale.bandwidth() / 2 -->
    {#each family_bins as bin_type}
        {#if bin_type === "family"}
        <line 
            x1={ xScale(bin_type) + xScale.bandwidth() / 2 }
            x2={ xScale(bin_type) + xScale.bandwidth() / 2 }
            y1={ yScale(box_plot_stats.family.min) }
            y2={ yScale(box_plot_stats.family.max) }
            stroke="black"
            width=40
        />
        <line
            x1={ xScale(bin_type) - boxWidth/2 + xScale.bandwidth() / 2 }
            x2={ xScale(bin_type) + boxWidth/2 + xScale.bandwidth() / 2 }
            y1={ yScale(box_plot_stats.family.q2) }
            y2={ yScale(box_plot_stats.family.q2) }
            stroke="black"
            width=80
        />

        <rect
            x={ xScale(bin_type) - boxWidth/2 + xScale.bandwidth() / 2}
            y={yScale(box_plot_stats.family.q3)} 
            width={boxWidth}
            height={yScale(box_plot_stats.family.q1) - yScale(box_plot_stats.family.q3)}
            stroke="black"
            fill="blue"
            fill-opacity=0.5
        />
        {/if}

        {#if bin_type === "non-family"}
        <line 
            x1={ xScale(bin_type) + xScale.bandwidth() / 2 }
            x2={ xScale(bin_type) + xScale.bandwidth() / 2 }
            y1={ yScale(box_plot_stats.non_family.min) }
            y2={ yScale(box_plot_stats.non_family.max) }
            stroke="black"
            width=40
        />
        <line
            x1={ xScale(bin_type) - boxWidth/2 + xScale.bandwidth() / 2 }
            x2={ xScale(bin_type) + boxWidth/2 + xScale.bandwidth() / 2 }
            y1={ yScale(box_plot_stats.non_family.q2) }
            y2={ yScale(box_plot_stats.non_family.q2) }
            stroke="black"
            width=80
        />
        <rect
            x={ xScale(bin_type) - boxWidth/2 + xScale.bandwidth() / 2}
            y={yScale(box_plot_stats.non_family.q3)} 
            width={boxWidth}
            height={yScale(box_plot_stats.non_family.q1) - yScale(box_plot_stats.non_family.q3)}
            stroke="black"
            fill="blue"
            fill-opacity=0.5
        />
        {/if}

        
        {#if bin_type === "mixed"}
        <line 
            x1={ xScale(bin_type) + xScale.bandwidth() / 2 }
            x2={ xScale(bin_type) + xScale.bandwidth() / 2 }
            y1={ yScale(box_plot_stats.mixed.min) }
            y2={ yScale(box_plot_stats.mixed.max) }
            stroke="black"
            width=40
        />
        <line
            x1={ xScale(bin_type) - boxWidth/2 + xScale.bandwidth() / 2 }
            x2={ xScale(bin_type) + boxWidth/2 + xScale.bandwidth() / 2 }
            y1={ yScale(box_plot_stats.mixed.q2) }
            y2={ yScale(box_plot_stats.mixed.q2) }
            stroke="black"
            width=80
        />
        <rect
            x={ xScale(bin_type) - boxWidth/2 + xScale.bandwidth() / 2}
            y={yScale(box_plot_stats.mixed.q3)} 
            width={boxWidth}
            height={yScale(box_plot_stats.mixed.q1) - yScale(box_plot_stats.mixed.q3)}
            stroke="black"
            fill="blue"
            fill-opacity=0.5
        />
        {/if}
        
    {/each}
    
    {#each data as d, index}
        
        <circle 
            class:selected={isDataSelected(d)}
            cx={ xScale(d.family_bins) + xScale.bandwidth() / 3.2 + Math.random()*80}
            cy={ yScale(d.eviction_rate) }
            r='3'
            fill="#B19CD9"
            on:mouseenter= {evt=> dotInteraction(index, evt)}
            on:mouseleave={evt => dotInteraction(index, evt)}
            tabindex="0"
            aria-describedby="eviction-tooltip"
            role="tooltip"
            aria-haspopup="true"
            on:focus={evt=> dotInteraction(index, evt)}
            on:blur={evt=> dotInteraction(index, evt)}
        />
        
    {/each}
    </g>
</svg>

<!-- <p>{hasSelection ? selectedEviction.length : "No"} eviction data selected</p> -->
    <!-- <dl class="meta_legend">
        <Pie data={languageBreakdownArray}></Pie>
    </dl> -->
    

