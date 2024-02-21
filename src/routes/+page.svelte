<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { marked } from 'marked';


  let markdown = `Rationale for Design Decisions:

Our visualization project aimed to present the annual mortality rates from seasonal influenza for individuals aged 65 and over in an intuitive and interactive manner.The design decisions were guided by the goal of maximizing clarity, accessibility, and informative value.

**Visual Encodings:**
- **Choropleth Map:** We chose a choropleth map to geographically represent mortality rates, allowing users to easily grasp spatial patterns and compare rates across countries. This method is particularly effective for displaying statistical data tied to geographical areas.

- **Adopting Color-Blind-Friendly Palettes:** Initially, we considered using a red-orange color scale for its intuitive representation of severity.     However, to accommodate users with color vision deficiencies, We chose to use high contrast colors to make it intuitive for the audience to see clearly, and to take care of the color blind. we selected a color-blind-friendly palette that maintains distinctiveness and clarity without relying on color differentiation that may pose challenges for color-blind users. This approach involved using shades and tones that are easily distinguishable by individuals with various types of color vision, ensuring that our visualization remains effective and inclusive.


- **Interactive Legend and Tooltips:** The interactive legend and tooltips enhance user engagement, providing precise data for hovered regions or legend values. This approach was chosen to offer detailed information on demand, reducing cognitive load and improving the user experience.


- **Fixed Display on Color Range Selection:** A new feature has been implemented to allow users to click on a specific color range within the legend. This action "locks in" the display to only show countries or regions within that selected mortality rate range. And A box will appear showing the mortality rate for each country in this range in ascending order. By doing so, users can focus on analyzing the data for countries that fall into specific mortality rate categories, enhancing the depth of exploration available within our visualization.For example, the mouse is placed on a country, the corresponding number and the corresponding country will be displayed with color, and then the legend click can have the table out, and the corresponding country will have color out.

**Alternatives Considered:**
- We considered using a linear color scale but found the square root scale more effectively differentiated the data for visual analysis.
- Interactive sliders for temporal data exploration were considered but ultimately not included due to the dataset focusing solely on the year 2011.   Future iterations might expand the temporal range for dynamic exploration.

### Overview of Development Process

**Team Collaboration:**

- **Mianchen Zhang** took on the critical roles of sourcing the GeoJSON file necessary for mapping and ensuring that the CSV data correctly matched the GeoJSON's ID system. This foundational work was essential for the accuracy and integrity of our visualization. Additionally, Mianchen was responsible for creating the initial visual representation, laying the groundwork for the project's interactive elements.
- **Yulin Chen** was tasked with implementing the remaining aspects of the visualization. Yulin's contributions were pivotal in advancing the project beyond its initial stages, focusing on enhancing interactivity, refining the visual encodings, and ensuring the user interface was both intuitive and engaging.
- This division of labor allowed our team to efficiently leverage individual strengths, ensuring a smooth development process.

**Challenges and Time Allocation:**
- **Data Preparation (30% of time):** Aligning the CSV data with the GeoJSON map codes and ensuring accuracy was a significant initial challenge.
- **Visualization Coding (50% of time):** Implementing the d3.js visual encodings, particularly the interactive elements like tooltips and the legend, took the bulk of development time. Fine-tuning the responsiveness and interactivity required extensive testing and adjustment.
- **Design (20% of time):** Designing an intuitive and accessible interface was crucial. Efforts were made to ensure the visualization was usable across various devices and screen sizes.

**Reflections:**

Future work could explore additional interactive features, such as temporal sliders or comparative analysis tools.

### Conclusion

Our visualization strives to provide an accessible and informative view of seasonal influenza mortality rates among the elderly in 2011. Through careful design and development decisions, we have created an interactive experience that we believe enhances understanding and engagement with the data. This project not only sheds light on important health metrics but also demonstrates the power of data visualization in communicating complex information effectively.`;
  let htmlContent;

  onMount(() => {
    htmlContent = marked(markdown);
  });

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
<div>{@html htmlContent}</div>
