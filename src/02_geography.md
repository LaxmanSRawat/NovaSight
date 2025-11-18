# Geographic Analysis

## Which boroughs and precincts experience the highest volume of 911 calls?

```js
// Load spatial files plus the aggregates generated from preprocessing notebooks
const boroughBoundaries = await FileAttachment("nyc-borough-boundaries.geojson").json();
const boroughCounts = (await FileAttachment("call_volume_by_borough.csv").csv({typed: true}))
  .filter(d => d.boro_nm && d.boro_nm !== "(Null)");
const precinctCounts = await FileAttachment("call_volume_by_precinct.csv").csv({typed: true});
const boroughProfiles = (await FileAttachment("borough_response_profile.csv").csv({typed: true}))
  .filter(d => d.boro_nm && d.boro_nm !== "(Null)");
const boroughMonthly = (await FileAttachment("borough_monthly_call_counts.csv").csv({typed: true}))
  .filter(d => d.boro_nm && d.boro_nm !== "(Null)");
const callCategories = Array.from(new Set(boroughCounts.map(d => d.category)));
const boroughNames = Array.from(new Set(boroughProfiles.map(d => d.boro_nm))).sort(d3.ascending);
```

```js echo
const focusOptions = ["All Categories", ...callCategories];
const callCategory = view(Inputs.radio(focusOptions, {
  label: "Focus on call set",
  value: "All Categories"
}));
```

```js echo
const precinctBorough = view(Inputs.select(["Citywide", ...boroughNames], {
  label: "Drill into a specific borough",
  value: "Citywide"
}));
```

## Citywide load vs. service speed

- **Call volume vs. median call duration** drives the axes, while bubble size reflects **arrival delay**.
- Use the controls above to focus on a single call category (or compare both) and spotlight any borough via the selector.

```js echo
// Filter for whichever dataset is active; "All" shows both categories together
const scatterData = callCategory === "All Categories"
  ? boroughProfiles
  : boroughProfiles.filter(d => d.category === callCategory);
const selectedBorough = precinctBorough === "Citywide" ? null : precinctBorough;
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
  .domain([Math.max(1, arrivalExtent[0]), arrivalExtent[1]])
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
  .attr("fill", d => callCategory === "All Categories" ? scatterColor(d.category) : d.category === "Confirmed Crime" ? "#0f6cbd" : "#d1495b")
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

if (callCategory === "All Categories") {
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
  .attr("transform", `translate(0, ${callCategory === "All Categories" ? callCategories.length * 20 + 20 : 12})`);

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

- Brooklyn and Queens dominate the scatter’s x-axis, while Staten Island bubbles remain small and near the origin.
- Potential-crime bubbles (red) tend to sit higher on the y-axis for the same boroughs, showing longer closure times.

## Call intensity across boroughs

```js echo
const mapWidth = 420;
const mapHeight = 520;
const projection = d3.geoMercator().fitSize([mapWidth, mapHeight], boroughBoundaries);
const path = d3.geoPath(projection);
const maxCalls = d3.max(boroughCounts, d => d.call_count);
const mapColor = d3.scaleSequential(d3.interpolateYlOrRd).domain([0, maxCalls]);

// Render confirmed and potential choropleths side-by-side
const grid = d3.create("div")
  .style("display", "grid")
  .style("grid-template-columns", "repeat(auto-fit, minmax(300px, 1fr))")
  .style("gap", "1.5rem");

const categoryColors = d3.scaleOrdinal()
  .domain(callCategories)
  .range(["#f5b041", "#c0392b"]);

for (const category of callCategories) {
  // Each loop emits one SVG tile
  const data = new Map(boroughCounts.filter(d => d.category === category).map(d => [d.boro_nm, d.call_count]));
  const svg = grid.append("svg")
    .attr("viewBox", [0, 0, mapWidth, mapHeight])
    .attr("width", mapWidth)
    .attr("height", mapHeight)
    .style("max-width", "100%")
    .style("height", "auto")
    .style("background", "#dfdfd6");

  svg.append("text")
    .attr("x", mapWidth / 2)
    .attr("y", 28)
    .attr("text-anchor", "middle")
    .attr("fill", "#000")
    .style("font-size", "18px")
    .style("font-weight", "600")
    .text(category);

  svg.append("g")
    .attr("transform", "translate(0, 40)")
    .selectAll("path")
    .data(boroughBoundaries.features)
    .join("path")
    .attr("d", path)
    .attr("fill", d => mapColor(data.get(d.properties.BoroName) || 0))
    .attr("stroke", "#ffffff")
    .attr("stroke-width", 1.2)
    .append("title")
    .text(d => `${d.properties.BoroName}
${(data.get(d.properties.BoroName) || 0).toLocaleString()} calls`);
}

