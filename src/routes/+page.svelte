<script>
    import * as d3 from "d3";
    import { onMount } from "svelte";
    import BinGraph from "./BinGraph.svelte";
    import DotAnimation from "./DotAnimation.svelte";
    import {
        computePosition,
        autoPlacement,
        offset
    } from '@floating-ui/dom';

    let dataRAW  = [];
    let data = [];
    let temp_data = [];
    let temp_dataRAW = [];
    let width = 800, height = 275; // changed the height of the graph from 600 to 450
    let yScale = d3.scaleLinear();
    let xScaleHousehold = d3.scaleBand();
    let xScaleRace = d3.scaleBand();
    let xScaleElder = d3.scaleBand();
    let xScaleCorp = d3.scaleBand();
    let yAxisGridlines;
    let brushedSelection;
    let selectedEvictions;
    let hasSelection;
    let selectedLines;
    let family_bins = ["family", "non-family", "mixed"];
    let race_bins = ["Black", "White", "Latino", "Other"];
    let elder_bins = ["some elders", "no elders"];
    let corp_bins = ["low", "medium", "high"];
    let text_description = ["Majority, more than 50%, of the tenants in the house are a particular race.", 
    "The tenants in the house are either all family members, not related at all, or a mix of family members and non family members.", 
    "The housing property either contains some elderly people or no elderly people.", 
    "The housing properties in the area have a majority of low, medium, or high corporate ownership."];
    let box_plot_stats_household_array = [];
    let box_plot_stats_race_array = [];
    let box_plot_stats_elder_array = [];
    let box_plot_stats_corp_array = [];
    let metric_to_graph = "Family Type";
    let checkbox_pushed = false;
    let hoveredIndex = -1;
    $: hoveredEviction = text_description[hoveredIndex]?? {};
    let tooltipPosition = {x:0, y:0};
    let evictionTooltip;
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
        d3.select(yAxisGridlines).call(d3.axisLeft(yScale).tickSize(-usableArea.width));
    }

    onMount(async() => {
        dataRAW  = await d3.csv("binned_data.csv", row=> ({
            ...row,
            mhi: Number(row.mhi),
            eviction_rate: Number(row.eviction_rate)
        }));

        temp_dataRAW = await d3.csv("evictions_per_month_with_bins.csv", row=> ({
            ...row,
            mhi: Number(row.mhi),
            eviction_rate: Number(row.eviction_rate),
            year: Number(row.year),
            month: Number(row.month),
            filings: Number(row.filings)
        }));

        // temp_data

    });

    // $: data  = radio_button_pushed ? dataRAW.filter((d) => d.month >= 4).filter((d) => d.month < 11).filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1) : dataRAW.filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1).filter((d) => d.mhi > minVal ).filter((d) => d.mhi < maxVal);
    $: data = dataRAW.filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1).filter((d) => d.mhi > minVal ).filter((d) => d.mhi < maxVal);
    $: temp_data = checkbox_pushed ? temp_dataRAW.filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1).filter((d) => d.majority_race !== '').filter((d) => d.month >= 4).filter((d) => d.month < 11) : temp_dataRAW.filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1).filter((d) => d.majority_race !== '').filter((d) => d.mhi > minVal).filter((d) => d.mhi < maxVal);
    $: selectedEvictions = brushedSelection ? data.filter(isDataSelected) : [];    
    $: hasSelection = brushedSelection && selectedEvictions.length > 0;
    $: selectedLines = (hasSelection ? selectedEviction: data).flatMap(d => d.mhi);
    $: yScale = yScale.domain([0, 0.2]).range([usableArea.bottom, usableArea.top]); // might need to switch values currently domain = [0, height], range = [0, 24] 
    
    // Setting X scale
    $: xScaleHousehold = xScaleHousehold.domain(family_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);    
    $: xScaleRace = xScaleRace.domain(race_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);
    $: xScaleElder = xScaleElder.domain(elder_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);
    $: xScaleCorp = xScaleCorp.domain(corp_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);

    // Computing box plots for family binned data
    $: box_plot_stats_household_array = computeStats(family_bins, "family", data);

    // Computing box plots for race binned data
    $: box_plot_stats_race_array = computeStats(race_bins, "race", data);

    // Computing box plots for elder binned data
    $: box_plot_stats_elder_array = computeStats(elder_bins, "elder", data);

    // Computing box plots for corporate ownership binned data
    $: box_plot_stats_corp_array = computeStats(corp_bins, "corporate", data);

    async function textInteraction (index, evt){
        let hoveredDot = evt.target;
        
        if (evt.type === "mouseenter" || evt.type === "focus")
        {
            hoveredIndex = index;
            tooltipPosition = await computePosition(hoveredDot, evictionTooltip, {
                strategy: "fixed",
                middleware: [
                    offset(1),
                    autoPlacement()
                ],
            });
        }

        else if (evt.type === "mouseleave" || evt.type === "blur")
        {
            hoveredIndex = -1;
        }
    } 

    // compute boxplots for each category for a bin (family, race, elder, corporate ownership)
    function computeStats(bins, metric, data)
    {
        let summary_stats = [];
        let data_filtered;

        for(let bin of bins)
        {
            // checks if metric to graph is associated with household type, family/non family/mixed
            if(metric === "family")
            {
                data_filtered = data.filter((d) => d.family_bins === bin);
            }

            // checks if metric to graph is associated with race, white/black/latino/other
            else if(metric === "race")
            {
                data_filtered = data.filter((d) => d.majority_race === bin);
            }
            
            // checks if metric to graph is associated with elder, some elders/no elders
            else if(metric === "elder")
            {
                data_filtered = data.filter((d) => d.elder_bins === bin);
            }

            // assumes the metric to graph is associated with corporate ownership, low/medium/high
            else
            {
                data_filtered = data.filter((d) => d.corp_bins === bin);
            }
            
            let summary_stat = calculate_box_plot(data_filtered);
            summary_stats.push(summary_stat);
        }
        return summary_stats;
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
        let minimum = quantile1 - 1.5*innerQuantileRange;
        let maximum = quantile3 + 1.5*innerQuantileRange;
        let summary_stats = {q1: quantile1, q2: quantile2, q3:quantile3, 
            innerQuantile:innerQuantileRange, min: d3.max([minimum, 0]), 
            max:d3.min([maximum, height]) }
            // makes sure that the min and max lines do not exceed the bounds of the plot

        return summary_stats
    } 
    function checkedBox(evt)
    {
        console.log(evt);
        if(checkbox_pushed === true)
        {
            checkbox_pushed = false;
        }

        if(checkbox_pushed === false)
        {
            checkbox_pushed = true;
        }
    }

    ////////////////// JS for the income slider //////////////////

    let minVal = 10000;
    let maxVal = 200000;
    const minGap = 30000;
    const sliderMinValue = 10000;
    const sliderMaxValue = 200000;
    const stepValue = 1000;

    // Reactive styling for slider track
    $: minPercent = ((minVal - sliderMinValue) / (sliderMaxValue - sliderMinValue)) * 100;
    $: maxPercent = ((maxVal - sliderMinValue) / (sliderMaxValue - sliderMinValue)) * 100;

    // Enforce the minimum gap dynamically, broken
    $: {
        if (maxVal - minVal < minGap) {
            // If decreasing maxVal or increasing minVal, adjust the other to maintain the gap
            if (maxVal - minVal < minGap) {
                maxVal = minVal + minGap;
            }

            if (minVal < sliderMinValue) {
                minVal = sliderMinValue;
                maxVal = minVal + minGap;
            }
            if (maxVal > sliderMaxValue) {
                maxVal = sliderMaxValue;
                minVal = maxVal - minGap;
            }
        }
    }

    // How to handle typed inputs
    function handleMinInput(event) {
        const inputVal = Math.max(Number(parseMoney(event.target.value)), sliderMinValue);
        minVal = Math.min(inputVal, maxVal - minGap);
    }

    // How to handle typed inputs
    function handleMaxInput(event) {
        const inputVal = Math.min(Number(parseMoney(event.target.value)), sliderMaxValue);
        maxVal = Math.max(inputVal, minVal + minGap);
    }

    // Checks with user has hit enter to change slider
    function checkEnterKey(event, handler) {
        if (event.key === 'Enter') {
        handler(event);
        event.target.blur(); // Optionally blur the input after processing to mimic the blur behavior
        }
    }

    // Function to format numbers as money
    function formatIncome(value) {
        return value.toLocaleString('en-US');
    }

    // Function to parse formatted money back to number
    function parseMoney(value) {
        return parseInt(value.replace(/[\$,]/g, ''), 10);
    }

    //////////////////////////////////////////////////////////////

