<script>
    import * as d3 from "d3";
    import {
        computePosition,
        autoPlacement,
        offset
    } from '@floating-ui/dom';
    import { onMount } from "svelte";
    // import { spring } from "svelte/motion";
    export let text = "";
    export let data = [];
    export let bins = [];
    export let metric = "";

    let temp_bins = ["Really warm", "Warm", "Cold", "Really Cold"];
    let margin = {top: 10, right: 10, bottom: 30, left:40};
    let height= 300;
    let width = 500;
    let startX = 5;
    let startY = height/3 ;
    let endX = width;
    let endY = Math.random()*height;
    let hoveredIndex = -1;
    $: hoveredEviction = data[hoveredIndex]?? {};
    let tooltipPosition = {x:0, y:0};
    let evictionTooltip;
    let xAxis, yAxis;
    let yAxisGridlines;
    let transitionDuration = 10000;
    let usableArea = {
        top: margin.top,
        bottom: height - margin.bottom,
        left: margin.left,
        right: width - margin.right
    };
    // let yScale = d3.scaleLinear();
    let yScale = d3.scaleBand();
    let winter_data = [];
    let spring_data = [];
    let summer_data = [];
    let fall_data = [];
    // let circle;
    let svg;
    let circle;
    let container;
    let fillColor = d3.scaleOrdinal(d3.schemeCategory10);
    const format = d3.format(".1~%");

    // $: data = data.filter( (d) => d.month < 5);
    // $: console.log(winter_data);
    // $: console.log(data);
    // $: yScale = yScale.domain([0, 12]).range([usableArea.bottom, usableArea.top]);
    $: yScale = yScale.domain(temp_bins).range([usableArea.bottom, usableArea.top]);
    $: {
        d3.select(yAxisGridlines).call(d3.axisRight(yScale).tickSize(width-100));
    }

    // $:{
    //     console.log( d3.select("svg").selectAll("circle") );
    // }

    // $: onMount ( () => {
    //     container = d3.select(".visual");
    //     circles = container.selectAll(".moving_dots");

    // });
    // $: svg = d3.select(svg);
    // $: console.log(svg);
    // $: console.log(circles);
    // $: console.log(container);
    

    // $: data = data.slice(0, 11); // gets the first 10 points of data 
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

    function computeYposition(month)
    {

    }

    function dataColoring(d, bin_type, i)
    {
        if (bin_type.includes("Family"))
        {
            return fillColor(d.family_bins);
        }

        else if (bin_type.includes("Race"))
        {
            return fillColor(d.majority_race);
        }

        else if (bin_type.includes("Elder"))
        {
            return fillColor(d.elder_bins);
        }

        else if (bin_type.includes("Corporate"))
        {
            return fillColor(d.corp_bins)
        }
    }

    // classifies the month attribute from the dataset to a temperature
    // which will be used to graph the y position of the data point
    function convertMonthToTemp(month)
    {
        // temperature bins: ["Really warm", "Warm", "Cold", "Really Cold"]
        // console.log(month);
        if((9 <= month) && (month < 11))
        {
            // console.log("Cold");
            return "Cold";
        }
        // really cold temperatures
        
        if (((1 <= month ) && (month < 4)) || (11 <= month))
        {
            // console.log("Really Cold");
            return "Really Cold";
        }

        // warm temperatures
        if ((4 <= month) && (month < 6 ))
        {
            // console.log("Warm");
            return "Warm";
        }

        // really warm temperatures
        if(6 <= month < 9)
        {
            // console.log("Really Warm");
            return "Really warm";
        }
    }

    let durations = ["5s", "10s", "20s"];

</script>
<style>

@keyframes moveCircles{
    to{
        /* final x position of cirlce */
        cx: 425;
        cy: var(--yposition);
        /* cy: 15; */
    }
}
.moving_dots{
    /* calc(500* var(--index))ms; */
    opacity: 75%;
    animation-name: moveCircles;
    /* animation-delay: calc(var(--index) * 100ms); */
    animation-duration: 20s;
    animation-timing-function:  ease-in-out infinite;
    /* animation-delay: calc(v ar(--index) * 100ms); */
    /* :nth-child(10)
    {
        animation-delay: var(--index)s;
    } */
}

.moving_dots:nth-child(1)
{
    animation-delay: 0s;
}

.moving_dots:nth-child(2)
{
    animation-delay: 3s;
}

