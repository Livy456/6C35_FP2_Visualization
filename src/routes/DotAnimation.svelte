<script>
    import * as d3 from "d3";
    import {
        computePosition,
        autoPlacement,
        offset
    } from '@floating-ui/dom';
    export let text = "";
    export let data = [];
    export let bins = [];
    export let metric = "";

    let temp_bins = ["Really warm", "Warm", "Cold", "Really Cold"];

</script>
<style>
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
        background-color:red;
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
        background-color:blue;
        opacity:20%;
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

</style>
<dl class="animation_container">
    <section class="bin_legend">
        {#each bins as bin}
            <label>{bin}</label>
            <div></div>
        {/each}
    </section>

    <div class="entire_visualization">
        <section class="visual">
            <svg> 
        </section>
        <section class="temp_legend">
            {#each bins as bin, i}
                <!-- <dt>{bin}</dt> -->
                <div class="bin_circles"/>
    
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

