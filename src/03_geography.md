---
style: text-style.css
---

<!-- Questions and Findings - For each question:
Clear question statement
Polished D3 visualization
Analysis and interpretation
Insights and implications -->

# Geographic Analysis

```js
// Load spatial files plus the aggregates generated from preprocessing notebooks
const boroughBoundaries = await FileAttachment("nyc-borough-boundaries.geojson").json();
const boroughCounts = (await FileAttachment("call_volume_by_borough.csv").csv({typed: true}))
  .filter(d => d.boro_nm && d.boro_nm !== "(Null)");
const precinctCounts = await FileAttachment("call_volume_by_precinct.csv").csv({typed: true});
const precinctProfiles = (await FileAttachment("precinct_response_profile.csv").csv({typed: true}))
  .filter(d => d.nypd_pct_cd > 0 && d.boro_nm && d.boro_nm !== "(null)" && d.boro_nm !== "(Null)");
const boroughProfiles = (await FileAttachment("borough_response_profile.csv").csv({typed: true}))
  .filter(d => d.boro_nm && d.boro_nm !== "(Null)");
const boroughSeries = (await FileAttachment("borough_call_timeseries.csv").csv({typed: true}))
  .filter(d => d.boro_nm && d.boro_nm !== "(Null)");
const callCategories = Array.from(new Set(boroughCounts.map(d => d.category)));
const boroughNames = Array.from(new Set(boroughProfiles.map(d => d.boro_nm))).sort(d3.ascending);
```

## How does 911 call volume relate to call duration and response delays across boroughs?

### Citywide load vs. service speed

- This chart compares call volume (x-axis) with median call duration (y-axis), while bubble size represents arrival delay
- Use the controls below to switch between call categories or highlight specific boroughs to see how service speed varies across the city

```js
const scatterFocus = view(Inputs.radio(["All Categories", ...callCategories], {
  label: "Call set",
  value: "All Categories"
}));
```

```js
const scatterBorough = view(Inputs.select(["Citywide", ...boroughNames], {
  label: "Borough",
  value: "Citywide"
}));
```

```js
// Filter for whichever dataset is active; "All" shows both categories together
const scatterData = scatterFocus === "All Categories"
  ? boroughProfiles
  : boroughProfiles.filter(d => d.category === scatterFocus);
const selectedBorough = scatterBorough === "Citywide" ? null : scatterBorough;
const scatterWidth = 960;
const scatterHeight = 460;
const scatterMargin = {top: 60, right: 160, bottom: 55, left: 80};
const scatterX = d3.scaleLinear()
  .domain([0, d3.max(scatterData, d => d.call_count) * 1.05]).nice()
  .range([scatterMargin.left, scatterWidth - scatterMargin.right]);
const scatterY = d3.scaleLinear()
  .domain([0, d3.max(scatterData, d => d.median_call_duration) * 1.15]).nice()
  .range([scatterHeight - scatterMargin.bottom, scatterMargin.top]);
const arrivalExtent = d3.extent(scatterData, d => d.median_arrival_delay);
const scatterR = d3.scaleSqrt()
  .domain([Math.max(1, arrivalExtent[0]), arrivalExtent[1] || Math.max(1, arrivalExtent[0]) + 1])
  .range([6, 24]);
const scatterColor = d3.scaleOrdinal()
  .domain(callCategories)
  .range(["#0f6cbd", "#d1495b"]);

// Scaffold the SVG with margins and backgrounds
const scatterSvg = d3.create("svg")
  .attr("viewBox", [0, 0, scatterWidth, scatterHeight])
  .attr("width", scatterWidth)
  .attr("height", scatterHeight)
  .style("max-width", "100%")
  .style("height", "auto")
  .style("background", "#dfdfd6");

const scatterInnerWidth = scatterWidth - scatterMargin.left - scatterMargin.right;
const scatterInnerHeight = scatterHeight - scatterMargin.top - scatterMargin.bottom;

// Horizontal grid lines
scatterSvg.append("g")
  .attr("transform", `translate(0, ${scatterHeight - scatterMargin.bottom})`)
  .call(d3.axisBottom(scatterX).ticks(5, "~s").tickSize(-scatterInnerHeight).tickFormat(() => ""))
  .selectAll("line")
  .attr("stroke", "#bfbfbf")
  .attr("stroke-dasharray", "3,3");

// Vertical grid lines
scatterSvg.append("g")
  .attr("transform", `translate(${scatterMargin.left},0)`)
  .call(d3.axisLeft(scatterY).ticks(5).tickSize(-scatterInnerWidth).tickFormat(() => ""))
  .selectAll("line")
  .attr("stroke", "#bfbfbf")
  .attr("stroke-dasharray", "3,3");

// Solid x-axis baseline
scatterSvg.append("line")
  .attr("x1", scatterMargin.left)
  .attr("x2", scatterWidth - scatterMargin.right)
  .attr("y1", scatterHeight - scatterMargin.bottom)
  .attr("y2", scatterHeight - scatterMargin.bottom)
  .attr("stroke", "#000")
  .attr("stroke-width", 1.4);

// Solid y-axis baseline
scatterSvg.append("line")
  .attr("x1", scatterMargin.left)
  .attr("x2", scatterMargin.left)
  .attr("y1", scatterMargin.top)
  .attr("y2", scatterHeight - scatterMargin.bottom)
  .attr("stroke", "#000")
  .attr("stroke-width", 1.4);

scatterSvg.append("g")
  .attr("transform", `translate(0, ${scatterHeight - scatterMargin.bottom})`)
  .call(d3.axisBottom(scatterX).ticks(5, "~s"))
  .selectAll("text")
  .style("fill", "#000");

scatterSvg.append("text")
  .attr("x", scatterWidth / 2)
  .attr("y", scatterHeight - 10)
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .text("Call volume (2024)");

scatterSvg.append("g")
  .attr("transform", `translate(${scatterMargin.left},0)`)
  .call(d3.axisLeft(scatterY))
  .selectAll("text")
  .style("fill", "#000");

scatterSvg.append("text")
  .attr("x", -scatterHeight / 2)
  .attr("y", 20)
  .attr("transform", "rotate(-90)")
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .text("Median call duration (minutes)");

// Plot each borough as a labeled dot
const nodes = scatterSvg.append("g")
  .selectAll("g.node")
  .data(scatterData)
  .join("g")
  .attr("transform", d => `translate(${scatterX(d.call_count)}, ${scatterY(d.median_call_duration)})`);

nodes.append("circle")
  .attr("r", d => scatterR(d.median_arrival_delay))
  .attr("fill", d => scatterFocus === "All Categories" ? scatterColor(d.category) : d.category === "Confirmed Crime" ? "#0f6cbd" : "#d1495b")
  .attr("opacity", d => selectedBorough && d.boro_nm !== selectedBorough ? 0.3 : 0.95)
  .attr("stroke", d => d.boro_nm === selectedBorough ? "#000" : "#5d768d")
  .attr("stroke-width", d => d.boro_nm === selectedBorough ? 2 : 1)
  .append("title")
  .text(d => `${d.boro_nm} · ${d.category}
${d.call_count.toLocaleString()} calls
Median duration: ${d.median_call_duration} min
Median arrival delay: ${d.median_arrival_delay} min`);

nodes.append("text")
  .text(d => d.boro_nm)
  .attr("dx", d => {
    if (d.boro_nm === "Staten Island" && d.category === "Potential Crime") return 10;
    if (d.boro_nm === "Queens" && d.category === "Confirmed Crime") return -15;
    return 0;
  })
  .attr("dy", d => {
    const base = -scatterR(d.median_arrival_delay) - 6;
    if (d.boro_nm === "Staten Island" && d.category === "Confirmed Crime") return base - 2;
    if (d.boro_nm === "Staten Island" && d.category === "Potential Crime") return base + 2;
    if (d.boro_nm === "Queens" && d.category === "Confirmed Crime") return base + 16;
    return base;
  })
  .attr("text-anchor", d => {
    if (d.boro_nm === "Staten Island" && d.category === "Potential Crime") return "start";
    if (d.boro_nm === "Queens" && d.category === "Confirmed Crime") return "end";
    return "middle";
  })
  .attr("fill", "#000")
  .style("font-size", "11px")
  .style("font-weight", "600");

const legend = scatterSvg.append("g")
  .attr("transform", `translate(${scatterWidth - scatterMargin.right / 5}, ${scatterMargin.top})`);

if (scatterFocus === "All Categories") {
  callCategories.forEach((cat, i) => {
    const g = legend.append("g").attr("transform", `translate(0, ${i * 20})`);
    g.append("rect")
      .attr("width", 14)
      .attr("height", 14)
      .attr("fill", scatterColor(cat));
    g.append("text")
      .attr("x", -10)
      .attr("y", 11)
      .attr("text-anchor", "end")
      .attr("fill", "#000")
      .style("font-size", "12px")
      .text(cat);
  });
}

const sizeLegend = legend.append("g")
  .attr("transform", `translate(0, ${scatterFocus === "All Categories" ? callCategories.length * 20 + 20 : 12})`);

const legendSizes = [arrivalExtent[0], d3.median(arrivalExtent), arrivalExtent[1]].map(d => Math.max(1, d));
legendSizes.forEach((val, i) => {
  const y = i * 34;
  sizeLegend.append("circle")
    .attr("cx", 0)
    .attr("cy", y)
    .attr("r", scatterR(val))
    .attr("fill", "#0f6cbd")
    .attr("opacity", 0.25)
    .attr("stroke", "#0f6cbd");
  sizeLegend.append("text")
    .attr("x", -scatterR(val) - 10)
    .attr("y", y + 4)
    .attr("text-anchor", "end")
    .attr("fill", "#000")
    .style("font-size", "12px")
    .text(`${d3.format(".1f")(val)} min arrival delay`);
});

display(scatterSvg.node());
```

