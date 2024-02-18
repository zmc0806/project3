<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let mortalityRates;
  let geojsonData;
  let projection;
  let pathGenerator;
  let colorScale;
  let tooltip = { x: 0, y: 0, text: '', visible: false };

  const width = 1160;
  const height = 800;

  // Function to show the tooltip
  function showTooltip(event, d) {
    tooltip = {
      x: event.pageX,
      y: event.pageY,
      text: `${d.properties.name}: ${d.properties.mortality.toFixed(2)} deaths per 100,000`,
      visible: true
    };
  }

  // Function to hide the tooltip
  function hideTooltip() {
    tooltip.visible = false;
  }

  onMount(async () => {
    const csvData = await d3.csv('/annual-mortality-rate-from-seasonal-influenza-ages-65.csv');
    geojsonData = await d3.json('/custom.geo.json');

    mortalityRates = new Map(csvData.map(row => [row.Code, +row['rate over65']]));

    // Assign mortality rate to geojson based on country code
    geojsonData.features.forEach(feature => {
      feature.properties.mortality = mortalityRates.get(feature.properties.adm0_a3) || 0;
    });

    // Create a projection and path generator
    projection = d3.geoNaturalEarth1().fitSize([width, height], geojsonData);
    pathGenerator = d3.geoPath().projection(projection);

    // Create a color scale
    const maxMortality = Math.max(...csvData.map(row => +row['rate over65']));
    colorScale = d3.scaleSequential([0, maxMortality], d3.interpolateReds);
  });
</script>

<svg width="{width}" height="{height}">
  {#if geojsonData && pathGenerator}
    {#each geojsonData.features as feature}
      <path
        d="{pathGenerator(feature)}"
        fill="{colorScale(feature.properties.mortality)}"
        stroke="#fff"
        on:mouseover="{(event) => showTooltip(event, feature)}"
        on:mouseout="{hideTooltip}"
      />
    {/each}
  {/if}
</svg>

{#if tooltip.visible}
  <div
    class="tooltip"
    style="position: absolute; left: {tooltip.x}px; top: {tooltip.y}px; pointer-events: none; background-color: #fff; padding: 5px; border: 1px solid #ccc;"
  >
    {tooltip.text}
  </div>
{/if}

<style>
  .tooltip {
    /* Add additional styles to your tooltip here */
  }
</style>