.moving_dots:nth-child(3)
{
    animation-delay: 10s;
}

.moving_dots:nth-child(15)
{
    animation-delay: 5s;
}

.description{
    border: 1px solid black;
    border-radius:3px;
    padding:10px;
    box-shadow:5px 5px 5px grey;
    margin-right:10px;
    row-gap:1fr;
}

.bin_legend{
    display: grid;
    grid-template-rows: 10fr 10fr;
    padding:50px;
    margin-bottom:20px;
    border-bottom:1px solid black;
    box-shadow: 5px 5px 5px grey;
    opacity:75%;
    border-radius:5px;
    margin-right:20px;
    margin-left:30px;
    
    label{
        grid-row:1;
        padding:10px;
    }
    div{
        grid-row:2;
        padding:5px;
        margin-left:15px;
        border-radius:40px;
        /* background-color:red; */
        border:4px solid black;
        width:10px;
        height:10px;
    }
}

.entire_visualization{
    display:grid;
    grid-template-columns: 10fr 10fr;
    margin-bottom:20px;

    .visual{
        width:400px;
        height:300px;
        /* background-color:blue; */
        opacity:80%;
        grid-column:1;
        margin:10px;
        /* padding:5px; */
    }

    .temp_legend{
    display:grid;
    grid-template-rows: auto auto auto auto;
        div{
            grid-row:1;
            border-radius:50px;
            background-color:red;
            border:4px solid black;
            width:10px;
            height:10px;
            subgrid-row:1;
        }  
        .percent_labels{
            grid-row: span 3;
        }
    }
}
/* 
svg{
    background-color: black;

} */

</style>
<dl class="animation_container">
    <section class="bin_legend">
        {#each bins as bin}
            <label>{bin}</label>
            <div style="background-color:{fillColor(bin)}"></div>
        {/each}
    </section>

    <dl id="eviction-tooltip" class="info tooltip" 
        hidden={hoveredIndex === -1}
        bind:this={evictionTooltip}
        style="top:{tooltipPosition.y}px; left:{tooltipPosition.x}px">
        <dt>Eviction Rate</dt>
        <dd> { format(hoveredEviction.eviction_rate, ".3f") } </dd>
    
        <dt>Eviction Month</dt>
        <dd>{ hoveredEviction.month }</dd>
    
        <dt>Eviction Year</dt>
        <dd> { hoveredEviction.year } </dd>
       
    </dl>

    <div class="entire_visualization">
        <section class="visual">
            <!-- x, y, width, height -->
            <!-- viewBox="0 0 180 140" -->
            <svg  
                width={width} height={height}> 
                <g class="gridlines" transform="translate({usableArea.left}, 0)" bind:this={yAxisGridlines} />
                <g transform="translate({usableArea.top})" bind:this={yAxis}/>
                {#each data as d, index}
                    <circle
                        class="moving_dots"
                        cx={ startX +Math.random()*60}
                        cy={ startY + Math.random()*200}
                        r="5"
                        fill={dataColoring(d, metric, index)}
                        stroke="white"
                        style="
                        --animation-duration: { durations[index% durations.length] };
                        --yposition:{ yScale( convertMonthToTemp(d.month) ) + yScale.bandwidth() / 2 + Math.random()*20}"
                        on:mouseenter= {evt=> dotInteraction(index, evt)}
                        on:mouseleave={evt => dotInteraction(index, evt)} 
                    />
                    <!-- animation-delay:{index * 100}ms -->
                    <!-- --index:{index} -->
                    <!-- --yposition:{yScale( convertMonthToTemp(d.month) ) + yScale.bandwidth() / 2 + Math.random()*20} -->
                    <!-- yposition css variable moves the dots to the appropriate bins -->
                {/each}
            </svg>
        </section>
        <section class="temp_legend">
            {#each bins as bin, i}
                <!-- <dt>{bin}</dt> -->
                <div 
                    class="bin_circles"
                    style="background-color:{fillColor(bin)}"
                />
    
                {#each temp_bins as bin, index}
                    <!-- <dd class="percent_labels">(bin, temp): ({index}, {i})%</dd> -->
                    <dd class="percent_labels">%</dd>
                {/each}
            {/each}
        </section>
    </div>
    
    <section class="description">
        <p>
            {text}
        </p>
    </section>
</dl>