<details>
<summary>Code</summary>

```javascript
// Filter for whichever dataset is active; "All" shows both categories together
const scatterData = scatterFocus === "All Categories"
  ? boroughProfiles
  : boroughProfiles.filter(d => d.category === scatterFocus);
const selectedBorough = scatterBorough === "Citywide" ? null : scatterBorough;
const scatterWidth = 960;
const scatterHeight = 460;
const scatterMargin = {top: 60, right: 160, bottom: 55, left: 80};
const scatterX = d3.scaleLinear()
  .domain([0, d3.max(scatterData, d => d.call_count) * 1.05]).nice()
  .range([scatterMargin.left, scatterWidth - scatterMargin.right]);
const scatterY = d3.scaleLinear()
  .domain([0, d3.max(scatterData, d => d.median_call_duration) * 1.15]).nice()
  .range([scatterHeight - scatterMargin.bottom, scatterMargin.top]);
const arrivalExtent = d3.extent(scatterData, d => d.median_arrival_delay);
const scatterR = d3.scaleSqrt()
  .domain([Math.max(1, arrivalExtent[0]), arrivalExtent[1] || Math.max(1, arrivalExtent[0]) + 1])
  .range([6, 24]);
const scatterColor = d3.scaleOrdinal()
  .domain(callCategories)
  .range(["#0f6cbd", "#d1495b"]);

const scatterSvg = d3.create("svg")
  .attr("viewBox", [0, 0, scatterWidth, scatterHeight])
  .attr("width", scatterWidth)
  .attr("height", scatterHeight)
  .style("max-width", "100%")
  .style("height", "auto")
  .style("background", "#dfdfd6");

const scatterInnerWidth = scatterWidth - scatterMargin.left - scatterMargin.right;
const scatterInnerHeight = scatterHeight - scatterMargin.top - scatterMargin.bottom;

scatterSvg.append("g")
  .attr("transform", `translate(0, ${scatterHeight - scatterMargin.bottom})`)
  .call(d3.axisBottom(scatterX).ticks(5, "~s").tickSize(-scatterInnerHeight).tickFormat(() => ""))
  .selectAll("line")
  .attr("stroke", "#bfbfbf")
  .attr("stroke-dasharray", "3,3");

scatterSvg.append("g")
  .attr("transform", `translate(${scatterMargin.left},0)`)
  .call(d3.axisLeft(scatterY).ticks(5).tickSize(-scatterInnerWidth).tickFormat(() => ""))
  .selectAll("line")
  .attr("stroke", "#bfbfbf")
  .attr("stroke-dasharray", "3,3");

scatterSvg.append("line")
  .attr("x1", scatterMargin.left)
  .attr("x2", scatterWidth - scatterMargin.right)
  .attr("y1", scatterHeight - scatterMargin.bottom)
  .attr("y2", scatterHeight - scatterMargin.bottom)
  .attr("stroke", "#000")
  .attr("stroke-width", 1.4);

scatterSvg.append("line")
  .attr("x1", scatterMargin.left)
  .attr("x2", scatterMargin.left)
  .attr("y1", scatterMargin.top)
  .attr("y2", scatterHeight - scatterMargin.bottom)
  .attr("stroke", "#000")
  .attr("stroke-width", 1.4);

scatterSvg.append("g")
  .attr("transform", `translate(0, ${scatterHeight - scatterMargin.bottom})`)
  .call(d3.axisBottom(scatterX).ticks(5, "~s"))
  .selectAll("text")
  .style("fill", "#000");

scatterSvg.append("text")
  .attr("x", scatterWidth / 2)
  .attr("y", scatterHeight - 10)
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .text("Call volume (2024)");

scatterSvg.append("g")
  .attr("transform", `translate(${scatterMargin.left},0)`)
  .call(d3.axisLeft(scatterY))
  .selectAll("text")
  .style("fill", "#000");

scatterSvg.append("text")
  .attr("x", -scatterHeight / 2)
  .attr("y", 20)
  .attr("transform", "rotate(-90)")
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .text("Median call duration (minutes)");

const nodes = scatterSvg.append("g")
  .selectAll("g.node")
  .data(scatterData)
  .join("g")
  .attr("transform", d => `translate(${scatterX(d.call_count)}, ${scatterY(d.median_call_duration)})`);

nodes.append("circle")
  .attr("r", d => scatterR(d.median_arrival_delay))
  .attr("fill", d => scatterFocus === "All Categories" ? scatterColor(d.category) : d.category === "Confirmed Crime" ? "#0f6cbd" : "#d1495b")
  .attr("opacity", d => selectedBorough && d.boro_nm !== selectedBorough ? 0.3 : 0.95)
  .attr("stroke", d => d.boro_nm === selectedBorough ? "#000" : "#5d768d")
  .attr("stroke-width", d => d.boro_nm === selectedBorough ? 2 : 1)
  .append("title")
  .text(d => `${d.boro_nm} · ${d.category}
${d.call_count.toLocaleString()} calls
Median duration: ${d.median_call_duration} min
Median arrival delay: ${d.median_arrival_delay} min`);

nodes.append("text")
  .text(d => d.boro_nm)
  .attr("dx", d => {
    if (d.boro_nm === "Staten Island" && d.category === "Potential Crime") return 10;
    if (d.boro_nm === "Queens" && d.category === "Confirmed Crime") return -15;
    return 0;
  })
  .attr("dy", d => {
    const base = -scatterR(d.median_arrival_delay) - 6;
    if (d.boro_nm === "Staten Island" && d.category === "Confirmed Crime") return base - 2;
    if (d.boro_nm === "Staten Island" && d.category === "Potential Crime") return base + 2;
    if (d.boro_nm === "Queens" && d.category === "Confirmed Crime") return base + 16;
    return base;
  })
  .attr("text-anchor", d => {
    if (d.boro_nm === "Staten Island" && d.category === "Potential Crime") return "start";
    if (d.boro_nm === "Queens" && d.category === "Confirmed Crime") return "end";
    return "middle";
  })
  .attr("fill", "#000")
  .style("font-size", "11px")
  .style("font-weight", "600");

const legend = scatterSvg.append("g")
  .attr("transform", `translate(${scatterWidth - scatterMargin.right / 5}, ${scatterMargin.top})`);

if (scatterFocus === "All Categories") {
  callCategories.forEach((cat, i) => {
    const g = legend.append("g").attr("transform", `translate(0, ${i * 20})`);
    g.append("rect")
      .attr("width", 14)
      .attr("height", 14)
      .attr("fill", scatterColor(cat));
    g.append("text")
      .attr("x", -10)
      .attr("y", 11)
      .attr("text-anchor", "end")
      .attr("fill", "#000")
      .style("font-size", "12px")
      .text(cat);
  });
}

const sizeLegend = legend.append("g")
  .attr("transform", `translate(0, ${scatterFocus === "All Categories" ? callCategories.length * 20 + 20 : 12})`);

const legendSizes = [arrivalExtent[0], d3.median(arrivalExtent), arrivalExtent[1]].map(d => Math.max(1, d));
legendSizes.forEach((val, i) => {
  const y = i * 34;
  sizeLegend.append("circle")
    .attr("cx", 0)
    .attr("cy", y)
    .attr("r", scatterR(val))
    .attr("fill", "#0f6cbd")
    .attr("opacity", 0.25)
    .attr("stroke", "#0f6cbd");
  sizeLegend.append("text")
    .attr("x", -scatterR(val) - 10)
    .attr("y", y + 4)
    .attr("text-anchor", "end")
    .attr("fill", "#000")
    .style("font-size", "12px")
    .text(`${d3.format(".1f")(val)} min arrival delay`);
});

display(scatterSvg.node());
```
</details>

### Analysis & Insights

<ol class="insight-list">
  <li>Bronx and Queens appear in the upper part of the chart for both confirmed and potential crimes. They have the largest bubbles as well, indicating long call durations and long arrival delays
    <div class="insight">
      <strong>Insight 💡</strong><br>
      These boroughs form the clearest “service strain cluster,” where high incident complexity and slow on-scene response reinforce each other. They represent the most critical bottlenecks for system-wide performance
    </div>
  </li>
  <li>Bronx (Potential Crime) sits noticeably higher than all other points (~70 min duration) with the largest bubble on the chart
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Ambiguous or escalating situations in the Bronx demand the most operator time and face the longest dispatch delays. This highlights a particularly vulnerable segment of 911 operations needing targeted intervention
    </div>
  </li>
  <li>Brooklyn shows the largest gap in call volume between confirmed and potential crime categories (~113k calls)
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Brooklyn’s surge in potential-crime calls indicates a heavy burden of ambiguous incidents that require operator attention. This imbalance suggests that the 911 load is driven more by community-reported concerns than confirmed emergencies. This could be good place to implement improved triage or automated screening tools that reduce operator workload without compromising public safety
    </div>
  </li>
  <li>Brooklyn (Potential Crime) shows high call volume (~340k) but retains moderate call duration and a smaller bubble relative to Bronx and Queens
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Brooklyn absorbs very large demand without proportional degradation in service speed. This suggests stronger dispatch capacity or more efficient field routing despite call volume being the highest on the chart
    </div>
  </li>
  <li>Manhattan consistently sits at the bottom of the duration axis for both categories and shows some of the smallest bubbles
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Manhattan demonstrates superior operational efficiency. This is likely due to dense unit availability and compact geography. It serves as a benchmark borough for balancing workload and response speed
    </div>
  </li>
  <li>Staten Island shows low call volume but relatively elevated call durations (~42-45 minutes) compared with Manhattan and Brooklyn
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Call resolution time in Staten Island isn't being driven by load but by structural factors like distance, limited staffing, or case types. Improving travel logistics or operator specialization would likely yield outsized gains here
    </div>
  </li>
  <li>The spatial separation on the x-axis reveals a pattern: higher-volume boroughs (Brooklyn, Queens) do not uniformly experience the worst delays. Whereas lower-volume boroughs (Bronx for potential crime) can show the highest delays
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Service speed is not purely volume-driven. Borough-specific incident profiles and unit distribution matter more than raw call counts. Resource allocation should be complexity-adjusted, not just volume-adjusted
    </div>
  </li>
