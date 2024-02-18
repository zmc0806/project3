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
  const legendData = [
    { range: [0, 10], color: '#fee5d9' },
    { range: [10, 20], color: '#fcae91' },
    { range: [20, 30], color: '#fb6a4a' },
    { range: [30, 40], color: '#de2d26' },
    { range: [40, 50], color: '#a50f15' },
    // Add more ranges and colors as needed
  ];

  // Function to apply highlight effect
  function highlightRange(range) {
    geojsonData.features.forEach(feature => {
      feature.properties.opacity = (feature.properties.mortality >= range[0] && feature.properties.mortality <= range[1]) ? 1 : 0.2;
    });
  }

  // Function to reset highlight effect
  function resetHighlight() {
    geojsonData.features.forEach(feature => {
      feature.properties.opacity = 1;
    });
  }

  onMount(async () => {
    const csvData = await d3.csv('annual-mortality-rate-from-seasonal-influenza-ages-65.csv');
    geojsonData = await d3.json('custom.geo.json');

    mortalityRates = new Map(csvData.map(row => [row.Code, +row['rate over65']]));

    geojsonData.features.forEach(feature => {
      feature.properties.mortality = mortalityRates.get(feature.properties.adm0_a3) || 0;
      feature.properties.opacity = 1; // Add an opacity property
    });

    projection = d3.geoNaturalEarth1().fitSize([width, height], geojsonData);
    pathGenerator = d3.geoPath().projection(projection);

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
        style="opacity: {feature.properties.opacity}"
        on:mouseover="{(event) => showTooltip(event, feature)}"
        on:mouseout="{hideTooltip}"
      />
    {/each}
  {/if}
</svg>

<div class="legend">
  {#each legendData as legendItem}
    <div
      class="legend-range"
      style="background-color: {legendItem.color}"
      on:mouseover="{() => highlightRange(legendItem.range)}"
      on:mouseout="{resetHighlight}"
    ></div>
  {/each}
</div>

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
  .legend {
    display: flex;
    justify-content: center;
    margin-top: 10px;
  }
  .legend-range {
    width: 20px;
    height: 20px;
    display: inline-block;
    margin: 0 2px;
    cursor: pointer;
  }
</style>
