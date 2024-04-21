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

</script>
<style>
.description{
    border: 1px solid black;
    border-radius:3px;
    padding:10px;
    box-shadow:5px 5px 5px grey;
    margin-right:10px;

}

.legend{

    display: grid;
    grid-template-rows: 10fr 10fr;
    padding:50px;
    
    label{
        grid-row:1;
    }
    div{
        grid-row:2;
        border-radius:50px;
        background-color:red;
        border:4px solid black;
        width:10px;
        height:10px;
    }
}

</style>
<dl class="animation_container">
    <section class="legend">
        {#each bins as bin}
            <label>{bin}</label>
            <div></div>
        {/each}
    </section>
    
    <section class="description">
        <p>In pulvinar ante enim, id gravida velit vulputate ut. Ut in dui vitae ex 
            consequat tempor id eu turpis. Fusce in orci dignissim, lacinia metus vel, 
            euismod neque. Nunc sem diam, placerat at diam in, fringilla auctor odio. 
            Phasellus dolor justo, condimentum at libero sodales, fringilla accumsan leo. 
            Nunc venenatis justo eget congue lacinia. Nunc aliquam, lacus vel pretium dictum, 
            turpis eros pulvinar sem, et placerat leo magna aliquam purus. Mauris sit amet 
            massa faucibus, suscipit mauris id, congue augue. Fusce vehicula vitae felis 
            at volutpat. Fusce aliquam lectus lorem, sed porta erat fermentum id. Praesent 
            a ante nisl. Suspendisse ac nulla eu lectus tincidunt feugiat.
        </p>
    </section>
</dl>