</ol>


## How does 911 call volume vary across boroughs?

### 911 call volume by borough

```js
const mapWidth = 420;
const mapHeight = 520;
const projection = d3.geoMercator().fitSize([mapWidth, mapHeight], boroughBoundaries);
const path = d3.geoPath(projection);
const maxCalls = d3.max(boroughCounts, d => d.call_count);
const mapColor = d3.scaleSequential(d3.interpolateYlOrRd).domain([0, maxCalls]);

const grid = d3.create('div')
  .style('display', 'grid')
  .style('grid-template-columns', 'repeat(auto-fit, minmax(300px, 1fr))')
  .style('gap', '1.5rem');

const selectedMaps = callCategories;

for (const category of selectedMaps) {
  const catRows = boroughCounts.filter(d => d.category === category);
  const data = new Map(catRows.map(d => [d.boro_nm, d.call_count]));
  const totalCalls = d3.sum(catRows, d => d.call_count);

  const svg = grid.append('svg')
    .attr('viewBox', [0, 0, mapWidth, mapHeight])
    .attr('width', mapWidth)
    .attr('height', mapHeight)
    .style('max-width', '100%')
    .style('height', 'auto')
    .style('background', '#dfdfd6');

  svg.append('text')
    .attr('x', mapWidth / 2)
    .attr('y', 28)
    .attr('text-anchor', 'middle')
    .attr('fill', '#000')
    .style('font-size', '18px')
    .style('font-weight', '600')
    .text(category);

  const mapGroup = svg.append('g').attr('transform', 'translate(0, 40)');

  mapGroup.selectAll('path')
    .data(boroughBoundaries.features)
    .join('path')
    .attr('d', path)
    .attr('fill', d => mapColor(data.get(d.properties.BoroName) || 0))
    .attr('stroke', '#2f2f2f')
    .attr('stroke-width', 1.2)
    .append('title')
    .text(d => `${d.properties.BoroName}