</script>

<style>
     /* CSS for Income Selector */

     .double-slider-box {
        margin: 0;
        box-sizing: border-box;
        display: grid;
        max-width: 30em;
        /* background-color: rgb(207, 207, 207); */
        padding: 20px 40px;
        border-radius: 1em;
        font-family: 'Helvetica Neue';
    }

    h3.range-title{

        text-align: center;

    }

    p.range-description {

        text-align: center;
        margin-bottom: 1em;
        margin-top: -0.5em;

    }

    .range-slider {

        position: relative;
        width: 100%;
        height: 5px;
        margin: 30px 0;
        background-color: rgb(114, 114, 114);

    }

    .slider-track {

        height: 100%;
        position: absolute;
        background-color: firebrick;

    }

    .range-slider input {

        position: absolute;
        width: 100%;
        background: none;
        pointer-events: none;
        top: 50%;
        transform: translateY(-60%);
        appearance: none;

    }

    input[type="range"]::-webkit-slider-thumb {

        height: 20px;
        width: 20px;
        border-radius: 50%;
        border: 3px solid #FFFFFF;
        background: #FFFFFF;
        pointer-events: auto;
        appearance: none;
        cursor: pointer;
        box-shadow: 0 0.125em 0.5625em rgba(0, 0, 0, 0.25)

    }

    /* this is the same thing, but for the Firefox (uncessary?) */
    input[type="range"]::-moz-slider-thumb {

        height: 25px;
        width: 25px;
        border-radius: 50%;
        border: 3px solid #FFFFFF;
        background: #FFFFFF;
        pointer-events: auto;
        cursor: pointer;
        -moz-appearance: none;
        box-shadow: 0 0.125rem 0.5625rem rgba(0, 0, 0, 0.25)

    }

    .input-box {

        display: flex;

    }

    .min-box, 
    .max-box {

        width: 50%;

    }

    .min-box {

        padding-right: 0.5em;
        margin-right: 0.5em;

    }

    .input-wrap {

        position: relative;
        display: flex;
        flex-wrap: wrap;
        align-items: stretch;
        width: 100%;
        
    }

    .input-addon {

        display: flex;
        align-items: center;
        padding: 0.625em 1em;
        font-size: 1em;
        font-weight: 400;
        line-height: 1.5;
        color: rgb(0, 0, 0);
        text-align: center;
        white-space: nowrap;
        background-color: rgb(181, 181, 181);
        border: 1px solid rgb(0, 0, 0);
        border-radius: 0.5em;
        border-top-right-radius: 0;
        border-bottom-right-radius: 0;

    }

    .input_field-min-input, 
    .input_field-max-input{

        margin-left: -1px;
        padding: 0.425em;
        font-size: 1em;
        border-radius: 0.5em;
        position: relative;
        flex: 1 1 auto;
        width: 1%;
        min-width: 0;
        color: rgb(0, 0, 0);
        background-color: white;
        background-clip: padding-box;
        border: 1px solid black;
        border-top-left-radius: 0;
        border-bottom-left-radius: 0;
    }

    /* CSS for Income Selector */