const legendWidth = 260;
const legendHeight = 14;
const legendScale = d3.scaleLinear().domain(mapColor.domain()).range([0, legendWidth]);
const defs = grid.append("svg")
  .attr("viewBox", [0, 0, legendWidth + 80, 60])
  .style("max-width", "100%")
  .style("height", "auto");

const gradientId = "mapGradient";
const gradient = defs.append("defs").append("linearGradient")
  .attr("id", gradientId)
  .attr("x1", "0%").attr("x2", "100%")
  .attr("y1", "0%").attr("y2", "0%");

for (let i = 0; i <= 10; i++) {
  const t = i / 10;
  gradient.append("stop")
    .attr("offset", `${t * 100}%`)
    .attr("stop-color", mapColor(legendScale.invert(t * legendWidth)));
}

defs.append("rect")
  .attr("x", 20)
  .attr("y", 10)
  .attr("width", legendWidth)
  .attr("height", legendHeight)
  .attr("fill", `url(#${gradientId})`)
  .attr("stroke", "#333");

defs.append("g")
  .attr("transform", `translate(20, ${10 + legendHeight})`)
  .call(d3.axisBottom(legendScale).ticks(5, "~s"))
  .selectAll("text")
  .style("fill", "#fff");

defs.append("text")
  .attr("x", 20)
  .attr("y", 56.5)
  .attr("fill", "#fff")
  .style("font-size", "12px")
  .text("Call volume per borough");

display(grid.node());
```

- Brooklyn stays the most saturated panel for both categories; Queens and the Bronx grow noticeably hotter in the potential-crime view.
- Staten Island’s panel remains light, indicating its lower share of calls regardless of dataset.

## Where is the mix shifting?

- Visualizes **potential minus confirmed** call share, so positive areas lean toward potential-crime workload.
- Values near zero mean similar shares; darker hues indicate boroughs with noticeable imbalances.

```js echo
// Compute share differences between potential and confirmed loads
// Positive values indicate places leaning toward potential calls
const shareMatrix = d3.rollup(
  boroughCounts,
  ([d]) => d.share,
  d => d.category,
  d => d.boro_nm
);
const confirmedShares = shareMatrix.get("Confirmed Crime") || new Map();
const potentialShares = shareMatrix.get("Potential Crime") || new Map();
const shareDiff = new Map(
  boroughNames.map(name => [name, (potentialShares.get(name) || 0) - (confirmedShares.get(name) || 0)])
);
const diffExtent = d3.max([...shareDiff.values()].map(Math.abs));
const diffColor = d3.scaleDiverging(d3.interpolateRdBu).domain([-diffExtent, 0, diffExtent]);

const diffWidth = 700;
const diffHeight = 520;
const diffSvg = d3.create("svg")
  .attr("viewBox", [0, 0, diffWidth, diffHeight])
  .attr("width", diffWidth)
  .attr("height", diffHeight)
  .style("max-width", "100%")
  .style("height", "auto")
  .style("background", "#dfdfd6");

diffSvg.append("text")
  .attr("x", diffWidth / 2)
  .attr("y", 32)
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-weight", "600")
  .text("Potential minus confirmed call share");

const diffGroup = diffSvg.append("g")
  .attr("transform", "translate(55, 31)")
  .selectAll("path")
  .data(boroughBoundaries.features)
  .join("path")
  .attr("d", path)
  .attr("fill", d => diffColor(shareDiff.get(d.properties.BoroName) || 0))
  .attr("stroke", "#fff")
  .attr("stroke-width", 1.2)
  .append("title")
  .text(d => {
    const diff = shareDiff.get(d.properties.BoroName) || 0;
    const pct = d3.format("+.1%") (diff);
    return `${d.properties.BoroName}
Potential share change: ${pct}`;
  });