${(data.get(d.properties.BoroName) || 0).toLocaleString()} calls`);

  // Borough labels and percent within each outline
  mapGroup.append('g')
    .selectAll('g')
    .data(boroughBoundaries.features)
    .join('g')
    .attr('transform', d => {
      const [x, y] = path.centroid(d);
      return d.properties.BoroName === 'Manhattan'
        ? `translate(${x - 10}, ${y + 6}) rotate(-70)`
        : d.properties.BoroName === 'Brooklyn'
        ? `translate(${x}, ${y - 10})`
        : d.properties.BoroName === 'Staten Island'
        ? `translate(${x + 8}, ${y})`
        : d.properties.BoroName === 'Queens'
        ? `translate(${x + 6}, ${y - 6})`
        : `translate(${x}, ${y})`;
    })
    .call(g => {
      g.append('text')
        .attr('text-anchor', 'middle')
        .attr('fill', '#111')
        .style('font-size', '11px')
        .style('font-weight', '700')
        .text(d => d.properties.BoroName);
      g.append('text')
        .attr('dy', 14)
        .attr('text-anchor', 'middle')
        .attr('fill', '#111')
        .style('font-size', '10px')
        .style('font-weight', '600')
        .text(d => {
          const calls = data.get(d.properties.BoroName) || 0;
          const pct = totalCalls ? (calls / totalCalls) * 100 : 0;
          return `${pct.toFixed(1)}%`;
        });
    });
}

const legendWidth = 260;
const legendHeight = 14;
const legendScale = d3.scaleLinear().domain(mapColor.domain()).range([0, legendWidth]);
const defs = grid.append('svg')
  .attr('viewBox', [0, 0, legendWidth + 80, 60])
  .style('max-width', '100%')
  .style('height', 'auto');

const gradientId = 'mapGradient';
const gradient = defs.append('defs').append('linearGradient')
  .attr('id', gradientId)
  .attr('x1', '0%').attr('x2', '100%')
  .attr('y1', '0%').attr('y2', '0%');

for (let i = 0; i <= 10; i++) {
  const t = i / 10;
  gradient.append('stop')
    .attr('offset', `${t * 100}%`)
    .attr('stop-color', mapColor(legendScale.invert(t * legendWidth)));
}

defs.append('rect')
  .attr('x', 20)
  .attr('y', 10)
  .attr('width', legendWidth)
  .attr('height', legendHeight)
  .attr('fill', `url(#${gradientId})`)
  .attr('stroke', '#333');

defs.append('g')
  .attr('transform', `translate(20, ${10 + legendHeight})`)
  .call(d3.axisBottom(legendScale).ticks(5, '~s'))
  .selectAll('text')
  .style('fill', '#000');

defs.append('text')
  .attr('x', 20)
  .attr('y', 56.5)
  .attr('fill', '#000')
  .style('font-size', '12px')
  .text('Call volume per borough');

display(grid.node());
```

<details>
<summary>Code</summary>

```javascript
const mapWidth = 420;
const mapHeight = 520;
const projection = d3.geoMercator().fitSize([mapWidth, mapHeight], boroughBoundaries);
const path = d3.geoPath(projection);
const maxCalls = d3.max(boroughCounts, d => d.call_count);
const mapColor = d3.scaleSequential(d3.interpolateYlOrRd).domain([0, maxCalls]);

const grid = d3.create('div')
  .style('display', 'grid')
  .style('grid-template-columns', 'repeat(auto-fit, minmax(300px, 1fr))')
  .style('gap', '1.5rem');

const selectedMaps = callCategories;

for (const category of selectedMaps) {
  const data = new Map(boroughCounts.filter(d => d.category === category).map(d => [d.boro_nm, d.call_count]));

  const svg = grid.append('svg')
    .attr('viewBox', [0, 0, mapWidth, mapHeight])
    .attr('width', mapWidth)
    .attr('height', mapHeight)
    .style('max-width', '100%')
    .style('height', 'auto')
    .style('background', '#dfdfd6');

  svg.append('text')
    .attr('x', mapWidth / 2)
    .attr('y', 28)
    .attr('text-anchor', 'middle')
    .attr('fill', '#000')
    .style('font-size', '18px')
    .style('font-weight', '600')
    .text(category);

  svg.append('g')
    .attr('transform', 'translate(0, 40)')
    .selectAll('path')
    .data(boroughBoundaries.features)
    .join('path')
    .attr('d', path)
    .attr('fill', d => mapColor(data.get(d.properties.BoroName) || 0))
    .attr('stroke', '#2f2f2f')
    .attr('stroke-width', 1.2)
    .append('title')
    .text(d => `${d.properties.BoroName}
${(data.get(d.properties.BoroName) || 0).toLocaleString()} calls`);
}

const legendWidth = 260;
const legendHeight = 14;
const legendScale = d3.scaleLinear().domain(mapColor.domain()).range([0, legendWidth]);
const defs = grid.append('svg')
  .attr('viewBox', [0, 0, legendWidth + 80, 60])
  .style('max-width', '100%')
  .style('height', 'auto');

const gradientId = 'mapGradient';
const gradient = defs.append('defs').append('linearGradient')
  .attr('id', gradientId)
  .attr('x1', '0%').attr('x2', '100%')
  .attr('y1', '0%').attr('y2', '0%');

for (let i = 0; i <= 10; i++) {
  const t = i / 10;
  gradient.append('stop')
    .attr('offset', `${t * 100}%`)
    .attr('stop-color', mapColor(legendScale.invert(t * legendWidth)));
}

defs.append('rect')
  .attr('x', 20)
  .attr('y', 10)
  .attr('width', legendWidth)
  .attr('height', legendHeight)
  .attr('fill', `url(#${gradientId})`)
  .attr('stroke', '#333');

defs.append('g')
  .attr('transform', `translate(20, ${10 + legendHeight})`)
  .call(d3.axisBottom(legendScale).ticks(5, '~s'))
  .selectAll('text')
  .style('fill', '#fff');

defs.append('text')
  .attr('x', 20)
  .attr('y', 56.5)
  .attr('fill', '#fff')
  .style('font-size', '12px')
  .text('Call volume per borough');

display(grid.node());
```

</details>

### Analysis & Insights

<ol class="insight-list">
  <li>Brooklyn has the highest call volume citywide by being the darkest-shaded borough in both confirmed and potential crime
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Brooklyn carries the heaviest 911 demand overall. Such concentrated volume makes it a borough requiring strong triage and dispatch capacity (This aligns with Brooklyn’s placement in Chart 1)
    </div>
  </li>
  <li>Manhattan shows the next-highest shading, with substantial call volume in both categories
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Manhattan maintains consistently high demand, reflecting dense population, business activity, and continuous mobility throughout the borough. Its elevated volume underscores the need for well-coordinated, high-frequency response coverage
    </div>
  </li>
  <li>Queens is moderately dark, indicating high call volume but slightly lower than Manhattan and Brooklyn
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Queens handles a large share of citywide calls and contributes significantly to overall 911 workload. Managing this volume effectively requires broad coverage and scalable response strategies
    </div>
  </li>
  <li>The Bronx displays lighter shading than the top three boroughs but still shows considerable call volume, especially for potential crime
    <div class="insight">
      <strong>Insight 💡</strong><br>
      The noticeable increase in potential-crime calls along with incident type helps explain why the borough exhibited longer durations and delays in Chart 1, despite not having the highest call count
    </div>
  </li>
  <li>Staten Island is the lightest-shaded borough, with the lowest call volume in both maps
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Staten Island's low overall demand suggests a quieter operational environment compared to the other boroughs. Its main challenges likely stem from geography and travel distance rather than call overload
    </div>
  </li>
</ol>


## How does 911 call volume vary within boroughs?

### Precinct-level 911 Hotspots

- Precincts are shaded by their total 911 call volume. Darker colors indicate higher workload across all call types 
- Call volume is computed as the count of all calls assigned to each precinct. This is mapped geographically to highlight intensity patterns that borough-level views cannot show

```js
const precinctMapCategory = view(Inputs.radio(["All Categories", ...callCategories], {
  label: "Call set",
  value: "All Categories"
}));
```

```js
const precinctGeo = await FileAttachment("NYPD_Precincts.geojson").json();

const precinctMapWidth = 900;
const precinctMapHeight = 760;
const precinctPadding = 30;
const precinctMapColor = d3.scaleSequential(d3.interpolateYlOrRd);

const precinctProjection = d3.geoMercator()
  .fitExtent(
    [[precinctPadding, precinctPadding + 20], [precinctMapWidth - precinctPadding, precinctMapHeight - precinctPadding - 30]],
    precinctGeo
  );
const precinctPath = d3.geoPath(precinctProjection);

// Volume by precinct (numeric)
const mapData = new Map(
  precinctCounts
    .filter(d => precinctMapCategory === "All Categories" || d.category === precinctMapCategory)
    .map(d => [+d.nypd_pct_cd, +d.call_count])
);

const maxPrecCalls = d3.max(mapData.values());
precinctMapColor.domain([0, maxPrecCalls || 1]);

const precinctSvg = d3.create("svg")
  .attr("viewBox", [0, 0, precinctMapWidth, precinctMapHeight])
  .attr("width", precinctMapWidth)
  .attr("height", precinctMapHeight)
  .style("max-width", "100%")
  .style("height", "auto")
  .style("background", "#dfdfd6");

precinctSvg.append("g")
  .selectAll("path")
  .data(precinctGeo.features)
  .join("path")
  .attr("d", precinctPath)
  .attr("fill", d => precinctMapColor(mapData.get(+d.properties.Precinct) || 0))
  .attr("stroke", "#fff")
  .attr("stroke-width", 1.2)
  .append("title")
  .text(d => {
    const pct = +d.properties.Precinct;
    const calls = mapData.get(pct) || 0;
    return `Precinct ${pct}\n${calls.toLocaleString()} calls`;
  });

// Borough separation stroke (background color gap)
precinctSvg.append("g")
  .selectAll("path")
  .data(boroughBoundaries.features)
  .join("path")
  .attr("d", precinctPath)
  .attr("fill", "none")
  .attr("stroke", "#dfdfd6")
  .attr("stroke-width", 6);

// Borough outlines for context
precinctSvg.append("g")
  .selectAll("path")
  .data(boroughBoundaries.features)
  .join("path")
  .attr("d", precinctPath)
  .attr("fill", "none")
  .attr("stroke", "#333")
  .attr("stroke-width", 2.5);

// Legend
const legWidth = 260;
const legHeight = 12;
const legMargin = {left: 40, top: 60};

const legScale = d3.scaleLinear()
  .domain(precinctMapColor.domain())
  .range([0, legWidth]);

const defs = precinctSvg.append("defs");
const gradId = "precinctGrad";
const gradient = defs.append("linearGradient")
  .attr("id", gradId)
  .attr("x1", "0%").attr("x2", "100%")
  .attr("y1", "0%").attr("y2", "0%");

for (let i = 0; i <= 10; i++) {
  const t = i / 10;
  gradient.append("stop")
    .attr("offset", `${t * 100}%`)
    .attr("stop-color", precinctMapColor(legScale.invert(t * legWidth)));
}

const legend = precinctSvg.append("g")
  .attr("transform", `translate(${legMargin.left}, ${legMargin.top + 14})`);

legend.append("rect")
  .attr("width", legWidth)
  .attr("height", legHeight)
  .attr("fill", `url(#${gradId})`)
  .attr("stroke", "#333");

legend.append("g")
  .attr("transform", `translate(0, ${legHeight})`)
  .call(d3.axisBottom(legScale).ticks(5, "~s"))
  .selectAll("text")
  .style("fill", "#000");

legend.append("text")
  .attr("x", 0)
  .attr("y", -6)
  .attr("fill", "#000")
  .style("font-size", "12px")
  .text("Call volume");

precinctSvg.append("text")
  .attr("x", legMargin.left)
  .attr("y", legMargin.top - 16)
  .attr("text-anchor", "start")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .style("font-size", "18px")
  .text("Precinct call volume");

display(precinctSvg.node());
```

<details>
<summary>Code</summary>

```javascript
const precinctMapCategory = view(Inputs.radio(["All Categories", ...callCategories], {
  label: "Call set",
  value: "All Categories"
}));

const precinctGeo = await FileAttachment("NYPD_Precincts.geojson").json();

const precinctMapWidth = 900;
const precinctMapHeight = 780;
const precinctMapColor = d3.scaleSequential(d3.interpolateYlOrRd);
const precinctPadding = 30;

const precinctProjection = d3.geoMercator()
  .fitExtent(
    [[precinctPadding, precinctPadding + 20], [precinctMapWidth - precinctPadding, precinctMapHeight - precinctPadding]],
    precinctGeo
  );
const precinctPath = d3.geoPath(precinctProjection);

const mapData = new Map(
  precinctCounts
    .filter(d => precinctMapCategory === "All Categories" || d.category === precinctMapCategory)
    .map(d => [d.nypd_pct_cd, d.call_count])
);

const maxPrecCalls = d3.max(mapData.values());
precinctMapColor.domain([0, maxPrecCalls]);

const precinctSvg = d3.create("svg")
  .attr("viewBox", [0, 0, precinctMapWidth, precinctMapHeight])
  .attr("width", precinctMapWidth)
  .attr("height", precinctMapHeight)
  .style("max-width", "100%")
  .style("height", "auto")
  .style("background", "#dfdfd6");

precinctSvg.append("text")
  .attr("x", precinctMapWidth / 2)
  .attr("y", 32)
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .style("font-size", "18px")
  .text("Precinct call volume");

precinctSvg.append("g")
  .selectAll("path")
  .data(precinctGeo.features)
  .join("path")
  .attr("d", precinctPath)
  .attr("fill", d => precinctMapColor(mapData.get(d.properties.Precinct) || 0))
  .attr("stroke", "#fff")
  .attr("stroke-width", 1.2)
  .append("title")
  .text(d => {
    const pct = d.properties.Precinct;
    const calls = mapData.get(pct) || 0;
    return `Precinct ${pct}\n${calls.toLocaleString()} calls`;
  });

precinctSvg.append("g")
  .selectAll("path")
  .data(boroughBoundaries.features)
  .join("path")
  .attr("d", precinctPath)
  .attr("fill", "none")
  .attr("stroke", "#333")
  .attr("stroke-width", 2.5);

const legWidth = 260;
const legHeight = 12;
const legMargin = {left: 40, top: 60};

const legScale = d3.scaleLinear()
  .domain(precinctMapColor.domain())
  .range([0, legWidth]);

const defs = precinctSvg.append("defs");
const gradId = "precinctGrad";
const gradient = defs.append("linearGradient")
  .attr("id", gradId)
  .attr("x1", "0%").attr("x2", "100%")
  .attr("y1", "0%").attr("y2", "0%");

for (let i = 0; i <= 10; i++) {
  const t = i / 10;
  gradient.append("stop")
    .attr("offset", `${t * 100}%`)
    .attr("stop-color", precinctMapColor(legScale.invert(t * legWidth)));
}

const legend = precinctSvg.append("g")
  .attr("transform", `translate(${legMargin.left}, ${legMargin.top})`);

legend.append("rect")
  .attr("width", legWidth)
  .attr("height", legHeight)
  .attr("fill", `url(#${gradId})`)
  .attr("stroke", "#333");

legend.append("g")
  .attr("transform", `translate(0, ${legHeight})`)
  .call(d3.axisBottom(legScale).ticks(5, "~s"))
  .selectAll("text")
  .style("fill", "#000");

legend.append("text")
  .attr("x", 0)
  .attr("y", -6)
  .attr("fill", "#000")
  .style("font-size", "12px")
  .text("Call volume per precinct");

display(precinctSvg.node());
```

</details>

### Analysis & Insights

<ol class="insight-list">
  <li>Precinct-level variation within boroughs is far greater than borough averages suggest
    <div class="insight">
      <strong>Insight 💡</strong><br>
      The workload is not evenly distributed even in high-volume boroughs like Brooklyn and Queens. Some precincts face double or triple the call volume of adjacent areas, signaling where resource allocation can have the greatest impact
    </div>
  </li>
  <li>Call volume is heavily concentrated in central and southeastern Brooklyn. Several precincts exceed 20k+ calls
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Brooklyn’s role as the city’s largest call generator becomes even clearer at the precinct level. High-density residential areas and major transit corridors drive sustained 911 demand
    </div>
  </li>
  <li>Northern Queens, the South Bronx, and Uptown Manhattan show elevated precinct volumes
    <div class="insight">
      <strong>Insight 💡</strong><br>
      These regions form secondary hotspots of 911 activity. They reflect mixed residential-commercial zones, large transit hubs, and busy public spaces. Both confirmed and potential-crime calls tend to cluster in these environments
    </div>
  </li>
  <li>Staten Island precincts are consistently among the lowest-volume areas
    <div class="insight">
      <strong>Insight 💡</strong><br>
      The wide geographic coverage and lower population density result in far fewer calls compared with the rest of the city. This supports earlier observations that Staten Island’s operational challenges stem more from travel distance and coverage rather than raw call intensity
    </div>
  </li>
</ol>

## Which precincts defy the volume-delay pattern, and why?

### Precinct call volume with arrival delay

- Precinct fill uses a grayscale ramp (light = fewer calls, dark = more calls)
- Centroid dots are colored by median arrival delay (green = faster, yellow = moderate, red = slower)

```js
const bivarCategory = view(Inputs.radio(["All Categories", ...callCategories], {
  label: "Call set",
  value: "All Categories"
}));
```

```js
const precinctGeo = await FileAttachment("NYPD_Precincts.geojson").json();

// Volume by precinct (numeric, same approach as hotspots)
const volRows = bivarCategory === "All Categories"
  ? precinctCounts
  : precinctCounts.filter(d => d.category === bivarCategory);
const callByPct = new Map();
volRows.forEach(d => {
  const pct = +d.nypd_pct_cd;
  const count = +d.call_count;
  callByPct.set(pct, (callByPct.get(pct) || 0) + count);
});

// Delay by precinct (weighted)
const delayRows = bivarCategory === "All Categories"
  ? precinctProfiles
  : precinctProfiles.filter(d => d.category === bivarCategory);
const delaySums = new Map();
const delayCalls = new Map();
delayRows.forEach(d => {
  const pct = +d.nypd_pct_cd;
  const calls = +d.call_count;
  const delay = +d.median_arrival_delay;
  delaySums.set(pct, (delaySums.get(pct) || 0) + delay * calls);
  delayCalls.set(pct, (delayCalls.get(pct) || 0) + calls);
});
const delayByPct = new Map(
  Array.from(delaySums, ([pct, sum]) => {
    const calls = delayCalls.get(pct) || 0;
    return [pct, calls ? sum / calls : NaN];
  })
);

const bivarWidth = 900;
const bivarHeight = 760;
const bivarPadding = 30;
const bivarProjection = d3.geoMercator()
  .fitExtent(
    [[bivarPadding, bivarPadding + 20], [bivarWidth - bivarPadding, bivarHeight - bivarPadding]],
    precinctGeo
  );
const bivarPath = d3.geoPath(bivarProjection);

const callExtent = d3.extent(callByPct.values());
const volColor = d3.scaleLinear()
  .domain([0, callExtent[1] || 1])
  .range(["#f2f2f2", "#2f2f2f"]);

const delayVals = Array.from(delayByPct.values()).filter(Number.isFinite);
const delayBreaks = delayVals.length ? [d3.quantile(delayVals, 0.33), d3.quantile(delayVals, 0.66)] : [5, 10];
const delayColor = d => {
  if (!Number.isFinite(d)) return "#777";
  if (d <= delayBreaks[0]) return "#3cb371";   // green = faster
  if (d <= delayBreaks[1]) return "#f6c343";   // yellow = moderate
  return "#d84a4a";                            // red = slower
};

// Helper to keep label/dot points inside polygons (handles concave shapes)
function interiorPoint(feature) {
  // Flatten rings
  const rings = feature.geometry.type === "Polygon"
    ? feature.geometry.coordinates
    : feature.geometry.coordinates.flat();

  // Pick largest ring by projected area
  let best = null;
  rings.forEach(ring => {
    const projected = ring.map(pt => bivarProjection(pt));
    const area = Math.abs(d3.polygonArea(projected));
    if (!best || area > best.area) best = {projected, area};
  });
  if (!best) return bivarPath.centroid(feature);

  let p = d3.polygonCentroid(best.projected);
  if (d3.polygonContains(best.projected, p)) return p;

  // Fallback: move halfway toward ring centroid until inside
  const center = d3.polygonCentroid(best.projected);
  let iter = 0;
  while (!d3.polygonContains(best.projected, p) && iter < 5) {
    p = [(p[0] + center[0]) / 2, (p[1] + center[1]) / 2];
    iter++;
  }
  return p;
}

const bivarLabel = typeof bivarCategory === "string" ? bivarCategory : "All Categories";

const bivarSvg = d3.create("svg")
  .attr("viewBox", [0, 0, bivarWidth, bivarHeight])
  .attr("width", bivarWidth)
  .attr("height", bivarHeight)
  .style("max-width", "100%")
  .style("height", "auto")
  .style("background", "#dfdfd6");

bivarSvg.append("text")
  .attr("x", bivarWidth / 2)
  .attr("y", 34)
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .style("font-size", "18px")
  .text("Precinct call volume + arrival delay");

bivarSvg.append("g")
  .selectAll("path")
  .data(precinctGeo.features)
  .join("path")
  .attr("d", bivarPath)
  .attr("fill", d => volColor(callByPct.get(+d.properties.Precinct) || 0))
  .attr("stroke", "#b3b3b3")
  .attr("stroke-width", 0.9)
  .append("title")
  .text(d => {
    const pct = +d.properties.Precinct;
    const calls = callByPct.get(pct) || 0;
    const delay = delayByPct.get(pct);
    return `Precinct ${pct}
${calls.toLocaleString()} calls
Median arrival delay: ${Number.isFinite(delay) ? d3.format(".1f")(delay) : "n/a"} min`;
  });

// Precinct borders overlay for clarity
bivarSvg.append("g")
  .selectAll("path")
  .data(precinctGeo.features)
  .join("path")
  .attr("d", bivarPath)
  .attr("fill", "none")
  .attr("stroke", "#4a4a4a")
  .attr("stroke-width", 1.1);

// Delay dots at precinct centroids
bivarSvg.append("g")
  .selectAll("circle")
  .data(precinctGeo.features)
  .join("circle")
  .attr("cx", d => interiorPoint(d)[0])
  .attr("cy", d => interiorPoint(d)[1])
  .attr("r", 5.5)
  .attr("fill", d => delayColor(delayByPct.get(+d.properties.Precinct)))
  .attr("stroke", "#000")
  .attr("stroke-width", 0.6)
  .attr("opacity", 0.9)
  .append("title")
  .text(d => {
    const pct = +d.properties.Precinct;
    const calls = callByPct.get(pct) || 0;
    const delay = delayByPct.get(pct);
    return `Precinct ${pct}
${calls.toLocaleString()} calls
Median arrival delay: ${Number.isFinite(delay) ? d3.format(".1f")(delay) : "n/a"} min`;
  });

// Borough outlines for context
bivarSvg.append("g")
  .selectAll("path")
  .data(boroughBoundaries.features)
  .join("path")
  .attr("d", bivarPath)
  .attr("fill", "none")
  .attr("stroke", "#333")
  .attr("stroke-width", 1.2);

// Volume legend (grayscale fill)
const volWidth = 240;
const volHeight = 12;
const volMargin = {left: 40, top: 70};
const volScale = d3.scaleLinear().domain(volColor.domain()).range([0, volWidth]);
const volDefs = bivarSvg.append("defs");
const volGradId = "bivarPrecinctVolGrad";
const volGrad = volDefs.append("linearGradient")
  .attr("id", volGradId)
  .attr("x1", "0%").attr("x2", "100%")
  .attr("y1", "0%").attr("y2", "0%");
for (let i = 0; i <= 10; i++) {
  const t = i / 10;
  volGrad.append("stop")
    .attr("offset", `${t * 100}%`)
    .attr("stop-color", volColor(volScale.invert(t * volWidth)));
}

const volLegend = bivarSvg.append("g")
  .attr("transform", `translate(${volMargin.left}, ${volMargin.top})`);

volLegend.append("rect")
  .attr("width", volWidth)
  .attr("height", volHeight)
  .attr("fill", `url(#${volGradId})`)
  .attr("stroke", "#333");

volLegend.append("g")
  .attr("transform", `translate(0, ${volHeight})`)
  .call(d3.axisBottom(volScale).ticks(5, "~s"))
  .selectAll("text")
  .style("fill", "#000");

volLegend.append("text")
  .attr("x", 0)
  .attr("y", -6)
  .attr("fill", "#000")
  .style("font-size", "12px")
  .text("Call volume");

// Delay legend (dot colors)
const delayLegend = bivarSvg.append("g")
  .attr("transform", `translate(${volMargin.left}, ${volMargin.top + 72})`);

[
  {label: `Fast (≤ ${d3.format(".1f")(delayBreaks[0])} min)`, color: "#3cb371"},
  {label: `Moderate (${d3.format(".1f")(delayBreaks[0])}–${d3.format(".1f")(delayBreaks[1])} min)`, color: "#f6c343"},
  {label: `Slow (> ${d3.format(".1f")(delayBreaks[1])} min)`, color: "#d84a4a"}
].forEach((d, i) => {
  const g = delayLegend.append("g").attr("transform", `translate(0, ${6 + i * 18})`);
  g.append("circle")
    .attr("r", 6)
    .attr("cx", 0)
    .attr("cy", 0)
    .attr("fill", d.color)
    .attr("stroke", "#000")
    .attr("stroke-width", 0.6);
  g.append("text")
    .attr("x", 12)
    .attr("y", 4)
    .attr("fill", "#000")
    .style("font-size", "12px")
    .text(d.label);
});

delayLegend.append("text")
  .attr("x", 0)
  .attr("y", -10)
  .attr("fill", "#000")
  .style("font-size", "12px")
  .text("Median arrival delay");

display(bivarSvg.node());
```

<details>
<summary>Code</summary>

```javascript
const bivarCategory = view(Inputs.radio(["All Categories", ...callCategories], {
  label: "Call set",
  value: "All Categories"
}));

const precinctGeo = await FileAttachment("NYPD_Precincts.geojson").json();

const volRows = bivarCategory === "All Categories"
  ? precinctCounts
  : precinctCounts.filter(d => d.category === bivarCategory);
const callByPct = new Map();
volRows.forEach(d => {
  const pct = +d.nypd_pct_cd;
  const count = +d.call_count;
  callByPct.set(pct, (callByPct.get(pct) || 0) + count);
});

const delays = bivarCategory === "All Categories"
  ? precinctProfiles
  : precinctProfiles.filter(d => d.category === bivarCategory);
const delayByPct = new Map();
delays.forEach(d => {
  const pct = +d.nypd_pct_cd;
  const val = +d.median_arrival_delay;
  if (!Number.isFinite(val)) return;
  const current = delayByPct.get(pct) || [];
  current.push(val);
  delayByPct.set(pct, current);
});
delayByPct.forEach((arr, pct) => {
  const mean = d3.mean(arr);
  delayByPct.set(pct, mean);
});

const bivarWidth = 960;
const bivarHeight = 860;
const bivarProjection = d3.geoMercator().fitSize([bivarWidth, bivarHeight], boroughBoundaries);
const bivarPath = d3.geoPath(bivarProjection);

const callExtent = d3.extent(callByPct.values());
const volColor = d3.scaleLinear()
  .domain([0, callExtent[1] || 1])
  .range(["#f2f2f2", "#2f2f2f"]);

const delayVals = Array.from(delayByPct.values()).filter(Number.isFinite);
const delayBreaks = delayVals.length ? [d3.quantile(delayVals, 0.33), d3.quantile(delayVals, 0.66)] : [5, 10];
const delayColor = d => {
  if (!Number.isFinite(d)) return "#777";
  if (d <= delayBreaks[0]) return "#3cb371";
  if (d <= delayBreaks[1]) return "#f6c343";
  return "#d84a4a";
};

function interiorPoint(feature) {
  const rings = feature.geometry.type === "Polygon"
    ? feature.geometry.coordinates
    : feature.geometry.coordinates.flat();
  let best = null;
  rings.forEach(ring => {
    const projected = ring.map(pt => bivarProjection(pt));
    const area = Math.abs(d3.polygonArea(projected));
    if (!best || area > best.area) best = {projected, area};
  });
  if (!best) return bivarPath.centroid(feature);
  let p = d3.polygonCentroid(best.projected);
  if (d3.polygonContains(best.projected, p)) return p;
  const center = d3.polygonCentroid(best.projected);
  let iter = 0;
  while (!d3.polygonContains(best.projected, p) && iter < 5) {
    p = [(p[0] + center[0]) / 2, (p[1] + center[1]) / 2];
    iter++;
  }
  return p;
}

const bivarSvg = d3.create("svg")
  .attr("viewBox", [0, 0, bivarWidth, bivarHeight])
  .attr("width", bivarWidth)
  .attr("height", bivarHeight)
  .style("max-width", "100%")
  .style("height", "auto")
  .style("background", "#dfdfd6");

bivarSvg.append("text")
  .attr("x", bivarWidth / 2)
  .attr("y", 34)
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .style("font-size", "18px")
  .text("Precinct call volume + arrival delay");

bivarSvg.append("g")
  .selectAll("path")
  .data(precinctGeo.features)
  .join("path")
  .attr("d", bivarPath)
  .attr("fill", d => volColor(callByPct.get(+d.properties.Precinct) || 0))
  .attr("stroke", "#b3b3b3")
  .attr("stroke-width", 0.9)
  .append("title")
  .text(d => {
    const pct = +d.properties.Precinct;
    const calls = callByPct.get(pct) || 0;
    const delay = delayByPct.get(pct);
    return `Precinct ${pct}
${calls.toLocaleString()} calls
Median arrival delay: ${Number.isFinite(delay) ? d3.format(".1f")(delay) : "n/a"} min`;
  });

bivarSvg.append("g")
  .selectAll("path")
  .data(precinctGeo.features)
  .join("path")
  .attr("d", bivarPath)
  .attr("fill", "none")
  .attr("stroke", "#4a4a4a")
  .attr("stroke-width", 1.1);

bivarSvg.append("g")
  .selectAll("circle")
  .data(precinctGeo.features)
  .join("circle")
  .attr("cx", d => interiorPoint(d)[0])
  .attr("cy", d => interiorPoint(d)[1])
  .attr("r", 5.5)
  .attr("fill", d => delayColor(delayByPct.get(+d.properties.Precinct)))
  .attr("stroke", "#000")
  .attr("stroke-width", 0.6)
  .attr("opacity", 0.9)
  .append("title")
  .text(d => {
    const pct = +d.properties.Precinct;
    const calls = callByPct.get(pct) || 0;
    const delay = delayByPct.get(pct);
    return `Precinct ${pct}
${calls.toLocaleString()} calls
Median arrival delay: ${Number.isFinite(delay) ? d3.format(".1f")(delay) : "n/a"} min`;
  });

bivarSvg.append("g")
  .selectAll("path")
  .data(boroughBoundaries.features)
  .join("path")
  .attr("d", bivarPath)
  .attr("fill", "none")
  .attr("stroke", "#333")
  .attr("stroke-width", 1.2);

const volWidth = 240;
const volHeight = 12;
const volMargin = {left: 40, top: 70};
const volScale = d3.scaleLinear().domain(volColor.domain()).range([0, volWidth]);
const volDefs = bivarSvg.append("defs");
const volGradId = "bivarPrecinctVolGrad";
const volGrad = volDefs.append("linearGradient")
  .attr("id", volGradId)
  .attr("x1", "0%").attr("x2", "100%")
  .attr("y1", "0%").attr("y2", "0%");
for (let i = 0; i <= 10; i++) {
  const t = i / 10;
  volGrad.append("stop")
    .attr("offset", `${t * 100}%`)
    .attr("stop-color", volColor(volScale.invert(t * volWidth)));
}

const volLegend = bivarSvg.append("g")
  .attr("transform", `translate(${volMargin.left}, ${volMargin.top})`);

volLegend.append("rect")
  .attr("width", volWidth)
  .attr("height", volHeight)
  .attr("fill", `url(#${volGradId})`)
  .attr("stroke", "#333");

volLegend.append("g")
  .attr("transform", `translate(0, ${volHeight})`)
  .call(d3.axisBottom(volScale).ticks(5, "~s"))
  .selectAll("text")
  .style("fill", "#000");

volLegend.append("text")
  .attr("x", 0)
  .attr("y", -6)
  .attr("fill", "#000")
  .style("font-size", "12px")
  .text("Call volume");

const delayLegend = bivarSvg.append("g")
  .attr("transform", `translate(${volMargin.left}, ${volMargin.top + 72})`);

[
  {label: `Fast (≤ ${d3.format(".1f")(delayBreaks[0])} min)`, color: "#3cb371"},
  {label: `Moderate (${d3.format(".1f")(delayBreaks[0])}–${d3.format(".1f")(delayBreaks[1])} min)`, color: "#f6c343"},
  {label: `Slow (> ${d3.format(".1f")(delayBreaks[1])} min)`, color: "#d84a4a"}
].forEach((d, i) => {
  const g = delayLegend.append("g").attr("transform", `translate(0, ${6 + i * 18})`);
  g.append("circle")
    .attr("r", 6)
    .attr("cx", 0)
    .attr("cy", 0)
    .attr("fill", d.color)
    .attr("stroke", "#000")
    .attr("stroke-width", 0.6);
  g.append("text")
    .attr("x", 12)
    .attr("y", 4)
    .attr("fill", "#000")
    .style("font-size", "12px")
    .text(d.label);
});

delayLegend.append("text")
  .attr("x", 0)
  .attr("y", -10)
  .attr("fill", "#000")
  .style("font-size", "12px")
  .text("Median arrival delay");

display(bivarSvg.node());
```
</details>

### Analysis & Insights

<ol class="insight-list">
  <li>Bronx Precinct 44 (Yankee Stadium area) is green even though it has one of the highest call volumes citywide
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Concentrated staffing in this high-crime hotspot has offset workload and kept travel times low. This makes Precinct 44 a useful model for other high-demand precincts that are still struggling with delays
    </div>
  </li>
  <li>The southernmost Brooklyn precinct (61) stands out with a green delay despite being surrounded by red
    <div class="insight">
      <strong>Insight 💡</strong><br>
      This precinct benefits from unusually good road access that connects the area directly to the rest of the city. Its largely residential, mid-rise street grid keeps emergency travel paths short and predictable. Neighboring slow precincts might improve response by tuning deployment/routing to mirror precinct 61
    </div>
  </li>
  <li>High-volume Manhattan corridor shows mostly green/yellow delays despite having high call volumes
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Dense, grid-like street layouts and high unit availability allow Manhattan to absorb extreme demand with minimal delay. Many outer-borough precincts perform worse with far fewer calls, underscoring the role of urban form and deployment density in shaping arrival times
    </div>
  </li>
  <li>Southern Brooklyn (Coney Island, Midwood, Flatbush belts) clusters in red delays even though many precincts are only mid-volume
    <div class="insight">
      <strong>Insight 💡</strong><br>
      This indicates a chronic dispatch congestion zone. These precincts operate under sustained pressure: dense populations, heavy street traffic, and slower routing options
    </div>
  </li>
  <li>Precinct 71 (Brooklyn) shows red delays even though its call volume is lower than surrounding precincts
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Possible contributors include dense mixed-use housing, high pedestrian activity around Eastern Parkway/Flatbush corridors, and frequent serious incidents that tie up units for longer. Improving deployment tactics and cross-precinct support may reduce delays more effectively than simply adding capacity
    </div>
  </li>
  <li>Staten Island shows yellow/red delays despite having some of the lowest call volumes
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Staten Island’s slowdown is structural (long travel distances, fewer units per square mile, and geographic isolation), not volume-driven
    </div>
  </li>
</ol>

## How do borough workloads change throughout the year?

### Temporal Patterns in 911 Call Volume

- Each line tracks call volume over time for every borough, revealing seasonal trends (monthly), recurring cycles (weekly), and day-to-day variability (daily)
- The chart highlights how workload intensity rises or falls across the calendar, helping identify predictable peaks that inform staffing and scheduling

```js
const monthCategory = view(Inputs.radio(["All Categories", ...callCategories], {
  label: "Call set",
  value: "All Categories"
}));

const timeInterval = view(Inputs.radio(["Monthly", "Weekly", "Daily"], {
  label: "Time granularity",
  value: "Monthly"
}));
```

```js
const parseDate = d3.utcParse("%Y-%m-%d");
const intervalKey = timeInterval.toLowerCase();

// Filter for the selected interval and category, then aggregate if "All Categories"
let monthData = boroughSeries
  .filter(d => d.interval === intervalKey)
  .map(d => ({
    ...d,
    date: d.date instanceof Date ? d.date : parseDate(d.date)
  }))
  .filter(d => d.date && d.boro_nm);

if (monthCategory !== "All Categories") {
  monthData = monthData.filter(d => d.category === monthCategory);
} else {
  monthData = d3.rollups(
    monthData,
    v => d3.sum(v, d => d.call_count),
    d => d.boro_nm,
    d => +d.date
  ).flatMap(([boro, entries]) =>
    entries.map(([ts, total]) => ({ boro_nm: boro, date: new Date(Number(ts)), call_count: total }))
  );
}

// Drop trailing partial periods so weekly/daily views don't show partial tails
const maxDate = d3.max(monthData, d => d.date);
const trimDays = timeInterval === "Weekly" ? 7 : timeInterval === "Daily" ? 3 : 0;
if (maxDate && trimDays) {
  const cutoff = d3.utcDay.offset(maxDate, -trimDays);
  monthData = monthData.filter(d => d.date <= cutoff);
}

const monthSeries = d3.groups(monthData, d => d.boro_nm)
  .map(([boro, values]) => ({boro, values: values.sort((a, b) => d3.ascending(a.date, b.date))}));

const monthWidth = 960;
const monthHeight = 500;
const monthMargin = {top: 60, right: 93, bottom: 70, left: 60};
const monthX = d3.scaleUtc()
  .domain(d3.extent(monthData, d => d.date))
  .range([monthMargin.left, monthWidth - monthMargin.right]);
const monthY = d3.scaleLinear()
  .domain([0, d3.max(monthData, d => d.call_count)]).nice()
  .range([monthHeight - monthMargin.bottom, monthMargin.top]);
const monthColor = d3.scaleOrdinal()
  .domain(boroughNames)
  .range(d3.schemeSet2);

const titleCategory = monthCategory === "All Categories" ? "All Categories" : monthCategory;
const titleText = `${timeInterval} · ${titleCategory}`;

const monthSvg = d3.create("svg")
  .attr("viewBox", [0, 0, monthWidth, monthHeight])
  .attr("width", monthWidth)
  .attr("height", monthHeight)
  .style("max-width", "100%")
  .style("height", "auto")
  .style("background", "#dfdfd6");

const monthInnerHeight = monthHeight - monthMargin.top - monthMargin.bottom;
const monthInnerWidth = monthWidth - monthMargin.left - monthMargin.right;
const tickCount = timeInterval === "Daily" ? 8 : timeInterval === "Weekly" ? 10 : 6;
let weekIndex = null;
if (timeInterval === "Weekly") {
  const weeks = Array.from(new Set(monthData.map(d => +d.date))).sort((a, b) => a - b);
  weekIndex = new Map(weeks.map((ts, i) => [ts, i + 1]));
}

monthSvg.append("g")
  .attr("transform", `translate(0, ${monthHeight - monthMargin.bottom})`)
  .call(d3.axisBottom(monthX).ticks(tickCount).tickSize(-monthInnerHeight).tickFormat(""))
  .selectAll("line")
  .attr("stroke", "#bfbfbf")
  .attr("stroke-dasharray", "3,3");

monthSvg.append("g")
  .attr("transform", `translate(${monthMargin.left},0)`)
  .call(d3.axisLeft(monthY).ticks(5).tickSize(-monthInnerWidth).tickFormat(""))
  .selectAll("line")
  .attr("stroke", "#bfbfbf")
  .attr("stroke-dasharray", "3,3");

monthSvg.append("line")
  .attr("x1", monthMargin.left)
  .attr("x2", monthWidth - monthMargin.right)
  .attr("y1", monthHeight - monthMargin.bottom)
  .attr("y2", monthHeight - monthMargin.bottom)
  .attr("stroke", "#000")
  .attr("stroke-width", 1.4);

monthSvg.append("line")
  .attr("x1", monthMargin.left)
  .attr("x2", monthMargin.left)
  .attr("y1", monthMargin.top)
  .attr("y2", monthHeight - monthMargin.bottom)
  .attr("stroke", "#000")
  .attr("stroke-width", 1.4);

monthSvg.append("text")
  .attr("x", monthMargin.left)
  .attr("y", monthMargin.top - 8)
  .attr("fill", "#000")
  .style("font-weight", "700")
  .style("font-size", "16px")
  .text(titleText);

const line = d3.line()
  .x(d => monthX(d.date))
  .y(d => monthY(d.call_count));

const monthPaths = monthSvg.append("g")
  .attr("fill", "none")
  .attr("stroke-width", 2)
  .attr("stroke-linejoin", "round")
  .attr("stroke-linecap", "round")
  .selectAll("path")
  .data(monthSeries)
  .join("path")
  .attr("stroke", d => monthColor(d.boro))
  .attr("d", d => line(d.values));

monthSvg.append("g")
  .attr("transform", `translate(0, ${monthHeight - monthMargin.bottom})`)
  .call(d3.axisBottom(monthX).ticks(tickCount))
  .selectAll("text")
  .style("fill", "#000");

monthSvg.append("g")
  .attr("transform", `translate(${monthMargin.left},0)`)
  .call(d3.axisLeft(monthY).ticks(5, "~s"))
  .selectAll("text")
  .style("fill", "#000");

monthSvg.append("text")
  .attr("x", monthMargin.left + (monthWidth - monthMargin.left - monthMargin.right) / 2)
  .attr("y", monthHeight - monthMargin.bottom / 2)
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .text("Date");

monthSvg.append("text")
  .attr("x", -monthHeight / 2)
  .attr("y", monthMargin.left / 2)
  .attr("transform", "rotate(-90)")
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .text("Calls");

const legendSpacing = 110;
const legendWidth = boroughNames.length * legendSpacing;
const legendX = monthWidth / 2 - legendWidth / 2;
const legendY = 12;
const monthLegend = monthSvg.append("g")
  .attr("transform", `translate(${legendX}, ${legendY})`);

boroughNames.forEach((boro, i) => {
  const g = monthLegend.append("g").attr("transform", `translate(${i * legendSpacing}, 0)`);
  g.append("line")
    .attr("x1", 0)
    .attr("x2", 20)
    .attr("y1", 6)
    .attr("y2", 6)
    .attr("stroke", monthColor(boro))
    .attr("stroke-width", 3);
  g.append("text")
    .attr("x", 26)
    .attr("y", 10)
    .attr("fill", "#000")
    .style("font-size", "12px")
    .text(boro);
});

const dateFmtMonth = d3.timeFormat("%B");
const dateFmtDay = d3.timeFormat("%Y-%m-%d");
const monthPoints = monthData.map(d => {
  const weekNum = weekIndex ? weekIndex.get(+d.date) : null;
  const label = timeInterval === "Weekly"
    ? `Week ${weekNum}`
    : timeInterval === "Monthly"
      ? dateFmtMonth(d.date)
      : dateFmtDay(d.date);
  return [monthX(d.date), monthY(d.call_count), d.boro_nm, label, d.call_count];
});
const monthFocus = monthSvg.append("g").attr("display", "none");
monthFocus.append("circle").attr("r", 5).attr("fill", "#000");
const monthLabel = monthFocus.append("text")
  .attr("text-anchor", "start")
  .attr("x", 9)
  .attr("y", -8)
  .attr("fill", "#000")
  .style("font-size", "12px");

monthSvg
  .on("pointerenter", () => monthFocus.attr("display", null))
  .on("pointermove", event => {
    const [xm, ym] = d3.pointer(event);
    const i = d3.leastIndex(monthPoints, ([x, y]) => Math.hypot(x - xm, y - ym));
    const [x, y, boro, month, count] = monthPoints[i];
    monthFocus.attr("transform", `translate(${x}, ${y})`);
    monthLabel.text(boro);
    monthLabel.selectAll("tspan").remove();
    monthLabel.append("tspan")
      .attr("x", 9)
      .attr("dy", 14)
      .text(month);
    monthLabel.append("tspan")
      .attr("x", 9)
      .attr("dy", 14)
      .text(count.toLocaleString());
    monthPaths.attr("stroke-opacity", d => d.boro === boro ? 1 : 0.25)
      .attr("stroke-width", d => d.boro === boro ? 3.5 : 1.5);
  })
  .on("pointerleave", () => {
    monthFocus.attr("display", "none");
    monthPaths.attr("stroke-opacity", 1).attr("stroke-width", 2);
  });

display(monthSvg.node());
```

<details>
<summary>Code</summary>

```javascript
const monthCategory = view(Inputs.radio(["All Categories", ...callCategories], {
  label: "Call set",
  value: "All Categories"
}));

const parseMonth = d3.utcParse("%Y-%m");
// Monthly rollups for whichever category selection is active
const monthData = boroughMonthly
  .filter(d => monthCategory === "All Categories" || d.category === monthCategory)
  .map(d => ({
    ...d,
    date: d.incident_month instanceof Date ? d.incident_month : parseMonth(d.incident_month)
  }))
  .filter(d => d.date);

const monthSeries = d3.groups(monthData, d => d.boro_nm)
  .map(([boro, values]) => ({boro, values: values.sort((a, b) => d3.ascending(a.date, b.date))}));

const monthWidth = 960;
const monthHeight = 500;
const monthMargin = {top: 60, right: 120, bottom: 70, left: 60};
const monthX = d3.scaleUtc()
  .domain(d3.extent(monthData, d => d.date))
  .range([monthMargin.left, monthWidth - monthMargin.right]);
const monthY = d3.scaleLinear()
  .domain([0, d3.max(monthData, d => d.call_count)]).nice()
  .range([monthHeight - monthMargin.bottom, monthMargin.top]);
const monthColor = d3.scaleOrdinal()
  .domain(boroughNames)
  .range(d3.schemeSet2);

const titleCategory = monthCategory === "All Categories" ? "All Categories" : monthCategory;
const titleText = `${timeInterval} · ${titleCategory}`;

const monthSvg = d3.create("svg")
  .attr("viewBox", [0, 0, monthWidth, monthHeight])
  .attr("width", monthWidth)
  .attr("height", monthHeight)
  .style("max-width", "100%")
  .style("height", "auto")
  .style("background", "#dfdfd6");

const monthInnerHeight = monthHeight - monthMargin.top - monthMargin.bottom;
const monthInnerWidth = monthWidth - monthMargin.left - monthMargin.right;

monthSvg.append("g")
  .attr("transform", `translate(0, ${monthHeight - monthMargin.bottom})`)
  .call(d3.axisBottom(monthX).ticks(6).tickSize(-monthInnerHeight).tickFormat(""))
  .selectAll("line")
  .attr("stroke", "#bfbfbf")
  .attr("stroke-dasharray", "3,3");

monthSvg.append("g")
  .attr("transform", `translate(${monthMargin.left},0)`)
  .call(d3.axisLeft(monthY).ticks(5).tickSize(-monthInnerWidth).tickFormat(""))
  .selectAll("line")
  .attr("stroke", "#bfbfbf")
  .attr("stroke-dasharray", "3,3");

monthSvg.append("line")
  .attr("x1", monthMargin.left)
  .attr("x2", monthWidth - monthMargin.right)
  .attr("y1", monthHeight - monthMargin.bottom)
  .attr("y2", monthHeight - monthMargin.bottom)
  .attr("stroke", "#000")
  .attr("stroke-width", 1.4);

monthSvg.append("line")
  .attr("x1", monthMargin.left)
  .attr("x2", monthMargin.left)
  .attr("y1", monthMargin.top)
  .attr("y2", monthHeight - monthMargin.bottom)
  .attr("stroke", "#000")
  .attr("stroke-width", 1.4);

monthSvg.append("text")
  .attr("x", monthMargin.left)
  .attr("y", monthMargin.top - 8)
  .attr("fill", "#000")
  .style("font-weight", "700")
  .style("font-size", "16px")
  .text(titleText);

const line = d3.line()
  .x(d => monthX(d.date))
  .y(d => monthY(d.call_count));

const monthPaths = monthSvg.append("g")
  .attr("fill", "none")
  .attr("stroke-width", 2)
  .attr("stroke-linejoin", "round")
  .attr("stroke-linecap", "round")
  .selectAll("path")
  .data(monthSeries)
  .join("path")
  .attr("stroke", d => monthColor(d.boro))
  .attr("d", d => line(d.values));

monthSvg.append("g")
  .attr("transform", `translate(0, ${monthHeight - monthMargin.bottom})`)
  .call(d3.axisBottom(monthX).ticks(6))
  .selectAll("text")
  .style("fill", "#000");

monthSvg.append("g")
  .attr("transform", `translate(${monthMargin.left},0)`)
  .call(d3.axisLeft(monthY).ticks(5, "~s"))
  .selectAll("text")
  .style("fill", "#000");

monthSvg.append("text")
  .attr("x", monthMargin.left + (monthWidth - monthMargin.left - monthMargin.right) / 2)
  .attr("y", monthHeight - monthMargin.bottom / 2)
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .text("Date");

monthSvg.append("text")
  .attr("x", -monthHeight / 2)
  .attr("y", monthMargin.left / 2)
  .attr("transform", "rotate(-90)")
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .text("Calls");

const legendSpacing = 110;
const legendWidth = boroughNames.length * legendSpacing;
const legendX = monthWidth / 2 - legendWidth / 2;
const legendY = 12;
const monthLegend = monthSvg.append("g")
  .attr("transform", `translate(${legendX}, ${legendY})`);

boroughNames.forEach((boro, i) => {
  const g = monthLegend.append("g").attr("transform", `translate(${i * legendSpacing}, 0)`);
  g.append("line")
    .attr("x1", 0)
    .attr("x2", 20)
    .attr("y1", 6)
    .attr("y2", 6)
    .attr("stroke", monthColor(boro))
    .attr("stroke-width", 3);
  g.append("text")
    .attr("x", 26)
    .attr("y", 10)
    .attr("fill", "#000")
    .style("font-size", "12px")
    .text(boro);
});

const dateFmtMonth = d3.timeFormat("%B");
const dateFmtDay = d3.timeFormat("%Y-%m-%d");
const monthPoints = monthData.map(d => {
  const weekNum = weekIndex ? weekIndex.get(+d.date) : null;
  const label = timeInterval === "Weekly"
    ? `Week ${weekNum}`
    : timeInterval === "Monthly"
      ? dateFmtMonth(d.date)
      : dateFmtDay(d.date);
  return [monthX(d.date), monthY(d.call_count), d.boro_nm, label, d.call_count];
});
const monthFocus = monthSvg.append("g").attr("display", "none");
monthFocus.append("circle").attr("r", 5).attr("fill", "#000");
const monthLabel = monthFocus.append("text")
  .attr("text-anchor", "start")
  .attr("x", 9)
  .attr("y", -8)
  .attr("fill", "#000")
  .style("font-size", "12px");

monthSvg
  .on("pointerenter", () => monthFocus.attr("display", null))
  .on("pointermove", event => {
    const [xm, ym] = d3.pointer(event);
    const i = d3.leastIndex(monthPoints, ([x, y]) => Math.hypot(x - xm, y - ym));
    const [x, y, boro, month, count] = monthPoints[i];
    monthFocus.attr("transform", `translate(${x}, ${y})`);
    monthLabel.text(boro);
    monthLabel.selectAll("tspan").remove();
    monthLabel.append("tspan")
      .attr("x", 9)
      .attr("dy", 14)
      .text(month);
    monthLabel.append("tspan")
      .attr("x", 9)
      .attr("dy", 14)
      .text(count.toLocaleString());
    monthPaths.attr("stroke-opacity", d => d.boro === boro ? 1 : 0.25)
      .attr("stroke-width", d => d.boro === boro ? 3.5 : 1.5);
  })
  .on("pointerleave", () => {
    monthFocus.attr("display", "none");
    monthPaths.attr("stroke-opacity", 1).attr("stroke-width", 2);
  });

display(monthSvg.node());
```

</details>

### Analysis & Insights

#### Overall pattern

- Stable, repeatable seasonal structure rather than dramatic spikes across all boroughs  
- Volumes rise slightly mid-year, stay steady through fall, and dip at year-end (especially in weekly trends)  
- Demand is driven by routine and high-frequency calls, not episodic surges

<ol class="insight-list">
  <li>Brooklyn consistently sits at the top across monthly, weekly, and daily timelines. It shows a small but steady mid-year increase
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Brooklyn’s workload is persistently high rather than sharply seasonal. Its slight summer uptick reflects increased public activity but the main story is sustained volume. Staffing models need to be built around continuous high baseline demand, not event-driven spikes
    </div>
  </li>
  <li>Manhattan and Queens track closely together, showing moderate increases in mid-year months and stable weekly cycles
    <div class="insight">
      <strong>Insight 💡</strong><br>
      These boroughs exhibit predictable, smooth variations (no abrupt surges). This suggests that their demand is shaped by routine population flow (commuters, business districts, and residential density). This predictability supports regularized scheduling, where shift timing rather than surge capacity becomes key
    </div>
  </li>
  <li>The Bronx shows lower day-to-day volatility compared with Manhattan and Brooklyn
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Bronx call patterns have no large oscillations. This indicates a steady residential-driven workload, where incremental changes can be anticipated and resource adjustments can be planned in advance
    </div>
  </li>
  <li>Staten Island remains the lowest-volume and minimal variation borough at all temporal scales
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Workload in Staten Island is highly stable and low amplitude. Operational needs here are more about coverage and travel time than call spikes, reinforcing earlier conclusions that volume is not the borough’s primary constraint
    </div>
  </li>
  <li>Daily and weekly charts reveal cyclical noise. There is no evidence of extreme surges or crisis-level spikes in any borough
    <div class="insight">
      <strong>Insight 💡</strong><br>
      Despite large population differences, boroughs follow relatively consistent patterns with manageable fluctuations. This suggests a 911 system driven more by routine, high-frequency incidents than by episodic shocks
    </div>
  </li>
</ol>