</style>

<h3 class="meta">
    <strong style="color: blue;">Interactive Voices:</strong>
    Mapping the Movement for Eviction Reform and Advocacy
</h3>

<div class="full_page">

    <div class="narrative1">
        <section>
            <h2>Massachusetts Needs Winter Eviction Bans</h2>
            <p>
                Despite ranking as one of the coldest major cities in America [1], Boston lacks eviction protection during winter months.
                To quote the article, “Mass. should ban evictions during winter months” written by Gary Klein and Sarah Rosenkrantz, 
                “...cities like Chicago and Washington, DC ban evictions during cold weather. And both New York and Connecticut 
                have bills pending in their legislatures that would permanently ban most evictions during the winter months.”  
                So this begs the question, why does Boston have no such legislation? Boston’s lack of winter protection 
                affects historically discriminated groups the most: families, the elderly, and people of color.  
                The goal of our visualization is to show the effect a winter eviction moratorium would have Boston’s population.
            </p>
        </section>
    </div>

    <div class="selection_column">
        <div class="double-slider-box">
            <h3 class="range-title">Yearly Income</h3>
            <p class="range-description">Use the slider tool to select a minimum and maximum income</p>
            <div class="range-slider">
                <span class="slider-track" style="left: {minPercent + 1}%; right: {100 - maxPercent}%"></span>
                <input type="range" bind:value={minVal} class="min-val" min={sliderMinValue} max={sliderMaxValue} step={stepValue}>
                <input type="range" bind:value={maxVal} class="max-val" min={sliderMinValue} max={sliderMaxValue} step={stepValue}>
            </div>
            <div class="input-box">
                <div class="min-box">
                    <div class="input-wrap">
                        <span class="input-addon">$</span>
                        <input type="text" class="input_field-min-input"
                               value={formatIncome(minVal)} 
                               on:blur={handleMinInput}
                               on:keydown={(event) => checkEnterKey(event, handleMinInput)}>
                    </div>
                </div>
                <div class="max-box">
                    <div class="input-wrap">
                        <span class="input-addon">$</span>
                        <input type="text" class="input_field-max-input"
                               value={formatIncome(maxVal)} 
                               on:blur={handleMaxInput}
                               on:keydown={(event) => checkEnterKey(event, handleMaxInput)}>
                    </div>
                </div>
            </div>
        </div>

        <dl id="eviction-tooltip" class="info tooltip" 
            hidden={hoveredIndex === -1}
            bind:this={evictionTooltip}
            style="top:{tooltipPosition.y}px; left:{tooltipPosition.x}px">
            <dt>Metric Description</dt>
            <dd> { hoveredEviction } </dd>
        </dl>
        
        <div class="metric_selection">
            <h4 on:mouseenter= {evt=> textInteraction(0, evt)}
                on:mouseleave={evt => textInteraction(0, evt)}>
                Race
            </h4>
            <dl class="race_selection">
                <input type="button" value="White" 
                on:click={ function() {metric_to_graph= "Race White"} }>
                
                <input type="button" value="Black" 
                on:click={ function() {metric_to_graph= "Race Black"} }>

                <input type="button" value="Latino" 
                on:click={ function() {metric_to_graph= "Race Latino"} }>

                <input type="button" value="Other" 
                on:click={ function() {metric_to_graph= "Race Other"} }>
            </dl>
            
            <h4
                on:mouseenter= {evt=> textInteraction(1, evt)}
                on:mouseleave={evt => textInteraction(1, evt)}>
                Family Type
            </h4>
            <dl class="family_selection">
                <input type="button" value="Mixed" 
                on:click={ function() {metric_to_graph= "Family Type Mixed"} }>

                <input type="button" value="Non Family" 
                on:click={ function() {metric_to_graph= "Family Type Non-Fam"} }>

                <input class="bottom_row" type="button" value="Family" 
                on:click={ function() {metric_to_graph= "Family Type FamilyType"} }>
            </dl>
            
            <h4
                on:mouseenter= {evt=> textInteraction(2, evt)}
                on:mouseleave={evt => textInteraction(2, evt)}>
                Elderly Population

            </h4>
            <dl class="elderly_selection">
                <input type="button" value="No Elderly" 
                on:click={ function() {metric_to_graph= "No Elderly"} }>

                <input type="button" value="Some Elderly" 
                on:click={ function() {metric_to_graph= "Some Elderly"} }>
            </dl>
            
            <h4
                on:mouseenter= {evt=> textInteraction(3, evt)}
                on:mouseleave={evt => textInteraction(3, evt)}>
                Corporate Ownership
            </h4>
            <dl class="corporate_selection">
                <input type="button" value="Low Corporate Ownership" 
                on:click={ function() {metric_to_graph= "Corporate Ownership Low"} }>

                <input type="button" value="Medium Corporate Ownership" 
                on:click={ function() {metric_to_graph= "Corporate Ownership Medium"} }>

                <input type="button" value="High Corporate Ownership" 
                on:click={ function() {metric_to_graph= "Corporate Ownership High"} }>
            </dl>
        </div>
    </div>
    
    <div class="full_graph">
        {#if metric_to_graph.includes("Family")}
            <BinGraph binned_data={box_plot_stats_household_array} yScale={yScale}
                      xScale={xScaleHousehold} metric={metric_to_graph} 
                      bin_type={family_bins} data={data} text={["Family 1"]}/>
        {/if}
        {#if metric_to_graph.includes("Race")}
            <BinGraph binned_data={box_plot_stats_race_array} yScale={yScale}
                      xScale={xScaleRace} metric={metric_to_graph}
                      bin_type={race_bins} data={data} text={["Race 1"]}/>
        {/if}
        {#if metric_to_graph.includes("Elderly")}
            <BinGraph binned_data={box_plot_stats_elder_array} yScale={yScale}
                      xScale={xScaleElder} metric={metric_to_graph}
                      bin_type={elder_bins} data={data} text={["Elderly 1"]}/>
        {/if}
        {#if metric_to_graph.includes("Corporate")}
            <BinGraph binned_data={box_plot_stats_corp_array} yScale={yScale}
                      xScale={xScaleCorp} metric={metric_to_graph}
                      bin_type={corp_bins} data={data} text={["Corporate 1"]}/>
        {/if}        
    </div>

    <!-- <div class="simulate_winter_ban">
        <input class="winter_button" type="checkbox" 
        on:click={evt => checkedBox(evt)}
        name="is_clicked"
        value={true} bind:this={checkbox_pushed}>
        <label>Simulate Winter Ban</label>
        <section>
            <p>
                Click on the <em>"Simulate Winter Ban"</em> button to see how the eviction rate changes 
                with the removal of evictions during winter months. Additionally, you will see
                which demographics are most impacted by winter bans.
            </p>
        </section>
    </div> -->

    <div class="eviction_animation">
        {#if metric_to_graph.includes("Family")}
            <DotAnimation data={temp_data} text={"[insert text for Family Bin here]"}
            bins={family_bins} metric={metric_to_graph} />
        {/if}

        {#if metric_to_graph.includes("Race")}
            <DotAnimation data={temp_data} text={"[insert text for Race Bin here]"}
            bins={race_bins} metric={metric_to_graph} />
        {/if}

        {#if metric_to_graph.includes("Elder")}
            <DotAnimation data={temp_data} text={"[insert text for Elder Bin here]"}
            bins={elder_bins} metric={metric_to_graph} />
        {/if}

        {#if metric_to_graph.includes("Corporate")}
            <DotAnimation data={temp_data} text={"[insert text for Corporate Bin here]"}
            bins={corp_bins} metric={metric_to_graph} />
        {/if}
         
    </div>
</div>