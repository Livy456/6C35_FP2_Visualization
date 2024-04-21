<script>
    import * as d3 from "d3";
    // import binned_csv from '$static/' ;
    import { onMount } from "svelte";
    import {
        computePosition,
        autoPlacement,
        offset
    } from '@floating-ui/dom';
    import BinGraph from "./BinGraph.svelte";

    let data = [];
    let width = 800, height = 275; // changed the height of the graph from 600 to 450
    let yScale = d3.scaleLinear();
    let xScaleHousehold = d3.scaleBand();
    let xScaleRace = d3.scaleBand();
    let xScaleElder = d3.scaleBand();
    let xScaleCorp = d3.scaleBand();
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
    let data_black_avg;
    let data_white_avg;
    let data_latino_avg;
    let data_other_avg;
    let data_family_avg;
    let data_no_family_avg;
    let data_mixed_avg;
    let data_elderly_avg;
    let data_no_elderly_avg;
    let data_low_avg;
    let data_medium_avg;
    let data_high_avg;
    let black_avg_p = "The average rate at which black tenants get evicted is ${format(data_black_avg, '.2f')}";
    let white_avg_p = "The average rate at which white tenants get evicted is ${format(data_white_avg, '.2f')}";
    let latino_avg_p = "The average rate at which latino tenants get evicted is ${format(data_latino_avg, '.2f')}";
    let other_avg_p = "The average rate at which tenants with another race get evicted is ${format(data_other_avg, '.2f')}";
    let fam_avg_p = "The average rate at which tenants with family type household get evicted is ${format(data_family_avg, '.2f')}";
    let non_fam_avg_p = "The average rate at which tenants with non family type household get evicted is ${format(data_no_family_avg, '.2f')}";
    let mixed_avg_p = "The average rate at which tenants with mixed family type household get evicted is ${format(data_mixed_avg, '.2f')}";
    let no_elder_avg_p = "The average rate at which tenants with no elderly population in household get evicted is ${format(data_no_elderly_avg, '.2f')}";
    let elder_avg_p = "The average rate at which tenants with some elderly population in household get evicted is ${format(data_elderly_avg, '.2f')}";
    let low_avg_p = "The average rate at which tenants living in properties with low corporate ownership get evicted is ${format(data_low_avg, '.2f')}";
    let medium_avg_p = "The average rate at which tenants living in properties with medium corporate ownership get evicted is ${format(data_medium_avg, '.2f')}";
    let high_avg_p = "The average rate at which tenants living in properties with high corporate ownership get evicted is ${format(data_high_avg, '.2f')}";
    let race_avg_p = [black_avg_p, white_avg_p, latino_avg_p, other_avg_p];
    let family_avg_p = [fam_avg_p, mixed_avg_p, non_fam_avg_p];
    let elderly_avg_p = [elder_avg_p, no_elder_avg_p];
    let corp_avg_p = [low_avg_p, medium_avg_p, high_avg_p];
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
    
    // $: {
    //     d3.select(svg).call(d3.brush().on("start brush end", brushed));
    //     d3.select(svg).selectAll(".dots, .overlay ~ *").raise();
    // }
    
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
    $: data_family_avg = d3.mean(data_family, d => d.eviction_rate);
    $: data_no_family_avg = d3.mean(data_non_family, d => d.eviction_rate);
    $: data_mixed_avg = d3.mean(data_mixed, d => d.eviction_rate);
    $: box_plot_stats_household.family = calculate_box_plot(data_family);
    $: box_plot_stats_household.non_family = calculate_box_plot(data_non_family);
    $: box_plot_stats_household.mixed = calculate_box_plot(data_mixed);
    $: box_plot_stats_household_array = [box_plot_stats_household.family, box_plot_stats_household.non_family, box_plot_stats_household.mixed]

    // Computing box plots for race data
    $: data_black = data.filter((d) => d.majority_race === "Black");
    $: data_white = data.filter((d) => d.majority_race === "White");
    $: data_latino = data.filter((d) => d.majority_race === "Latino");
    $: data_other = data.filter((d) => d.majority_race === "Other");
    $: data_black_avg = d3.mean(data_black, d => d.eviction_rate);
    $: data_white_avg = d3.mean(data_white, d => d.eviction_rate);
    $: data_latino_avg = d3.mean(data_latino, d => d.eviction_rate);
    $: data_other_avg = d3.mean(data_other, d => d.eviction_rate);
    $: box_plot_stats_race.black = calculate_box_plot(data_black);
    $: box_plot_stats_race.white = calculate_box_plot(data_white);
    $: box_plot_stats_race.latino = calculate_box_plot(data_latino);
    $: box_plot_stats_race.other = calculate_box_plot(data_other);
    $: box_plot_stats_race_array = [box_plot_stats_race.black, box_plot_stats_race.white, box_plot_stats_race.latino, box_plot_stats_race.other];
    
    
    
    //DEBUGGING!!!!
    // $: console.log(box_plot_stats_race.black);
    // $: console.log(box_plot_stats_race.white.q1);




    // Computing box plots for elder or no elders
    $: data_elders = data.filter((d) => d.elder_bins === "some elders");
    $: data_no_elders = data.filter((d) => d.elder_bins === "no elders");
    $: data_elderly_avg = d3.mean(data_elders, d => d.eviction_rate);
    $: data_no_elderly_avg = d3.mean(data_no_elders, d => d.eviction_rate);
    $: box_plot_stats_elderly.some_elder = calculate_box_plot(data_elders);
    $: box_plot_stats_elderly.no_elder = calculate_box_plot(data_no_elders);
    $: box_plot_stats_elder_array = [box_plot_stats_elderly.some_elder, box_plot_stats_elderly.no_elder];

    // Computing box plots for corporate ownership
    $: data_corp_low = data.filter((d) => d.corp_bins === "low");
    $: data_corp_medium = data.filter((d) => d.corp_bins === "medium");
    $: data_corp_high = data.filter((d) => d.corp_bins === "high");
    $: data_low_avg = d3.mean(data_corp_low, d => d.eviction_rate);
    $: data_medium_avg = d3.mean(data_corp_medium, d => d.eviction_rate);
    $: data_high_avg = d3.mean(data_corp_high, d => d.eviction_rate);
    $: box_plot_stats_corp.low = calculate_box_plot(data_corp_low);
    $: box_plot_stats_corp.medium = calculate_box_plot(data_corp_medium);
    $: box_plot_stats_corp.high = calculate_box_plot(data_corp_high);
    $: box_plot_stats_corp_array = [box_plot_stats_corp.low, box_plot_stats_corp.medium, box_plot_stats_corp.high];
    
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
            {console.log(box_plot_stats_household_array)}
            {console.log(box_plot_stats_household_array[0].q1)}

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
</div>