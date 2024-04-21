<script>
    import * as d3 from "d3";
    import { onMount } from "svelte";
    import BinGraph from "./BinGraph.svelte";
    import MapVisual from "./MapVisual.svelte";

    let data = [];
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
    let data_mixed;
    let data_family;
    let data_non_family;
    let box_plot_stats_household_array = [];
    let box_plot_stats_race_array = [];
    let box_plot_stats_elder_array = [];
    let box_plot_stats_corp_array = [];
    let box_plot_stats_household = {mixed: {}, non_family: {}, family: {}};
    let box_plot_stats_race = {black: {}, white: {}, latino: {}, other: {}};
    let box_plot_stats_elderly = {some_elder: {}, no_elder: {}};
    let box_plot_stats_corp = {low: {}, medium: {}, high: {}};
    let metric_to_graph = "Family Type";
        
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
        data = await d3.csv("binned_data.csv", row=> ({
            ...row,
            mhi: Number(row.mhi),
            eviction_rate: Number(row.eviction_rate),
            family_bins: String(row.family_bins)
        }));

    });

    $: data = data.filter((d) => d.family_bins !== '').filter((d) => d.eviction_rate < 1).filter((d) => d.mhi > 0);
    $: selectedEvictions = brushedSelection ? data.filter(isDataSelected) : [];    
    $: hasSelection = brushedSelection && selectedEvictions.length > 0;
    $: selectedLines = (hasSelection ? selectedEviction: data).flatMap(d => d.mhi);
    $: yScale = yScale.domain([0, 0.2]).range([usableArea.bottom, usableArea.top]); // might need to switch values currently domain = [0, height], range = [0, 24] 
    
    // Setting X scale
    $: xScaleHousehold = xScaleHousehold.domain(family_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);    
    $: xScaleRace = xScaleRace.domain(race_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);
    $: xScaleElder = xScaleElder.domain(elder_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);
    $: xScaleCorp = xScaleCorp.domain(corp_bins).range( [usableArea.left, usableArea.right] ).padding(0.2);

    // Computing box plots for family bin data
    $: data_mixed = data.filter((d) => d.family_bins === "mixed");
    $: data_family = data.filter((d) => d.family_bins === "family");
    $: data_non_family = data.filter((d) => d.family_bins === "non-family");
    $: box_plot_stats_household.family = calculate_box_plot(data_family);
    $: box_plot_stats_household.non_family = calculate_box_plot(data_non_family);
    $: box_plot_stats_household.mixed = calculate_box_plot(data_mixed);
    $: box_plot_stats_household_array = [box_plot_stats_household.family, box_plot_stats_household.non_family, box_plot_stats_household.mixed]

    // Computing box plots for race data
    $: data_black = data.filter((d) => d.majority_race === "Black");
    $: data_white = data.filter((d) => d.majority_race === "White");
    $: data_latino = data.filter((d) => d.majority_race === "Latino");
    $: data_other = data.filter((d) => d.majority_race === "Other");
    $: box_plot_stats_race.black = calculate_box_plot(data_black);
    $: box_plot_stats_race.white = calculate_box_plot(data_white);
    $: box_plot_stats_race.latino = calculate_box_plot(data_latino);
    $: box_plot_stats_race.other = calculate_box_plot(data_other);
    $: box_plot_stats_race_array = [box_plot_stats_race.black, box_plot_stats_race.white, box_plot_stats_race.latino, box_plot_stats_race.other];

    // Computing box plots for elder or no elders
    $: data_elders = data.filter((d) => d.elder_bins === "some elders");
    $: data_no_elders = data.filter((d) => d.elder_bins === "no elders");
    $: box_plot_stats_elderly.some_elder = calculate_box_plot(data_elders);
    $: box_plot_stats_elderly.no_elder = calculate_box_plot(data_no_elders);
    $: box_plot_stats_elder_array = [box_plot_stats_elderly.some_elder, box_plot_stats_elderly.no_elder];

    // Computing box plots for corporate ownership
    $: data_corp_low = data.filter((d) => d.corp_bins === "low");
    $: data_corp_medium = data.filter((d) => d.corp_bins === "medium");
    $: data_corp_high = data.filter((d) => d.corp_bins === "high");
    $: box_plot_stats_corp.low = calculate_box_plot(data_corp_low);
    $: box_plot_stats_corp.medium = calculate_box_plot(data_corp_medium);
    $: box_plot_stats_corp.high = calculate_box_plot(data_corp_high);
    $: box_plot_stats_corp_array = [box_plot_stats_corp.low, box_plot_stats_corp.medium, box_plot_stats_corp.high];
    
    function filterData()
    {
        
    }

    function calculate_box_plot(binned_data)
    {   
        let eviction_rate_array = [];

        for (let d of binned_data)
        {
            // console.log(d);
            // if (d.eviction_rate)
            // {
            eviction_rate_array.push(d.eviction_rate);
            // }
        }

        // console.log(eviction_rate_array);

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

</script>

<h3 class="meta">
    <strong style="color: blue;">Interactive Voices:</strong>
    Mapping the Movement for Eviction Reform and Advocacy
</h3>

<div class="full_page">

    <div class="narrative1">
        <section>
            <h2>Goal</h2>
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
    
    <div class="metric_selection">
        <h4>Race</h4>
        <dl class="race_selection">
            <input type="button" value="White" on:click={ function() {metric_to_graph= "Race White"} }>
            <input type="button" value="Black" on:click={ function() {metric_to_graph= "Race Black"} }>
            <input type="button" value="Latino" on:click={ function() {metric_to_graph= "Race Latino"} }>
            <input type="button" value="Other" on:click={ function() {metric_to_graph= "Race Other"} }>
        </dl>
        
        <h4>Family Type</h4>
        <dl class="family_selection">
            <input type="button" value="Mixed" on:click={ function() {metric_to_graph= "Family Type Mixed"} }>
            <input type="button" value="Non Family" on:click={ function() {metric_to_graph= "Family Type Non-Fam"} }>
            <input class="bottom_row" type="button" value="Family" on:click={ function() {metric_to_graph= "Family Type FamilyType"} }>
        </dl>
        
        <h4>Elderly Population</h4>
        <dl class="elderly_selection">
            <input type="button" value="No Elderly" on:click={ function() {metric_to_graph= "No Elderly"} }>
            <input type="button" value="Some Elderly" on:click={ function() {metric_to_graph= "Some Elderly"} }>
        </dl>
        
        <h4>Corporate Ownership</h4>
        <dl class="corporate_selection">
            <input type="button" value="Low Corporate Ownership" on:click={ function() {metric_to_graph= "Corporate Ownership Low"} }>
            <input type="button" value="Medium Corporate Ownership" on:click={ function() {metric_to_graph= "Corporate Ownership Medium"} }>
            <input type="button" value="High Corporate Ownership" on:click={ function() {metric_to_graph= "Corporate Ownership High"} }>
        </dl>
    </div>

    <div class="full_graph">
        {#if metric_to_graph.includes("Family")}
            <BinGraph binned_data={box_plot_stats_household_array} yScale={yScale}
                      xScale={xScaleHousehold} metric={metric_to_graph} 
                      bin_type={family_bins} data={data} binned_info={"Family Metric"}/>
        {/if}
        {#if metric_to_graph.includes("Race")}
            <BinGraph binned_data={box_plot_stats_race_array} yScale={yScale}
                      xScale={xScaleRace} metric={metric_to_graph}
                      bin_type={race_bins} data={data} binned_info={"Race Metric"}/>
        {/if}
        {#if metric_to_graph.includes("Elderly")}
            <BinGraph binned_data={box_plot_stats_elder_array} yScale={yScale}
                      xScale={xScaleElder} metric={metric_to_graph}
                      bin_type={elder_bins} data={data} binned_info={"Elderly Metric"}/>
        {/if}
        {#if metric_to_graph.includes("Corporate")}
            <BinGraph binned_data={box_plot_stats_corp_array} yScale={yScale}
                      xScale={xScaleCorp} metric={metric_to_graph}
                      bin_type={corp_bins} data={data} binned_info={"Corporate Metric"}/>
        {/if}        
    </div>

    <div class="simulate_winter_ban">
        <input class="winter_button" type="button" value="Simulate Winter Ban">
        <section>
            <p>
                Click on the <em>"Simulate Winter Ban"</em> button to see how the eviction rate changes 
                with the removal of evictions during winter months. Additionally, you will see
                which demographics are most impacted by winter bans.
            </p>
        </section>
    </div>

    <div class="map_visualization">
        <MapVisual></MapVisual>
    </div>
</div>