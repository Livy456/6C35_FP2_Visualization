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
    export let yScale = d3.scaleLinear();

    let width = 850, height = 275; // changed the height of the graph from 600 to 450
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
    
    $: {
        d3.select(svg).call(d3.brush().on("start brush end", brushed));
        d3.select(svg).selectAll(".dots, .overlay ~ *").raise();
    }
    
    $: selectedEvictions = brushedSelection ? data.filter(isDataSelected) : [];    
    $: hasSelection = brushedSelection && selectedEvictions.length > 0;
    $: selectedLines = (hasSelection ? selectedEviction: data).flatMap(d => d.mhi);
    
    function compute_cx(d)
    {
        if(metric.includes("Family"))
        {
            return xScale(d.family_bins) + xScale.bandwidth() / 3.5 + Math.random()*80;
        }

        if(metric.includes("Elder"))
        {
            return xScale(d.elder_bins) + xScale.bandwidth() / 3.5 + Math.random()*80;
        }

        if(metric.includes("Race"))
        {
            return xScale(d.majority_race) + xScale.bandwidth() / 3.5 + Math.random()*80;
        }
        if(metric.includes("Corporate"))
        {
            return xScale(d.corp_bins) + xScale.bandwidth() / 3.5 + Math.random()*80;
        }
    }

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

    function brushed (evt)
    {
        brushedSelection = evt.selection;
    }

    function isDataSelected(dot, metric)
    {
        if(!brushedSelection)
        {
            return false;
        }

        // get points of brushed selection box
        let top_left = {x: brushedSelection[0][0], y: brushedSelection[0][1]};
        let bottom_right = {x: brushedSelection[1][0], y: brushedSelection[1][1]};
        let data_y_coord = yScale(dot.eviction_rate);

        if (metric.includes("Family Type"))
        {
            // account for the x shift
            // gets coordinate of commit data point
            let data_x_coord = xScale(dot.family_bins) + xScale.bandwidth() / 3.5;
            // checks if commit data point is within selected brush region
            if(data_x_coord >= top_left.x && data_x_coord <= bottom_right.x && 
            data_y_coord >= top_left.y && data_y_coord <= bottom_right.y)
            {
                console.log("I'm a brushed family data!!");
                return true;
            }
        }

        else if (metric.includes("Race"))
        {
            // account for the x shift
            // gets coordinate of commit data point
            let data_x_coord = xScale(dot.majority_race) + xScale.bandwidth() / 5.5;
            // checks if commit data point is within selected brush region
            if(data_x_coord >= top_left.x && data_x_coord <= bottom_right.x && 
            data_y_coord >= top_left.y && data_y_coord <= bottom_right.y)
            {
                console.log("I'm a brushed family data!!");
                return true;
            }
        }

        else if (metric.includes("Elder"))
        {
            // account for the x shift
            // gets coordinate of commit data point
            let data_x_coord = xScale(dot.elder_bins) + xScale.bandwidth() / 3.5;
            // checks if commit data point is within selected brush region
            if(data_x_coord >= top_left.x && data_x_coord <= bottom_right.x && 
            data_y_coord >= top_left.y && data_y_coord <= bottom_right.y)
            {
                console.log("I'm a brushed family data!!");
                return true;
            }
        }

        else if (metric.includes("Corporate"))
        {
            // account for the x shift
            // gets coordinate of commit data point
            let data_x_coord = xScale(dot.corp_bins) + xScale.bandwidth() / 4.5;
            // checks if commit data point is within selected brush region
            if(data_x_coord >= top_left.x && data_x_coord <= bottom_right.x && 
            data_y_coord >= top_left.y && data_y_coord <= bottom_right.y)
            {
                console.log("I'm a brushed family data!!");
                return true;
            }
        } 

        return false;        
    }
    
</script>

<style>
div.full_graph{
    display: subgrid;
    border: rgb(122, 115, 115);
    border-style: solid;
    border-radius: 2px;
    padding-left: 2fr;
}

p{
    display: inline;
}
</style>
<div class="full_graph">
    {#if metric.includes("Race")}
        <h2 style="margin-top: 3rem"> Number of Evictions vs Race</h2>
    {/if}

    {#if metric.includes("Family")}
        <h2 style="margin-top: 3rem"> Number of Evictions vs Family Type</h2>
    {/if}

    {#if metric.includes("Elderly")}
        <h2 style="margin-top: 3rem"> Number of Evictions vs Elderly Population</h2>
    {/if}

    {#if metric.includes("Corporate")}
        <h2 style="margin-top: 3rem"> Number of Evictions vs Corporate Ownership</h2>
    {/if}
    
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
    
        <dt>Family Type</dt>
        <dd> { hoveredEviction.family_bins } </dd>
    
        <dt>Elderly Population</dt>
        <dd> { hoveredEviction.elder_bins } </dd>

        <dt>Corporate Ownership</dt>
        <dd> { hoveredEviction.corp_bins } </dd>        
    </dl>
    <svg viewBox="0 0 {width} {height}" bind:this={svg}>
        <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
        <g class="gridlines" transform="translate({usableArea.left}, 0)" bind:this={yAxisGridlines} />
        <g transform="translate({usableArea.top})" bind:this={yAxis}/>
        
        <g class="dots">
        <!-- MAKE SURE TO SHIFT ALL X VALUES BY + xScale.bandwidth() / 2 -->

        {#each binned_data as bin, index}
            {d3.select(xAxis).call(d3.axisBottom(xScale))}
            <line 
                x1={ xScale(bin_type[index]) + xScale.bandwidth() / 2 }
                x2={ xScale(bin_type[index]) + xScale.bandwidth() / 2 }
                y1={ yScale(bin.min) }
                y2={ yScale(bin.max) }
                stroke="black"
                width=40
            />

            <line
                x1={ xScale(bin_type[index]) - boxWidth/2 + xScale.bandwidth() / 2 }
                x2={ xScale(bin_type[index]) + boxWidth/2 + xScale.bandwidth() / 2 }
                y1={ yScale(bin.q2) }
                y2={ yScale(bin.q2) }
                stroke="black"
                width=80
            />
            
            <rect
                x={ xScale(bin_type[index]) - boxWidth/2 + xScale.bandwidth() / 2}
                y={ yScale(bin.q3) } 
                width={boxWidth}
                height={yScale(bin.q1) - yScale(bin.q3)}
                stroke="black"
                fill="red"
                fill-opacity=0.5
            />        
        {/each}
        
        <!-- MIGHT NEED TO PUT IN THE SPECIFIC BIN TYPE AND USE IF STATEMENTS -->
        {#each data as d, index}           
            <circle 
                class:selected={isDataSelected(d, metric)}
                cx= {compute_cx(d)}
                cy={ yScale(d.eviction_rate) }
                r='3'
                fill="black"
                fill-opacity= 60%
                stroke="white"
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

    <dl class="metric_info">
        {#each binned_info as info}
            {#if metric.includes("Black")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("White")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("Latino")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("Other")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("FamilyType")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("Non-Fam")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("Mixed")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("No Elder")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("Some Elder")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("Low")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("Medium")}
                <p>{info}</p>
            {/if}
            {#if metric.includes("High")}
                <p>{info}</p>
            {/if}
        {/each}
    </dl>
</div>