const diffLegendHeight = 180;
const diffScale = d3.scaleLinear().domain([-diffExtent, diffExtent]).range([diffLegendHeight, 0]);
const diffGradient = diffSvg.append("defs").append("linearGradient")
  .attr("id", "diffGradient")
  .attr("x1", "0%").attr("x2", "0%")
  .attr("y1", "100%").attr("y2", "0%");

for (let i = 0; i <= 10; i++) {
  const t = i / 10;
  const value = diffScale.invert(diffLegendHeight * (1 - t));
  diffGradient.append("stop")
    .attr("offset", `${t * 100}%`)
    .attr("stop-color", diffColor(value));
}

const legendGroup = diffSvg.append("g")
  .attr("transform", `translate(${diffWidth - 80}, ${80})`);

legendGroup.append("rect")
  .attr("width", 18)
  .attr("height", diffLegendHeight)
  .attr("fill", "url(#diffGradient)")
  .attr("stroke", "#333");

// Axis labels to show exact percentage shifts
legendGroup.append("g")
  .attr("transform", "translate(18, 0)")
  .call(d3.axisRight(diffScale).ticks(5, "+.0%"))
  .selectAll("text")
  .style("fill", "#000");

legendGroup.append("text")
  .attr("x", 9)
  .attr("y", -12)
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-size", "12px")
  .text("Potential heavy");

legendGroup.append("text")
  .attr("x", 9)
  .attr("y", diffLegendHeight + 16)
  .attr("text-anchor", "middle")
  .attr("fill", "#000")
  .style("font-size", "12px")
  .text("Confirmed heavy");

display(diffSvg.node());
```

- Queens and the Bronx show positive share differences, signalling that potential-crime calls make up a larger slice of their workload.
- Manhattan and Staten Island go negative, meaning confirmed incidents still dominate their mix.

## Drill into precinct workloads

- Use the borough selector to focus on a specific area (or stay on “Citywide”) and inspect the busiest precincts.
- The chart refreshes automatically as you toggle between confirmed vs. potential calls.

```js echo
// Build the top precinct list for the active category/borough filter
const precinctData = precinctCounts
  .filter(d => (callCategory === "All Categories" || d.category === callCategory)
    && (precinctBorough === "Citywide" || d.boro_nm === precinctBorough))
  .sort((a, b) => d3.descending(a.call_count, b.call_count))
  .slice(0, 12);

const precMargin = {top: 35, right: 30, bottom: 30, left: 220};
const precWidth = 960;
const precHeight = Math.max(precinctData.length * 28 + precMargin.top + precMargin.bottom, 260);
const precX = d3.scaleLinear()
  .domain([0, d3.max(precinctData, d => d.call_count)]).nice()
  .range([precMargin.left, precWidth - precMargin.right]);
const precY = d3.scaleBand()
  .domain(precinctData.map(d => `${d.nypd_pct_cd} • ${d.boro_nm}`))
  .range([precMargin.top, precHeight - precMargin.bottom])
  .padding(0.25);

const precSvg = d3.create("svg")
  .attr("viewBox", [0, 0, precWidth, precHeight])
  .attr("width", precWidth)
  .attr("height", precHeight)
  .style("max-width", "100%")
  .style("height", "auto")
  .style("background", "#dfdfd6");

const precInnerHeight = precHeight - precMargin.top - precMargin.bottom;
precSvg.append("g")
  .attr("transform", `translate(0, ${precHeight - precMargin.bottom})`)
  .call(d3.axisBottom(precX).ticks(6, "~s").tickSize(-precInnerHeight).tickFormat(""))
  .selectAll("line")
  .attr("stroke", "#bfbfbf")
  .attr("stroke-dasharray", "3,3");

precSvg.append("line")
  .attr("x1", precMargin.left)
  .attr("x2", precWidth - precMargin.right)
  .attr("y1", precHeight - precMargin.bottom)
  .attr("y2", precHeight - precMargin.bottom)
  .attr("stroke", "#000")
  .attr("stroke-width", 1.3);

