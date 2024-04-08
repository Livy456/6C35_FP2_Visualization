<script>
    import * as d3 from "d3";
    // import binned_csv from '$static/' ;
    import { onMount } from "svelte";
    import {
        computePosition,
        autoPlacement,
        offset
    } from '@floating-ui/dom';

    let data = [];
    let width = 800, height = 275; // changed the height of the graph from 600 to 450
    let yScale = d3.scaleLinear();
    let xScaleHousehold = d3.scaleBand();
    let xScaleRace = d3.scaleBand();
    let xScaleElder = d3.scaleBand();
    let rScale = d3.scaleSqrt();
    let xAxis, yAxis;
    let yAxisGridlines;
    let hoveredIndex = -1;
    $: hoveredEviction = data[hoveredIndex]?? {};
    let tooltipPosition = {x:0, y:0};
    let evictionTooltip;
    let svg;
    let input;
    let brushedSelection;
    let selectedEvictions;
    let hasSelection;
    let selectedLines;
    let boxWidth = 100;
    let family_bins = ["mixed", "non-family", "family"];
    let race_bins = ["White", "Black", "Latino", "Other"];
    let elder_bins = ["some elders", "no elders"];
    let data_mixed;
    let data_family;
    let data_non_family;
    let box_plot_stats_household = {mixed: {}, non_family: {}, family: {}}
    let box_plot_stats_race = {black: {}, white: {}, latino: {}, other: {}}
    let box_plot_stats_elderly = {some_elder: {}, no_elder: {}}
    let metric_to_graph = "Household Type";
    
    let languageBreakdown;
    let languageBreakdownArray;
    
    const format = d3.format(".1~%");
        
    // defining axes
    let margin = {top: 10, right: 10, bottom: 30, left:40};
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
        // d3.select(xAxis).call(d3.axisBottom(xScaleHousehold));

        // d3.select(yAxis).call(d3.axisLeft(yScale));
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

    // Changes the metric to Household type
    
    
    $: {
        d3.select(svg).call(d3.brush().on("start brush end", brushed));
        d3.select(svg).selectAll(".dots, .overlay ~ *").raise();
    }
    
    $: selectedEvictions = brushedSelection ? data.filter(isDataSelected) : [];    
    $: hasSelection = brushedSelection && selectedEvictions.length > 0;
    $: selectedLines = (hasSelection ? selectedEviction: data).flatMap(d => d.mhi);
    $: yScale = yScale.domain([0, 0.2]).range([usableArea.bottom, usableArea.top]); // might need to switch values currently domain = [0, height], range = [0, 24] 
    
    // Setting X scale
    $: xScaleHousehold = xScaleHousehold.domain(family_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);    
    $: xScaleRace = xScaleRace.domain(race_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);
    $: xScaleElder = xScaleElder.domain(elder_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);

    // Computing box plots for family bin data
    $: data_mixed = data.filter((d) => d.family_bins === "mixed");
    $: data_family = data.filter((d) => d.family_bins === "family");
    $: data_non_family = data.filter((d) => d.family_bins === "non-family");
    $: box_plot_stats_household.family = calculate_box_plot(data_family);
    $: box_plot_stats_household.non_family = calculate_box_plot(data_non_family);
    $: box_plot_stats_household.mixed = calculate_box_plot(data_mixed);

    // Computing box plots for race data
    $: data_black = data.filter((d) => d.majority_race === "Black");
    $: data_white = data.filter((d) => d.majority_race === "White");
    $: data_latino = data.filter((d) => d.majority_race === "Latino");
    $: data_other = data.filter((d) => d.majority_race === "Other");
    $: box_plot_stats_race.black = calculate_box_plot(data_black);
    $: box_plot_stats_race.white = calculate_box_plot(data_white);
    $: box_plot_stats_race.latino = calculate_box_plot(data_latino);
    $: box_plot_stats_race.other = calculate_box_plot(data_other);

    // Computing box plots for elder or no elders
    $: data_elders = data.filter((d) => d.elder_bins === "some elders");
    $: data_no_elders = data.filter((d) => d.elder_bins === "no elders");
    $: box_plot_stats_elderly.some_elder = calculate_box_plot(data_elders);
    $: box_plot_stats_elderly.no_elder = calculate_box_plot(data_no_elders);
    

    // function updateMetric(evt)
    // {
    //     let metric = evt.value;
    //     console.log("metric", evt);

    //     if (metric === "Household Type")
    //     {
    //         metric_to_graph = "Household Type";
    //     }

    //     else if (metric === "Race")
    //     {
    //         metric_to_graph = "Race";
    //     }

    //     else if (metric === "Elderly")
    //     {
    //         metric_to_graph = "Elderly";
    //     }
    // }

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
        console.log("binned_data", binned_data);

        for (let d of binned_data)
        {
            if (d.eviction_rate)
            {
                eviction_rate_array.push(d.eviction_rate);
            }
            
        }
        console.log(eviction_rate_array);

        let quantile1 = d3.quantile(eviction_rate_array, 0.25);
        let quantile2 = d3.quantile(eviction_rate_array, 0.5);
        let quantile3 = d3.quantile(eviction_rate_array, 0.75);
        let innerQuantileRange = quantile3 - quantile1;
        let minimum = quantile1 - .25*innerQuantileRange; // MIGHT NEED TO RECALCULATE THIS, ORIGINALLY -1.5*innerQuartileRange
        let maximum = quantile3 + 1.5*innerQuantileRange;
        let summary_stats = {q1: quantile1, q2: quantile2, q3:quantile3, innerQuantile:innerQuantileRange, min: minimum, max:maximum }
        
        console.log(summary_stats);

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
        let data_x_coord = xScaleHousehold(dot.family_bins);
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
<!-- on:click={ function() {metric_to_graph= "Race"} } -->
<div class="metric_selection">
    <input type="button" value="Race" on:click={ function() {metric_to_graph= "Race"} }>
    <input type="button" value="Elderly" on:click={ function() {metric_to_graph= "Elderly"} }>
    <input type="button" value="Household Type" on:click={ function() {metric_to_graph= "Household Type"} }>
