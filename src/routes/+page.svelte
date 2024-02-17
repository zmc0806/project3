<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let mortalityRates;
  let geojsonData;
  let projection;
  let pathGenerator;
  let colorScale;

  const width = 960;
  const height = 600;

  onMount(async () => {
    const csvData = await d3.csv('annual-mortality-rate-from-seasonal-influenza-ages-65.csv');
    geojsonData = await d3.json('custom.geo.json');

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
      />
    {/each}
  {/if}
</svg>