precSvg.append("line")
  .attr("x1", precMargin.left)
  .attr("x2", precMargin.left)
  .attr("y1", precMargin.top)
  .attr("y2", precHeight - precMargin.bottom)
  .attr("stroke", "#000")
  .attr("stroke-width", 1.3);

precSvg.append("g")
  .selectAll("rect")
  .data(precinctData)
  .join("rect")
  .attr("x", precMargin.left)
  .attr("y", d => precY(`${d.nypd_pct_cd} • ${d.boro_nm}`))
  .attr("width", d => precX(d.call_count) - precMargin.left)
  .attr("height", precY.bandwidth())
  .attr("fill", "#0f6cbd")
  .append("title")
  .text(d => `${d.nypd_pct_cd} (${d.boro_nm})
${d.call_count.toLocaleString()} calls`);

precSvg.append("g")
  .attr("transform", `translate(0, ${precHeight - precMargin.bottom})`)
  .call(d3.axisBottom(precX).ticks(6, "~s"))
  .selectAll("text")
  .style("fill", "#000");

precSvg.append("g")
  .attr("transform", `translate(${precMargin.left},0)`)
  .call(d3.axisLeft(precY))
  .selectAll("text")
  .style("fill", "#000");

precSvg.append("text")
  .attr("x", precMargin.left)
  .attr("y", precMargin.top - 10)
  .attr("fill", "#000")
  .style("font-weight", "600")
  .text(precinctBorough === "Citywide"
    ? "Top precincts citywide"
    : `Top precincts in ${precinctBorough}`);

display(precSvg.node());
```
 
- Citywide view keeps Brooklyn’s 75th precinct at the top, with several Bronx precincts close behind.
- Switching the selector to a single borough immediately reveals that borough’s busiest precincts—for example, Queens highlights precincts 114 and 109.

## When do borough workloads spike?

```js echo
const parseMonth = d3.utcParse("%Y-%m");
// Monthly rollups for whichever category selection is active
const monthData = boroughMonthly
  .filter(d => callCategory === "All Categories" || d.category === callCategory)
  .map(d => ({
    ...d,
    date: d.incident_month instanceof Date ? d.incident_month : parseMonth(d.incident_month)
  }))
  .filter(d => d.date);

const monthSeries = d3.groups(monthData, d => d.boro_nm)
  .map(([boro, values]) => ({boro, values: values.sort((a, b) => d3.ascending(a.date, b.date))}));

const monthWidth = 960;
const monthHeight = 500;
const monthMargin = {top: 30, right: 20, bottom: 40, left: 60};
const monthX = d3.scaleUtc()
  .domain(d3.extent(monthData, d => d.date))
  .range([monthMargin.left, monthWidth - monthMargin.right]);
const monthY = d3.scaleLinear()
  .domain([0, d3.max(monthData, d => d.call_count)]).nice()
  .range([monthHeight - monthMargin.bottom, monthMargin.top]);
const monthColor = d3.scaleOrdinal()
  .domain(boroughNames)
  .range(d3.schemeSet2);

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

const monthLegend = monthSvg.append("g")
  .attr("transform", `translate(${monthMargin.left}, ${monthMargin.top})`);

boroughNames.forEach((boro, i) => {
  const g = monthLegend.append("g").attr("transform", `translate(${(i % 3) * 180}, ${Math.floor(i / 3) * 18})`);
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

const monthPoints = monthData.map(d => [monthX(d.date), monthY(d.call_count), d.boro_nm, d.incident_month, d.call_count]);
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
    monthLabel.text(`${boro} · ${month}: ${count.toLocaleString()}`);
    monthPaths.attr("stroke-opacity", d => d.boro === boro ? 1 : 0.25)
      .attr("stroke-width", d => d.boro === boro ? 3.5 : 1.5);
  })
  .on("pointerleave", () => {
    monthFocus.attr("display", "none");
    monthPaths.attr("stroke-opacity", 1).attr("stroke-width", 2);
  });

display(monthSvg.node());
```

- Brooklyn and Manhattan peak early in the year and again during summer, while Queens and the Bronx ramp steadily into late summer.
- Staten Island’s line stays comparatively flat, echoing the low-volume story from earlier charts.