</div>

<h2 style="margin-top: 3rem">Number of Evictions vs {metric_to_graph}</h2>
    
<dl id="eviction-tooltip" class="info tooltip" 
    hidden={hoveredIndex === -1}
    bind:this={evictionTooltip}
    style="top:{tooltipPosition.y}px; left:{tooltipPosition.x}px">
    <dt>Eviction Rate</dt>
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
    {console.log("Before the Graphing!!")}
    {console.log(metric_to_graph)}
    {console.log("data:", data)}
    

    {#if metric_to_graph === "Household Type"}
        {d3.select(xAxis).call(d3.axisBottom(xScaleHousehold))}
        {#each family_bins as bin_type}
            <!-- {console.log("Hello!!!")} -->
            {#if bin_type === "family"}
            <line 
                x1={ xScaleHousehold(bin_type) + xScaleHousehold.bandwidth() / 2 }
                x2={ xScaleHousehold(bin_type) + xScaleHousehold.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_household.family.min) }
                y2={ yScale(box_plot_stats_household.family.max) }
                stroke="black"
                width=40
            />
            <line
                x1={ xScaleHousehold(bin_type) - boxWidth/2 + xScaleHousehold.bandwidth() / 2 }
                x2={ xScaleHousehold(bin_type) + boxWidth/2 + xScaleHousehold.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_household.family.q2) }
                y2={ yScale(box_plot_stats_household.family.q2) }
                stroke="black"
                width=80
            />

            <rect
                x={ xScaleHousehold(bin_type) - boxWidth/2 + xScaleHousehold.bandwidth() / 2}
                y={ yScale(box_plot_stats_household.family.q3) } 
                width={boxWidth}
                height={yScale(box_plot_stats_household.family.q1) - yScale(box_plot_stats_household.family.q3)}
                stroke="black"
                fill="blue"
                fill-opacity=0.5
            />
            {/if}

            {#if bin_type === "non-family"}
            <line 
                x1={ xScaleHousehold(bin_type) + xScaleHousehold.bandwidth() / 2 }
                x2={ xScaleHousehold(bin_type) + xScaleHousehold.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_household.non_family.min) }
                y2={ yScale(box_plot_stats_household.non_family.max) }
                stroke="black"
                width=40
            />
            <line
                x1={ xScaleHousehold(bin_type) - boxWidth/2 + xScaleHousehold.bandwidth() / 2 }
                x2={ xScaleHousehold(bin_type) + boxWidth/2 + xScaleHousehold.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_household.non_family.q2) }
                y2={ yScale(box_plot_stats_household.non_family.q2) }
                stroke="black"
                width=80
            />
            <rect
                x={ xScaleHousehold(bin_type) - boxWidth/2 + xScaleHousehold.bandwidth() / 2}
                y={ yScale(box_plot_stats_household.non_family.q3) } 
                width={boxWidth}
                height={yScale(box_plot_stats_household.non_family.q1) - yScale(box_plot_stats_household.non_family.q3)}
                stroke="black"
                fill="blue"
                fill-opacity=0.5
            />
            {/if}

            {#if bin_type === "mixed"}
            <line 
                x1={ xScaleHousehold(bin_type) + xScaleHousehold.bandwidth() / 2 }
                x2={ xScaleHousehold(bin_type) + xScaleHousehold.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_household.mixed.min) }
                y2={ yScale(box_plot_stats_household.mixed.max) }
                stroke="black"
                width=40
            />
            <line
                x1={ xScaleHousehold(bin_type) - boxWidth/2 + xScaleHousehold.bandwidth() / 2 }
                x2={ xScaleHousehold(bin_type) + boxWidth/2 + xScaleHousehold.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_household.mixed.q2) }
                y2={ yScale(box_plot_stats_household.mixed.q2) }
                stroke="black"
                width=80
            />
            <rect
                x={ xScaleHousehold(bin_type) - boxWidth/2 + xScaleHousehold.bandwidth() / 2}
                y={ yScale(box_plot_stats_household.mixed.q3) } 
                width={boxWidth}
                height={yScale(box_plot_stats_household.mixed.q1) - yScale(box_plot_stats_household.mixed.q3)}
                stroke="black"
                fill="blue"
                fill-opacity=0.5
            />
            {/if}
            
        {/each}

        {#each data as d, index}
        
            <circle 
                class:selected={isDataSelected(d)}
                cx={ xScaleHousehold(d.family_bins) + xScaleHousehold.bandwidth() / 3.2 + Math.random()*80}
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
    {/if}

    <!-- GRAPH FOR RACE METRIC -->
    {#if metric_to_graph === "Race"}
        {d3.select(xAxis).call(d3.axisBottom(xScaleRace))}
        
        {#each race_bins as bin_type}
            
            <!-- box plot for black people -->
            {#if bin_type === "Black"}
            <line 
                x1={ xScaleRace(bin_type) + xScaleRace.bandwidth() / 2 }
                x2={ xScaleRace(bin_type) + xScaleRace.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_race.black.min) }
                y2={ yScale(box_plot_stats_race.black.max) }
                stroke="black"
                width=40
            />
            <line
                x1={ xScaleRace(bin_type) - boxWidth/2 + xScaleRace.bandwidth() / 2 }
                x2={ xScaleRace(bin_type) + boxWidth/2 + xScaleRace.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_race.black.q2) }
                y2={ yScale(box_plot_stats_race.black.q2) }
                stroke="black"
                width=80
            />

            <!-- FIX THE HEIGHT OF THE BOX -->
            <!-- {yScale(box_plot_stats_race.black.q1) - yScale(box_plot_stats_race.black.q3)} -->
            <!-- { yScale(box_plot_stats_race.black.q3)} -->
            <rect
                x={ xScaleRace(bin_type) - boxWidth/2 + xScaleRace.bandwidth() / 2}
                y= { yScale(box_plot_stats_race.black.q3)}
                width={boxWidth}
                height={yScale(box_plot_stats_race.black.q1) - yScale(box_plot_stats_race.black.q3)}
                stroke="black"
                fill="blue"
                fill-opacity=0.5
            />
            {/if}

            {#if bin_type === "White"}
            <line 
                x1={ xScaleRace(bin_type) + xScaleRace.bandwidth() / 2 }
                x2={ xScaleRace(bin_type) + xScaleRace.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_race.white.min) }
                y2={ yScale(box_plot_stats_race.white.max) }
                stroke="black"
                width=40
            />
            <line
                x1={ xScaleRace(bin_type) - boxWidth/2 + xScaleRace.bandwidth() / 2 }
                x2={ xScaleRace(bin_type) + boxWidth/2 + xScaleRace.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_race.white.q2) }
                y2={ yScale(box_plot_stats_race.white.q2) }
                stroke="black"
                width=80
            />

            <rect
                x={ xScaleRace(bin_type) - boxWidth/2 + xScaleRace.bandwidth() / 2}
                y= { yScale(box_plot_stats_race.white.q3)}
                width={boxWidth}
                height={yScale(box_plot_stats_race.white.q1) - yScale(box_plot_stats_race.white.q3)}
                stroke="black"
                fill="blue"
                fill-opacity=0.5
            />
            {/if}

            {#if bin_type === "Latino"}
            <line 
                x1={ xScaleRace(bin_type) + xScaleRace.bandwidth() / 2 }
                x2={ xScaleRace(bin_type) + xScaleRace.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_race.latino.min) }
                y2={ yScale(box_plot_stats_race.latino.max) }
                stroke="black"
                width=40
            />
            <line
                x1={ xScaleRace(bin_type) - boxWidth/2 + xScaleRace.bandwidth() / 2 }
                x2={ xScaleRace(bin_type) + boxWidth/2 + xScaleRace.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_race.latino.q2) }
                y2={ yScale(box_plot_stats_race.latino.q2) }
                stroke="black"
                width=80
            />

            <rect
                x={ xScaleRace(bin_type) - boxWidth/2 + xScaleRace.bandwidth() / 2}
                y= { yScale(box_plot_stats_race.latino.q3)}
                width={boxWidth}
                height={yScale(box_plot_stats_race.latino.q1) - yScale(box_plot_stats_race.latino.q3)}
                stroke="black"
                fill="blue"
                fill-opacity=0.5
            />
            {/if}

            {#if bin_type === "Other"}
            <line 
                x1={ xScaleRace(bin_type) + xScaleRace.bandwidth() / 2 }
                x2={ xScaleRace(bin_type) + xScaleRace.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_race.other.min) }
                y2={ yScale(box_plot_stats_race.other.max) }
                stroke="black"
                width=40
            />
            <line
                x1={ xScaleRace(bin_type) - boxWidth/2 + xScaleRace.bandwidth() / 2 }
                x2={ xScaleRace(bin_type) + boxWidth/2 + xScaleRace.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_race.other.q2) }
                y2={ yScale(box_plot_stats_race.other.q2) }
                stroke="black"
                width=80
            />

            <rect
                x={ xScaleRace(bin_type) - boxWidth/2 + xScaleRace.bandwidth() / 2}
                y= { yScale(box_plot_stats_race.other.q3)}
                width={boxWidth}
                height={yScale(box_plot_stats_race.other.q1) - yScale(box_plot_stats_race.other.q3)}
                stroke="black"
                fill="blue"
                fill-opacity=0.5
            />
            {/if}

        {/each}

        {#each data as d, index}
        
            <circle 
                class:selected={isDataSelected(d)}
                cx={ xScaleRace(d.majority_race) + xScaleRace.bandwidth() / 3.2 + Math.random()*80}
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
    {/if}

    <!-- PLOT FOR ELDER METRIC -->
    {#if metric_to_graph === "Elderly"}
        {d3.select(xAxis).call(d3.axisBottom(xScaleElder))}
        {#each elder_bins as bin_type}
            
            <!-- box plot for black people -->
            {#if bin_type === "some elders"}
            <line 
                x1={ xScaleElder(bin_type) + xScaleElder.bandwidth() / 2 }
                x2={ xScaleElder(bin_type) + xScaleElder.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_elderly.some_elder.min) }
                y2={ yScale(box_plot_stats_elderly.some_elder.max) }
                stroke="black"
                width=40
            />
            <line
                x1={ xScaleElder(bin_type) - boxWidth/2 + xScaleElder.bandwidth() / 2 }
                x2={ xScaleElder(bin_type) + boxWidth/2 + xScaleElder.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_elderly.some_elder.q2) }
                y2={ yScale(box_plot_stats_elderly.some_elder.q2) }
                stroke="black"
                width=80
            />

            <rect
                x={ xScaleElder(bin_type) - boxWidth/2 + xScaleElder.bandwidth() / 2}
                y={yScale(box_plot_stats_elderly.some_elder.q3)} 
                width={boxWidth}
                height={yScale(box_plot_stats_elderly.some_elder.q1) - yScale(box_plot_stats_elderly.some_elder.q3)}
                stroke="black"
                fill="blue"
                fill-opacity=0.5
            />
            {/if}

            {#if bin_type === "no elders"}
            <line 
                x1={ xScaleElder(bin_type) + xScaleElder.bandwidth() / 2 }
                x2={ xScaleElder(bin_type) + xScaleElder.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_elderly.no_elder.min) }
                y2={ yScale(box_plot_stats_elderly.no_elder.max) }
                stroke="black"
                width=40
            />
            <line
                x1={ xScaleElder(bin_type) - boxWidth/2 + xScaleElder.bandwidth() / 2 }
                x2={ xScaleElder(bin_type) + boxWidth/2 + xScaleElder.bandwidth() / 2 }
                y1={ yScale(box_plot_stats_elderly.no_elder.q2) }
                y2={ yScale(box_plot_stats_elderly.no_elder.q2) }
                stroke="black"
                width=80
            />
            <rect
                x={ xScaleElder(bin_type) - boxWidth/2 + xScaleElder.bandwidth() / 2}
                y={yScale(box_plot_stats_elderly.no_elder.q3)} 
                width={boxWidth}
                height={yScale(box_plot_stats_elderly.no_elder.q1) - yScale(box_plot_stats_elderly.no_elder.q3)}
                stroke="black"
                fill="blue"
                fill-opacity=0.5
            />
            {/if}

        {/each}

        {#each data as d, index}
        
            <circle 
                class:selected={isDataSelected(d)}
                cx={ xScaleElder(d.elder_bins) + xScaleElder.bandwidth() / 3.2 + Math.random()*80}
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
    {/if}
    
    </g>
</svg>

<!-- <p>{hasSelection ? selectedEviction.length : "No"} eviction data selected</p> -->
    <!-- <dl class="meta_legend">
        <Pie data={languageBreakdownArray}></Pie>
    </dl> -->
    

