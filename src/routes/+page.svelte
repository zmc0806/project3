<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let mortalityRates = new Map(); 
  let geojsonData;
  let projection;
  let pathGenerator;
  let colorScale;
  let tooltip = { x: 0, y: 0, text: '', visible: false };
  let maxMortality = 102.75;
  let legendValues = [];
  let hoveredMortalityRate = null;
  let hoveredCountryMortality = null;

  let selectedRange = null;
  function selectRange(value) {
  if (selectedRange === value) {
    // Toggle off if the same value is clicked
    selectedRange = null;
    sortedCountries = [];
  } else {
    selectedRange = value;
    // Update sortedCountries based on the new selected range
    sortedCountries = geojsonData.features
      .filter(feature => feature.properties.mortality >= selectedRange && feature.properties.mortality < selectedRange + maxMortality / 9)
      .map(feature => ({ name: feature.properties.name, mortality: feature.properties.mortality }))
      .sort((a, b) => b.mortality - a.mortality);
  }
}




  function highlightCountries(rate) {
    hoveredMortalityRate = rate;
  }
  function resetHighlight() {
    hoveredMortalityRate = null;
  }
  
  const width = 1200;
  const height = 760;
  let selectedCountry = null;

  function highlightLegendFromCountry(event, feature) {
  hoveredCountryMortality = feature.properties.mortality; // Set the hovered country's mortality rate
  showTooltip(event, feature); // Continue showing the tooltip
}

function resetLegendHighlight() {
  hoveredCountryMortality = null; // Reset the hovered country's mortality rate
  hideTooltip(); // Hide the tooltip
}


  function highlightBorder(event, feature) {
    selectedCountry = feature.properties.adm0_a3;
    showTooltip(event, feature);
  }
  function resetBorder() {
    selectedCountry = null;
    hideTooltip();
  }

  // Function to show the tooltip
  function showTooltip(event, d) {
    const mortalityRate = d.properties.mortality.toFixed(2); // Assuming this is a number
    tooltip = {
      x: event.pageX,
      y: event.pageY,
      content: {
        title: d.properties.name,
        year: '2011', // You will need to get the actual year from your data
        mortalityRate: mortalityRate
      },
      visible: true
    };
  }

  // Function to hide the tooltip
  function hideTooltip() {
    tooltip.visible = false;
  }

  let sortedCountries = [];



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
    maxMortality = Math.max(...mortalityRates.values());
    colorScale = d3.scaleSequentialSqrt(d3.interpolateRgbBasisClosed(["#2c7bb6", "#abd9e9", "#ffffbf", "#fdae61", "#d7191c"])) .domain([0, maxMortality])

    legendValues = Array.from({length: 10}, (_, i) => i * maxMortality / 9);
  
    
  });
</script>
<svg width="{width}" height="{height}" style="background-color: #eaeaea;">
  {#if geojsonData && pathGenerator}
  
  {#each geojsonData.features as feature}
  <path
    d="{pathGenerator(feature)}"
    fill="{selectedRange !== null ? (feature.properties.mortality >= selectedRange && feature.properties.mortality < selectedRange + maxMortality/9 ? colorScale(feature.properties.mortality) : '#ddd') : (hoveredMortalityRate === null || (feature.properties.mortality >= hoveredMortalityRate && feature.properties.mortality < hoveredMortalityRate + maxMortality/9) ? colorScale(feature.properties.mortality) : '#ddd')}"
    stroke="{selectedRange !== null ? 'none' : (hoveredMortalityRate !== null && feature.properties.mortality >= hoveredMortalityRate && feature.properties.mortality < hoveredMortalityRate + maxMortality/9 ? 'black' : '#fff')}"
    stroke-width="0.5"
    stroke-opacity="2"
    on:mouseover="{(event) => highlightBorder(event, feature)}"
    on:mouseover="{(event) => highlightLegendFromCountry(event, feature)}"
    on:mouseout="{resetBorder}"
    on:mouseout="{resetLegendHighlight}"
  />
{/each}


    <text x="55%" y="40" text-anchor="middle" font-size="20px" fill="#433">
      Respiratory death rate from seasonal influenza, age 65+, 2011
    </text>

    <g class="legend" transform="translate({(width - legendValues.length * 40) / 2}, {height - 60})">
      {#each legendValues as value, index}
      <rect
      x={index * 40}
      y={0}
      width={300 / legendValues.length}
      height={50}
      fill="{colorScale(value)}"
      stroke="{selectedRange === value ? 'black' : (hoveredCountryMortality !== null && hoveredCountryMortality >= value && hoveredCountryMortality < value + maxMortality / 9 ? 'black' : 'none')}"
      stroke-width="{selectedRange === value || (hoveredCountryMortality !== null && hoveredCountryMortality >= value && hoveredCountryMortality < value + maxMortality / 9) ? 2 : 0}"
      on:click="{() => selectRange(value)}"
      on:mouseover="{() => highlightCountries(value)}"
      on:mouseout="{resetHighlight}"
    />
        <text
          x={(index * 40-10) + (300 / legendValues.length / 2)} 
          y={60}
          font-size="12px"
          text-anchor="middle"
          on:mouseover="{() => highlightCountries(value)}"
          on:mouseout="{resetHighlight}"
        >
          {Math.round(value)}
        </text>
      {/each}
    </g>    
  {/if}
</svg>

{#if tooltip.visible}
{#if selectedRange !== null}
  <div class="country-table">
    <table>
      <tr>
        <th>Country</th>
        <th>Mortality Rate</th>
      </tr>
      {#each sortedCountries as country}
        <tr>
          <td>{country.name}</td>
          <td>{country.mortality.toFixed(2)}</td>
        </tr>
      {/each}
    </table>
  </div>
{/if}


  <div class="tooltip" style="left: {tooltip.x}px; top: {tooltip.y}px;">
    <div class="tooltip-header">{tooltip.content.title}</div>
    <div class="tooltip-body">
      <div class="tooltip-year">{tooltip.content.year}</div>
      <div class="tooltip-mortality">deaths per 100,000</div>
      <div class="tooltip-rate">{tooltip.content.mortalityRate}</div>
    </div>
  </div>
{/if}

<style>
  .tooltip {
    position: absolute;
    background-color: #fff;
    padding: 10px;
    border-radius: 2px;
    border: 1px solid #ccc;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    pointer-events: none;
    transition: opacity 0.3s;
    opacity: 0.9;
    z-index: 10;
  }
  .tooltip-header {
    font-weight: bold;
    color: #060606;
  }
  .tooltip-body {
    margin-top: 5px;
  }
  .tooltip-year {
    color: #4a4949;
  }
  .tooltip-mortality {
    color: #7c7474;
  }
  .tooltip-rate {
    font-weight: bold;
    color: #ed0000;
  }

  path:hover {
    stroke: rgb(41, 40, 40);
    stroke-width: 1.25;
    stroke-opacity: 1;
  }
  .legend text {
    fill: #000;
    text-anchor: start;
  } 
  .country-table {
  position: absolute;
  left: 10px;
  top: 0px;
  background: white;
  border: 1px solid #ccc;
  padding: 10px;

  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  max-height: 100%;
  overflow-y: auto;
}

.country-table table {
  width: 100%;
  border-collapse: collapse;
}

.country-table th, .country-table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.country-table th {
  background-color: #f2f2f2;
}

</style>
