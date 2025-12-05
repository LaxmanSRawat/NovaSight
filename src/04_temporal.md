---
style: text-style.css
---

<!-- Questions and Findings - For each question:
Clear question statement
Polished D3 visualization
Analysis and interpretation
Insights and implications -->

# The Rhythm of Emergency: Temporal Insights in 911 calls
<br>

## Which Incident takes the longest time, and which incidents are resolved quickly? 

  ```js 
  const focusOptions = ["Crime", "Potential Crime"];
  const callCategory = view(Inputs.radio(focusOptions, {
    label: "Focus on call set",
    value: "Crime"
  }));
  ```
<br>

<details>
  <summary> Data Loading </summary>

  Loading the top 10 crime and potential crime category by median of call duration (data processed using <a href="https://github.com/LaxmanSRawat/NovaSight/blob/main/temporal_analysis_top_10_category_by_average_total_duration.ipynb" rel="external">python jupyter notebook) </a>

  ```js echo
  file
  ```

  ```js echo
  // Load top 10 crime category by call duration data
  const crime_top_10_categories_by_incident_duration = FileAttachment("nyc_data_crime_top_10_categories_by_call_duration.csv").csv({typed: true})
  const potential_crime_top_10_categories_by_incident_duration = FileAttachment("nyc_data_potential_crime_top_10_categories_by_call_duration.csv").csv({typed: true})
  const crime_bottom_10_categories_by_incident_duration = FileAttachment("nyc_data_crime_bottom_10_categories_by_call_duration.csv").csv({typed: true})
  const potential_crime_bottom_10_categories_by_incident_duration = FileAttachment("nyc_data_potential_crime_bottom_10_categories_by_call_duration.csv").csv({typed: true})
  
   const data = callCategory == "Crime"
  ? crime_top_10_categories_by_incident_duration
  : potential_crime_top_10_categories_by_incident_duration;

  const data_b10 = callCategory == "Crime"
  ? crime_bottom_10_categories_by_incident_duration
  : potential_crime_bottom_10_categories_by_incident_duration;

  ```

  ```js echo
    data
  ```
</details>



<br>

<!-- PLOT 1 -->
```js 
// Set up dimensions
const margin = { top: 50, right: 80, bottom: 70, left: callCategory === "Crime" ? 330:380 };
const width = 900 - margin.left - margin.right;
const height = 500 - margin.top - margin.bottom;

// Group data by typ_desc
const groupedData = d3.group(data, d => d.typ_desc);
const boxPlotData = Array.from(groupedData, ([key, values]) => ({
  type: key,
  stats: calculateBoxPlotStats(values.map(d => d.call_duration_minutes)),
  values: values.map(d => d.call_duration_minutes)
}));

// Sort by median in descending order
boxPlotData.sort((a, b) => b.stats.median - a.stats.median);

// Create SVG with gradient background
const svg = d3.create("svg")
  .attr("width", width + margin.left + margin.right)
  .attr("height", height + margin.top + margin.bottom)
  .attr("viewBox", [0, 0, width + margin.left + margin.right, height + margin.top + margin.bottom])
  .style("max-width", "100%")
  .style("height", "auto");

// Add gradient definition
const defs = svg.append("defs");
const gradient = defs.append("linearGradient")
  .attr("id", "bgGradient")
  .attr("x1", "0%")
  .attr("y1", "0%")
  .attr("x2", "0%")
  .attr("y2", "100%");

gradient.append("stop")
  .attr("offset", "0%")
  .attr("stop-color", "#f8f9fa");

gradient.append("stop")
  .attr("offset", "100%")
  .attr("stop-color", "#e9ecef");

// Apply gradient background
svg.append("rect")
  .attr("width", "100%")
  .attr("height", "100%")
  .attr("fill", "url(#bgGradient)");

const g = svg.append("g")
  .attr("transform", `translate(${margin.left},${margin.top})`);

// Create scales
const yScale = d3.scaleBand()
  .domain(boxPlotData.map(d => d.type))
  .range([0, height])
  .padding(0.35);

const xScale = d3.scaleLinear()
  .domain([0, d3.max(boxPlotData, d => d3.max([d.stats.max, ...d.stats.outliers]))])
  .nice()
  .range([0, width]);

// Add subtle grid lines
g.append("g")
  .attr("class", "grid")
  .attr("transform", `translate(0,${height})`)
  .call(d3.axisBottom(xScale)
    .tickSize(-height)
    .tickFormat("")
  )
  .style("stroke", "#7089a3ff")
  .style("stroke-opacity", 0.8)
  .style("stroke-dasharray", "3,3");

// Create axes with improved styling
g.append("g")
  .attr("class", "x-axis")
  .attr("transform", `translate(0,${height})`)
  .call(d3.axisBottom(xScale).ticks(8))
  .selectAll("text")
  .style("font-size", "13px")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif")
  .style("fill", "#495057")
  .style("font-weight", "500");

g.select(".x-axis")
  .select(".domain")
  .style("stroke", "#adb5bd")
  .style("stroke-width", "2px");

g.select(".x-axis")
  .selectAll(".tick line")
  .style("stroke", "#adb5bd");

g.append("g")
  .attr("class", "y-axis")
  .call(d3.axisLeft(yScale))
  .selectAll("text")
  .style("font-size", "12px")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif")
  .style("fill", "#343a40")
  .style("font-weight", "500");

g.select(".y-axis")
  .select(".domain")
  .style("stroke", "#adb5bd")
  .style("stroke-width", "2px");

g.select(".y-axis")
  .selectAll(".tick line")
  .style("stroke", "#adb5bd");

// Add axis labels with improved styling
g.append("text")
  .attr("text-anchor", "middle")
  .attr("x", width / 2)
  .attr("y", height + margin.bottom - 15)
  .style("font-size", "15px")
  .style("font-weight", "600")
  .style("letter-spacing", "0.5px")
  .text("INCIDENT DURATION (MINUTES)")
  .style("fill", "#212529")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

g.append("text")
  .attr("text-anchor", "middle")
  .attr("transform", "rotate(-90)")
  .attr("x", -height / 2)
  .attr("y", -margin.left + 20)
  .style("font-size", "15px")
  .style("font-weight", "600")
  .style("letter-spacing", "0.5px")
  .text("INCIDENT TYPE")
  .style("fill", "#212529")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

// Add title
g.append("text")
  .attr("text-anchor", "middle")
  .attr("x", width / 2)
  .attr("y", -20)
  .style("font-size", "18px")
  .style("font-weight", "700")
  .style("letter-spacing", "0.5px")
  .text("Top 10 Incident by Avg. (Median) Incident Duration")
  .style("fill", "#212529")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

// Color scale for boxes
const colorScale = d3.scaleSequential(d3.interpolateViridis)
  .domain([boxPlotData.length - 1, 0]);

// Draw box plots with enhanced styling
const boxHeight = yScale.bandwidth();

boxPlotData.forEach((d, i) => {
  const y = yScale(d.type);
  const center = y + boxHeight / 2;
  const boxColor = colorScale(i);

  const boxGroup = g.append("g");

  // Horizontal line from min to Q1
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.min))
    .attr("x2", xScale(d.stats.q1))
    .attr("y1", center)
    .attr("y2", center)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2);

  // Horizontal line from Q3 to max
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.q3))
    .attr("x2", xScale(d.stats.max))
    .attr("y1", center)
    .attr("y2", center)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2);

  // Min vertical line (whisker cap)
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.min))
    .attr("x2", xScale(d.stats.min))
    .attr("y1", center - boxHeight / 3.5)
    .attr("y2", center + boxHeight / 3.5)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2.5)
    .attr("stroke-linecap", "round");

  // Max vertical line (whisker cap)
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.max))
    .attr("x2", xScale(d.stats.max))
    .attr("y1", center - boxHeight / 3.5)
    .attr("y2", center + boxHeight / 3.5)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2.5)
    .attr("stroke-linecap", "round");

  // Box shadow
  boxGroup.append("rect")
    .attr("x", xScale(d.stats.q1) + 3)
    .attr("y", y + 3)
    .attr("width", xScale(d.stats.q3) - xScale(d.stats.q1))
    .attr("height", boxHeight)
    .attr("fill", "#000")
    .attr("opacity", 0.15)
    .attr("rx", 4);

  // Box (IQR) with gradient
  const boxGradient = defs.append("linearGradient")
    .attr("id", `boxGradient${i}`)
    .attr("x1", "0%")
    .attr("y1", "0%")
    .attr("x2", "0%")
    .attr("y2", "100%");

  boxGradient.append("stop")
    .attr("offset", "0%")
    .attr("stop-color", d3.color(boxColor).brighter(0.3));

  boxGradient.append("stop")
    .attr("offset", "100%")
    .attr("stop-color", boxColor);

  boxGroup.append("rect")
    .attr("x", xScale(d.stats.q1))
    .attr("y", y)
    .attr("width", xScale(d.stats.q3) - xScale(d.stats.q1))
    .attr("height", boxHeight)
    .attr("fill", `url(#boxGradient${i})`)
    .attr("stroke", d3.color(boxColor).darker(0.5))
    .attr("stroke-width", 2)
    .attr("rx", 4);

  // Median line with glow effect
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.median))
    .attr("x2", xScale(d.stats.median))
    .attr("y1", y)
    .attr("y2", y + boxHeight)
    .attr("stroke", "#fff")
    .attr("stroke-width", 4)
    .attr("opacity", 0.8);

  boxGroup.append("line")
    .attr("x1", xScale(d.stats.median))
    .attr("x2", xScale(d.stats.median))
    .attr("y1", y)
    .attr("y2", y + boxHeight)
    .attr("stroke", "#212529")
    .attr("stroke-width", 2.5);

  // Outliers with better styling
  d.stats.outliers.forEach(outlier => {
    boxGroup.append("circle")
      .attr("cx", xScale(outlier))
      .attr("cy", center)
      .attr("r", 4.5)
      .attr("fill", "#dc3545")
      .attr("stroke", "#fff")
      .attr("stroke-width", 1.5)
      .attr("opacity", 0.8);
  });
});

display(svg.node());
```


```js 
// Set up dimensions
const margin = { top: 50, right: 80, bottom: 70, left: callCategory === "Crime" ? 330:380 };
const width = 900 - margin.left - margin.right;
const height = 500 - margin.top - margin.bottom;

// Group data by typ_desc
const groupedData = d3.group(data_b10, d => d.typ_desc);
const boxPlotData = Array.from(groupedData, ([key, values]) => ({
  type: key,
  stats: calculateBoxPlotStats(values.map(d => d.call_duration_minutes)),
  values: values.map(d => d.call_duration_minutes)
}));

// Sort by median in descending order
boxPlotData.sort((a, b) => b.stats.median - a.stats.median);

// Create SVG with gradient background
const svg = d3.create("svg")
  .attr("width", width + margin.left + margin.right)
  .attr("height", height + margin.top + margin.bottom)
  .attr("viewBox", [0, 0, width + margin.left + margin.right, height + margin.top + margin.bottom])
  .style("max-width", "100%")
  .style("height", "auto");

// Add gradient definition
const defs = svg.append("defs");
const gradient = defs.append("linearGradient")
  .attr("id", "bgGradient")
  .attr("x1", "0%")
  .attr("y1", "0%")
  .attr("x2", "0%")
  .attr("y2", "100%");

gradient.append("stop")
  .attr("offset", "0%")
  .attr("stop-color", "#f8f9fa");

gradient.append("stop")
  .attr("offset", "100%")
  .attr("stop-color", "#e9ecef");

// Apply gradient background
svg.append("rect")
  .attr("width", "100%")
  .attr("height", "100%")
  .attr("fill", "url(#bgGradient)");

const g = svg.append("g")
  .attr("transform", `translate(${margin.left},${margin.top})`);

// Create scales
const yScale = d3.scaleBand()
  .domain(boxPlotData.map(d => d.type))
  .range([0, height])
  .padding(0.35);

const xScale = d3.scaleLinear()
  .domain([0, d3.max(boxPlotData, d => d3.max([d.stats.max, ...d.stats.outliers]))])
  .nice()
  .range([0, width]);

// Add subtle grid lines
g.append("g")
  .attr("class", "grid")
  .attr("transform", `translate(0,${height})`)
  .call(d3.axisBottom(xScale)
    .tickSize(-height)
    .tickFormat("")
  )
  .style("stroke", "#7089a3ff")
  .style("stroke-opacity", 0.8)
  .style("stroke-dasharray", "3,3");

// Create axes with improved styling
g.append("g")
  .attr("class", "x-axis")
  .attr("transform", `translate(0,${height})`)
  .call(d3.axisBottom(xScale).ticks(8))
  .selectAll("text")
  .style("font-size", "13px")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif")
  .style("fill", "#495057")
  .style("font-weight", "500");

g.select(".x-axis")
  .select(".domain")
  .style("stroke", "#adb5bd")
  .style("stroke-width", "2px");

g.select(".x-axis")
  .selectAll(".tick line")
  .style("stroke", "#adb5bd");

g.append("g")
  .attr("class", "y-axis")
  .call(d3.axisLeft(yScale))
  .selectAll("text")
  .style("font-size", "12px")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif")
  .style("fill", "#343a40")
  .style("font-weight", "500");

g.select(".y-axis")
  .select(".domain")
  .style("stroke", "#adb5bd")
  .style("stroke-width", "2px");

g.select(".y-axis")
  .selectAll(".tick line")
  .style("stroke", "#adb5bd");

// Add axis labels with improved styling
g.append("text")
  .attr("text-anchor", "middle")
  .attr("x", width / 2)
  .attr("y", height + margin.bottom - 15)
  .style("font-size", "15px")
  .style("font-weight", "600")
  .style("letter-spacing", "0.5px")
  .text("INCIDENT DURATION (MINUTES)")
  .style("fill", "#212529")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

g.append("text")
  .attr("text-anchor", "middle")
  .attr("transform", "rotate(-90)")
  .attr("x", -height / 2)
  .attr("y", -margin.left + 20)
  .style("font-size", "15px")
  .style("font-weight", "600")
  .style("letter-spacing", "0.5px")
  .text("INCIDENT TYPE")
  .style("fill", "#212529")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

// Add title
g.append("text")
  .attr("text-anchor", "middle")
  .attr("x", width / 2)
  .attr("y", -20)
  .style("font-size", "18px")
  .style("font-weight", "700")
  .style("letter-spacing", "0.5px")
  .text("Bottom 10 Incident by Avg. (Median) Incident Duration")
  .style("fill", "#212529")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

// Color scale for boxes
const colorScale = d3.scaleSequential(d3.interpolateViridis)
  .domain([boxPlotData.length - 1, 0]);

// Draw box plots with enhanced styling
const boxHeight = yScale.bandwidth();

boxPlotData.forEach((d, i) => {
  const y = yScale(d.type);
  const center = y + boxHeight / 2;
  const boxColor = colorScale(i);

  const boxGroup = g.append("g");

  // Horizontal line from min to Q1
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.min))
    .attr("x2", xScale(d.stats.q1))
    .attr("y1", center)
    .attr("y2", center)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2);

  // Horizontal line from Q3 to max
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.q3))
    .attr("x2", xScale(d.stats.max))
    .attr("y1", center)
    .attr("y2", center)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2);

  // Min vertical line (whisker cap)
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.min))
    .attr("x2", xScale(d.stats.min))
    .attr("y1", center - boxHeight / 3.5)
    .attr("y2", center + boxHeight / 3.5)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2.5)
    .attr("stroke-linecap", "round");

  // Max vertical line (whisker cap)
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.max))
    .attr("x2", xScale(d.stats.max))
    .attr("y1", center - boxHeight / 3.5)
    .attr("y2", center + boxHeight / 3.5)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2.5)
    .attr("stroke-linecap", "round");

  // Box shadow
  boxGroup.append("rect")
    .attr("x", xScale(d.stats.q1) + 3)
    .attr("y", y + 3)
    .attr("width", xScale(d.stats.q3) - xScale(d.stats.q1))
    .attr("height", boxHeight)
    .attr("fill", "#000")
    .attr("opacity", 0.15)
    .attr("rx", 4);

  // Box (IQR) with gradient
  const boxGradient = defs.append("linearGradient")
    .attr("id", `boxGradient${i}`)
    .attr("x1", "0%")
    .attr("y1", "0%")
    .attr("x2", "0%")
    .attr("y2", "100%");

  boxGradient.append("stop")
    .attr("offset", "0%")
    .attr("stop-color", d3.color(boxColor).brighter(0.3));

  boxGradient.append("stop")
    .attr("offset", "100%")
    .attr("stop-color", boxColor);

  boxGroup.append("rect")
    .attr("x", xScale(d.stats.q1))
    .attr("y", y)
    .attr("width", xScale(d.stats.q3) - xScale(d.stats.q1))
    .attr("height", boxHeight)
    .attr("fill", `url(#boxGradient${i})`)
    .attr("stroke", d3.color(boxColor).darker(0.5))
    .attr("stroke-width", 2)
    .attr("rx", 4);

  // Median line with glow effect
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.median))
    .attr("x2", xScale(d.stats.median))
    .attr("y1", y)
    .attr("y2", y + boxHeight)
    .attr("stroke", "#fff")
    .attr("stroke-width", 4)
    .attr("opacity", 0.8);

  boxGroup.append("line")
    .attr("x1", xScale(d.stats.median))
    .attr("x2", xScale(d.stats.median))
    .attr("y1", y)
    .attr("y2", y + boxHeight)
    .attr("stroke", "#212529")
    .attr("stroke-width", 2.5);

  // Outliers with better styling
  d.stats.outliers.forEach(outlier => {
    boxGroup.append("circle")
      .attr("cx", xScale(outlier))
      .attr("cy", center)
      .attr("r", 4.5)
      .attr("fill", "#dc3545")
      .attr("stroke", "#fff")
      .attr("stroke-width", 1.5)
      .attr("opacity", 0.8);
  });
});

display(svg.node());
```

<details>
<summary> Code </summary>

```js echo
// Set up dimensions
const margin = { top: 50, right: 80, bottom: 70, left: 330 };
const width = 900 - margin.left - margin.right;
const height = 500 - margin.top - margin.bottom;

// Group data by typ_desc
const groupedData = d3.group(data, d => d.typ_desc);
const boxPlotData = Array.from(groupedData, ([key, values]) => ({
  type: key,
  stats: calculateBoxPlotStats(values.map(d => d.call_duration_minutes)),
  values: values.map(d => d.call_duration_minutes)
}));

// Sort by median in descending order
boxPlotData.sort((a, b) => b.stats.median - a.stats.median);

// Create SVG with gradient background
const svg = d3.create("svg")
  .attr("width", width + margin.left + margin.right)
  .attr("height", height + margin.top + margin.bottom)
  .attr("viewBox", [0, 0, width + margin.left + margin.right, height + margin.top + margin.bottom])
  .style("max-width", "100%")
  .style("height", "auto");

// Add gradient definition
const defs = svg.append("defs");
const gradient = defs.append("linearGradient")
  .attr("id", "bgGradient")
  .attr("x1", "0%")
  .attr("y1", "0%")
  .attr("x2", "0%")
  .attr("y2", "100%");

gradient.append("stop")
  .attr("offset", "0%")
  .attr("stop-color", "#f8f9fa");

gradient.append("stop")
  .attr("offset", "100%")
  .attr("stop-color", "#e9ecef");

// Apply gradient background
svg.append("rect")
  .attr("width", "100%")
  .attr("height", "100%")
  .attr("fill", "url(#bgGradient)");

const g = svg.append("g")
  .attr("transform", `translate(${margin.left},${margin.top})`);

// Create scales
const yScale = d3.scaleBand()
  .domain(boxPlotData.map(d => d.type))
  .range([0, height])
  .padding(0.35);

const xScale = d3.scaleLinear()
  .domain([0, d3.max(boxPlotData, d => d3.max([d.stats.max, ...d.stats.outliers]))])
  .nice()
  .range([0, width]);

// Add subtle grid lines
g.append("g")
  .attr("class", "grid")
  .attr("transform", `translate(0,${height})`)
  .call(d3.axisBottom(xScale)
    .tickSize(-height)
    .tickFormat("")
  )
  .style("stroke", "#dee2e6")
  .style("stroke-opacity", 0.3)
  .style("stroke-dasharray", "3,3");

// Create axes with improved styling
g.append("g")
  .attr("class", "x-axis")
  .attr("transform", `translate(0,${height})`)
  .call(d3.axisBottom(xScale).ticks(8))
  .selectAll("text")
  .style("font-size", "13px")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif")
  .style("fill", "#495057")
  .style("font-weight", "500");

g.select(".x-axis")
  .select(".domain")
  .style("stroke", "#adb5bd")
  .style("stroke-width", "2px");

g.select(".x-axis")
  .selectAll(".tick line")
  .style("stroke", "#adb5bd");

g.append("g")
  .attr("class", "y-axis")
  .call(d3.axisLeft(yScale))
  .selectAll("text")
  .style("font-size", "12px")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif")
  .style("fill", "#343a40")
  .style("font-weight", "500");

g.select(".y-axis")
  .select(".domain")
  .style("stroke", "#adb5bd")
  .style("stroke-width", "2px");

g.select(".y-axis")
  .selectAll(".tick line")
  .style("stroke", "#adb5bd");

// Add axis labels with improved styling
g.append("text")
  .attr("text-anchor", "middle")
  .attr("x", width / 2)
  .attr("y", height + margin.bottom - 15)
  .style("font-size", "15px")
  .style("font-weight", "600")
  .style("letter-spacing", "0.5px")
  .text("INCIDENT DURATION (MINUTES)")
  .style("fill", "#212529")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

g.append("text")
  .attr("text-anchor", "middle")
  .attr("transform", "rotate(-90)")
  .attr("x", -height / 2)
  .attr("y", -margin.left + 20)
  .style("font-size", "15px")
  .style("font-weight", "600")
  .style("letter-spacing", "0.5px")
  .text("INCIDENT TYPE")
  .style("fill", "#212529")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

// Add title
g.append("text")
  .attr("text-anchor", "middle")
  .attr("x", width / 2)
  .attr("y", -20)
  .style("font-size", "18px")
  .style("font-weight", "700")
  .style("letter-spacing", "0.5px")
  .text("Top 10 Incident by Avg. (Median) Incident Duration")
  .style("fill", "#212529")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

// Color scale for boxes
const colorScale = d3.scaleSequential(d3.interpolateViridis)
  .domain([boxPlotData.length - 1, 0]);

// Draw box plots with enhanced styling
const boxHeight = yScale.bandwidth();

boxPlotData.forEach((d, i) => {
  const y = yScale(d.type);
  const center = y + boxHeight / 2;
  const boxColor = colorScale(i);

  const boxGroup = g.append("g");

  // Horizontal line from min to Q1
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.min))
    .attr("x2", xScale(d.stats.q1))
    .attr("y1", center)
    .attr("y2", center)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2);

  // Horizontal line from Q3 to max
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.q3))
    .attr("x2", xScale(d.stats.max))
    .attr("y1", center)
    .attr("y2", center)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2);

  // Min vertical line (whisker cap)
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.min))
    .attr("x2", xScale(d.stats.min))
    .attr("y1", center - boxHeight / 3.5)
    .attr("y2", center + boxHeight / 3.5)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2.5)
    .attr("stroke-linecap", "round");

  // Max vertical line (whisker cap)
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.max))
    .attr("x2", xScale(d.stats.max))
    .attr("y1", center - boxHeight / 3.5)
    .attr("y2", center + boxHeight / 3.5)
    .attr("stroke", "#495057")
    .attr("stroke-width", 2.5)
    .attr("stroke-linecap", "round");

  // Box shadow
  boxGroup.append("rect")
    .attr("x", xScale(d.stats.q1) + 3)
    .attr("y", y + 3)
    .attr("width", xScale(d.stats.q3) - xScale(d.stats.q1))
    .attr("height", boxHeight)
    .attr("fill", "#000")
    .attr("opacity", 0.15)
    .attr("rx", 4);

  // Box (IQR) with gradient
  const boxGradient = defs.append("linearGradient")
    .attr("id", `boxGradient${i}`)
    .attr("x1", "0%")
    .attr("y1", "0%")
    .attr("x2", "0%")
    .attr("y2", "100%");

  boxGradient.append("stop")
    .attr("offset", "0%")
    .attr("stop-color", d3.color(boxColor).brighter(0.3));

  boxGradient.append("stop")
    .attr("offset", "100%")
    .attr("stop-color", boxColor);

  boxGroup.append("rect")
    .attr("x", xScale(d.stats.q1))
    .attr("y", y)
    .attr("width", xScale(d.stats.q3) - xScale(d.stats.q1))
    .attr("height", boxHeight)
    .attr("fill", `url(#boxGradient${i})`)
    .attr("stroke", d3.color(boxColor).darker(0.5))
    .attr("stroke-width", 2)
    .attr("rx", 4);

  // Median line with glow effect
  boxGroup.append("line")
    .attr("x1", xScale(d.stats.median))
    .attr("x2", xScale(d.stats.median))
    .attr("y1", y)
    .attr("y2", y + boxHeight)
    .attr("stroke", "#fff")
    .attr("stroke-width", 4)
    .attr("opacity", 0.8);

  boxGroup.append("line")
    .attr("x1", xScale(d.stats.median))
    .attr("x2", xScale(d.stats.median))
    .attr("y1", y)
    .attr("y2", y + boxHeight)
    .attr("stroke", "#212529")
    .attr("stroke-width", 2.5);

  // Outliers with better styling
  d.stats.outliers.forEach(outlier => {
    boxGroup.append("circle")
      .attr("cx", xScale(outlier))
      .attr("cy", center)
      .attr("r", 4.5)
      .attr("fill", "#dc3545")
      .attr("stroke", "#fff")
      .attr("stroke-width", 1.5)
      .attr("opacity", 0.8);
  });
});

svg.node();
```

</details>

<br>

### Analysis & Insights 
<br>

### 1. Crime Category
<br>

#### Top 10 Incident by Median Duration
* ASSAULT (IN PROGRESS): SHOTS/LTD ACC HWY leads with the longest median duration (~6 hrs) whereas LARCENY (PAST): OTHER/LTD ACC HWY shows the widest distribution, with median around ~8.5 hrs but extending to ~10 hrs

<div class="insight"><strong>Insight 💡</strong><br> Highway-related incidents dominate the top positions, suggesting location complexity significantly impacts resolution time </div>

* Child abuse cases appear multiple times across different assault categories, showing varying durations (200-400 minutes)
* School-related incidents (burglary and larceny) show moderate durations with relatively tight distributions

<div class="insight"><strong>Insight 💡</strong><br> Child abuse cases need specialized personnel and may involve child protective services, extending duration</div>

* Several categories show extreme outliers extending to 1,400-1,600 minutes (23-27 hours)

<div class="insight"><strong>Insight 💡</strong><br> The presence of outliers indicates that standard protocols can be disrupted by case complexity, evidence gathering needs, or inter-agency coordination</div>

#### Bottom 10 Incident by Median Duration

* Transit-related incidents dominate the bottom 10, with most showing median durations under 50 minutes
* OTHER-CRIME INCIDENT: MARIJUANA/TRANSIT has the shortest median (~5-10 minutes)

<div class="insight"><strong>Insight 💡</strong><br> Transit incidents are handled with exceptional speed, likely due to Specialized transit police units, well-established protocols, need to minimize service disruptions, contained, accessible environments</div>

* Despite fast medians, many categories show extensive outliers (up to 1,500 minutes)
* DISORDERLY: PERSON/TRANSIT and DISORDERLY: GROUP/TRANSIT have numerous outliers clustered around 200-400 minutes
<div class="insight"><strong>Insight 💡</strong><br> The contrast between tight medians and distant outliers suggests most cases are routine, but complications can dramatically extend duration (10-20x normal duration)</div>


### 2. Potential Crime Category
<br>

### Analysis & Insights
<br>

#### Top 10 Incident by Median Duration

* SUSP LETTER: OUTSIDE leads with the longest median duration (4-5 hours)
* INVESTIGATE/POSSIBLE CRIME: MARIJUANA/SCHOOL shows similar high median duration (~200-250 minutes)
<div class="insight"><strong>Insight 💡</strong><br> Suspicious letter incidents require extensive protocols including hazmat assessment, evidence collection, area evacuation, and specialized unit deployment, significantly extending resolution time </div>


* Location matters: "INSIDE" incidents tend to have longer durations than highway-related ones
* School-related investigations show consistently high durations, suggesting thorough protocols for youth-involved cases
<div class="insight"><strong>Insight 💡</strong><br> Holding suspects requires booking procedures, questioning, warrant checks, legal consultations, and transport arrangements - creating standardized 3-4 hour processing windows </div>


* PO/SECURITY HOLDING SUSPECT: CIV CLOTHES/INSIDE and PO/SECURITY HOLDING SUSPECT: UNIFORM/INSIDE shows extreme outliers extending to 1,000 minutes (~17 hours)
<div class="insight"><strong>Insight 💡</strong><br> Outliers in holding suspect cases suggest complications like: identifying uncooperative suspects, waiting for specialized detectives, warrant confirmations, or inter-jurisdictional transfers </div>

#### Bottom 10 Incident by Median Duration


* All bottom 10 incidents show median durations under 20 minutes - remarkably fast
* Nine out of ten are transit-related, with medians clustered around 10-15 minutes

<div class="insight"><strong>Insight 💡</strong><br> Transit and highway disputes are resolved with exceptional speed, likely due to: dedicated rapid response units, need to minimize public disruption, standardized de-escalation protocols, and accessible locations for quick officer deployment </div>

* Weapons-related investigations (firearm, knife, shots fired) resolve faster than expected - all under 20 minutes median

<div class="insight"><strong>Insight 💡</strong><br> Potential crimes that don't materialize into actual incidents are quickly assessed and cleared, suggesting efficient triage protocols that distinguish genuine threats from false alarms </div>

* Despite ultra-fast medians, several categories show extreme outliers extending to 2,000-16,000 minutes (33-267 hours!)
* INVESTIGATE/POSSIBLE CRIME Category has a dramatic outlier reaching at ~16,000 minutes (~11 days)

<div class="insight"><strong>Insight 💡</strong><br> The extraordinary contrast between 10-minute medians and multi-day outliers indicates that while most potential crimes are quickly determined to be non-threats, rare cases evolve into extended investigations, evidence recovery operations, or ongoing surveillance activities </div>

<br>


<!-- Question 2 -->
## How have 911 call volumes and incident types changed over the year? 
<br>

  ```js 
  const callCategory2 = view(Inputs.radio(focusOptions, {
    label: "Focus on call set",
    value: "Crime"
  }));
  ```

<br>

<details>
  <summary> Data Loading </summary>

  Loading the count of incident date grouped by incident date (data processed using <a href="https://github.com/LaxmanSRawat/NovaSight/blob/main/temporal_analysis_count_incident_agg_by_time.ipynb" rel="external">python jupyter notebook) </a>

  <br>

  ```js echo
  // Load the count of type description grouped by indcident date 
  const crime_data = FileAttachment("nyc_count_crime_type_by_date.csv").csv({typed: true})
  
  const potential_crime_data = FileAttachment("nyc_count_potential_crime_type_by_date.csv").csv({typed: true})
  
  const data2 = callCategory2 == "Crime"
  ? crime_data
  : potential_crime_data;

  ```

  ```js
  data2
  ```

  <br>

  ```js echo
  const crime_data_by_date_categorised = data2.map(d=> {
                        return {typ_desc: d.typ_desc.split('(')[0].split(':')[0].trim(), incident_date: d.incident_date, count: d.count}
})
  ```

  ```js echo
      // Aggregate counts by typ_desc and date
    const aggregatedData = d3.rollup(
      crime_data_by_date_categorised,
      v => d3.sum(v, d => d.count),  
      d => d.typ_desc,                
      d => d.incident_date
    );

    // Convert to array 
    const crime_data_by_date_categorised_arr = [];
    aggregatedData.forEach((dates, typ_desc) => {
      dates.forEach((count, incident_date) => {
        crime_data_by_date_categorised_arr.push({ typ_desc, incident_date, count });
      });
    });

    const crime_data_by_date_categorised_arr_sorted = crime_data_by_date_categorised_arr.map(d => ({
    ...d,
    incident_date: new Date(d.incident_date)
  })).sort((a, b) => a.incident_date - b.incident_date);

  ```

  ```js echo
  crime_data_by_date_categorised_arr
  ```
</details>

```js 
Swatches(d3.scaleOrdinal()
  .domain(new Set(crime_data_by_date_categorised_arr_sorted.map(d => d.typ_desc)))
  .range(d3.schemeCategory10))
```


<!-- Plot 2 -->
```js
// Set Dimensions and Margins
const width = 928;
const height = 500;
const marginTop = 60;
const marginRight = 60;
const marginBottom = 60;
const marginLeft = 60;

// X-Axis 
const x = d3.scaleTime()
  .domain(d3.extent(crime_data_by_date_categorised_arr_sorted, d => d.incident_date))
  .range([marginLeft, width - marginRight]);

// Y-Axis
const y = d3.scaleLinear()
  .domain([0, d3.max(crime_data_by_date_categorised_arr_sorted, d => d.count)]).nice()
  .range([height - marginBottom, marginTop]);

const color = d3.scaleOrdinal()
  .domain(new Set(crime_data_by_date_categorised_arr_sorted.map(d => d.typ_desc)))
  .range(d3.schemeCategory10);

// Create the SVG Container
const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", [0, 0, width, height])
    .attr("style", "max-width: 100%; height: auto;");

// Background
const defs = svg.append("defs");
const bgGradient = defs.append("linearGradient")
  .attr("id", "bgGradient")
  .attr("x1", "0%")
  .attr("y1", "0%")
  .attr("x2", "0%")
  .attr("y2", "100%");

bgGradient.append("stop").attr("offset", "0%").attr("stop-color", "#f8fafc");
bgGradient.append("stop").attr("offset", "100%").attr("stop-color", "#e2e8f0");

svg.append("rect")
  .attr("width", width)
  .attr("height", height)
  .attr("fill", "url(#bgGradient)");

svg.append("g")
    .attr("class", "grid")
    .attr("stroke", "#4d5968ff")
    .attr("stroke-opacity", 0.8)
    .attr("stroke-dasharray", "3,3")
    .call(g => g.append("g")
      .selectAll("line")
      .data(x.ticks())
      .join("line")
        .attr("x1", d => 0.5 + x(d)).attr("x2", d => 0.5 + x(d))
        .attr("y1", marginTop).attr("y2", height - marginBottom))
    .call(g => g.append("g")
      .selectAll("line")
      .data(y.ticks())
      .join("line")
        .attr("y1", d => 0.5 + y(d)).attr("y2", d => 0.5 + y(d))
        .attr("x1", marginLeft).attr("x2", width - marginRight));

// Axes
const xAxis = svg.append("g")
    .attr("transform", `translate(0,${height - marginBottom})`)
    .call(d3.axisBottom(x).ticks(width / 80).tickSizeOuter(0));

xAxis.selectAll("text").style("fill", "#475569").style("font-weight", "500");
xAxis.select(".domain").style("stroke", "#94a3b8");
xAxis.selectAll(".tick line").style("stroke", "#94a3b8");

const yAxis = svg.append("g")
    .attr("transform", `translate(${marginLeft},0)`)
    .call(d3.axisLeft(y).ticks(8));

yAxis.selectAll("text").style("fill", "#475569").style("font-weight", "500");
yAxis.select(".domain").style("stroke", "#94a3b8");
yAxis.selectAll(".tick line").style("stroke", "#94a3b8");

yAxis.append("text")
    .attr("transform", "rotate(-90)")
    .attr("x", -(height - marginBottom - marginTop) / 2 - marginTop)
    .attr("y", -45)
    .attr("fill", "#1e293b")
    .attr("text-anchor", "middle")
    .attr("font-size", "14px")
    .attr("font-weight", "600")
    .text("Count of Incidents");

svg.append("text")
    .attr("x", width / 2)
    .attr("y", 30)
    .attr("text-anchor", "middle")
    .attr("fill", "#1e293b")
    .attr("font-size", "18px")
    .attr("font-weight", "700")
    .text("Incident Trends Over Time by Date of Year");

// --- 3. Lines ---
const line = d3.line()
  .x(d => x(d.incident_date))
  .y(d => y(d.count))
  .curve(d3.curveMonotoneX);

const groupedData = d3.group(crime_data_by_date_categorised_arr_sorted, d => d.typ_desc);

const linesGroup = svg.append("g")
    .attr("fill", "none")
    .attr("stroke-width", 2.5)
    .attr("stroke-linejoin", "round")
    .attr("stroke-linecap", "round");

linesGroup.selectAll("path")
  .data(groupedData)
  .join("path")
    .attr("stroke", ([group, _]) => color(group))
    .attr("d", ([_, values]) => line(values))
    .attr("opacity", 0.85);


// --- 5. INTERACTION LOGIC ---

// A. Create the interaction container elements
const interactionGroup = svg.append("g")
  .style("pointer-events", "none")
  .attr("display", "none");

// The vertical line
const verticalLine = interactionGroup.append("line")
  .attr("stroke", "#475569")
  .attr("stroke-width", 1)
  .attr("stroke-dasharray", "4,4")
  .attr("y1", marginTop)
  .attr("y2", height - marginBottom);

// The date text at the top
const dateLabel = interactionGroup.append("text")
  .attr("y", marginTop - 10)
  .attr("text-anchor", "middle")
  .attr("font-size", "12px")
  .attr("font-weight", "bold")
  .attr("fill", "#1e293b");

// A group to hold the dynamic dots and count labels
const tooltipGroup = interactionGroup.append("g");

// B. Define the bisector
const bisect = d3.bisector(d => d.incident_date).center;

// C. Mouse Event Handlers with Bounds Check
svg.on("pointerenter", () => {
  // We can leave this empty or perform setup, 
  // but visibility is now controlled in pointermove
})
.on("pointerleave", () => {
  interactionGroup.attr("display", "none");
  linesGroup.selectAll("path").attr("opacity", 0.85).attr("stroke-width", 2.5);
})
.on("pointermove", (event) => {
  const [xPos, yPos] = d3.pointer(event);

  // --- BOUNDS CHECK START ---
  // If the mouse is outside the chart area (left, right, top, or bottom margins)
  if (xPos < marginLeft || xPos > width - marginRight || 
      yPos < marginTop || yPos > height - marginBottom) {
      
      interactionGroup.attr("display", "none");
      return; // Stop processing
  }
  // --- BOUNDS CHECK END ---

  // If we are inside, show the group
  interactionGroup.attr("display", null);
  
  // 1. Invert x-position to get the corresponding date
  const date = x.invert(xPos);

  // 2. Update vertical line position
  verticalLine.attr("x1", xPos).attr("x2", xPos);
  
  // 3. Update top date label
  dateLabel
    .attr("x", xPos)
    .text(d3.timeFormat("%b %d")(date));

  // 4. Update intersection dots and labels
  tooltipGroup.selectAll("*").remove();

  groupedData.forEach((values, key) => {
    const index = bisect(values, date);
    
    if (index < values.length && index >= 0) {
      const d = values[index];
      const px = x(d.incident_date);
      const py = y(d.count);

      // Only show dots if the data point is close to the mouse cursor X-wise
      if (Math.abs(px - xPos) < 20) {
        
        // Add dot
        tooltipGroup.append("circle")
          .attr("cx", px)
          .attr("cy", py)
          .attr("r", 4)
          .attr("fill", "white")
          .attr("stroke", color(key))
          .attr("stroke-width", 2);

        // Add count text
        const label = tooltipGroup.append("text")
          .attr("x", px + 8) 
          .attr("y", py + 4)
          .attr("font-size", "11px")
          .attr("font-weight", "bold")
          .attr("fill", color(key))
          .text(d.count);
          
        // Add white halo
        label.attr("stroke", "white")
             .attr("stroke-width", 3)
             .attr("stroke-linejoin", "round")
             .attr("paint-order", "stroke")
             .clone(true)
             .attr("fill", color(key))
             .attr("stroke", "none");
      }
    }
  });
});

display(svg.node());
```

<details>
<summary> Code </summary>

```js echo

// Set Dimensions and Margins
const width = 928;
const height = 500;
const marginTop = 60;
const marginRight = 180;
const marginBottom = 60;
const marginLeft = 60;

// X-Axis 
const x = d3.scaleTime()
  .domain(d3.extent(crime_data_by_date_categorised_arr_sorted, d => d.incident_date))
  .range([marginLeft, width - marginRight]);

// Y-Axis
const y = d3.scaleLinear()
  .domain([0, d3.max(crime_data_by_date_categorised_arr_sorted, d => d.count)]).nice()
  .range([height - marginBottom, marginTop]);

const color = d3.scaleOrdinal()
  .domain(new Set(crime_data_by_date_categorised_arr_sorted.map(d => d.typ_desc)))
  .range(d3.schemeCategory10);

// Create the SVG Container
const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", [0, 0, width, height])
    .attr("style", "max-width: 100%; height: auto;");

// Background
const defs = svg.append("defs");
const bgGradient = defs.append("linearGradient")
  .attr("id", "bgGradient")
  .attr("x1", "0%")
  .attr("y1", "0%")
  .attr("x2", "0%")
  .attr("y2", "100%");

bgGradient.append("stop").attr("offset", "0%").attr("stop-color", "#f8fafc");
bgGradient.append("stop").attr("offset", "100%").attr("stop-color", "#e2e8f0");

svg.append("rect")
  .attr("width", width)
  .attr("height", height)
  .attr("fill", "url(#bgGradient)");

svg.append("g")
    .attr("class", "grid")
    .attr("stroke", "#4d5968ff")
    .attr("stroke-opacity", 0.8)
    .attr("stroke-dasharray", "3,3")
    .call(g => g.append("g")
      .selectAll("line")
      .data(x.ticks())
      .join("line")
        .attr("x1", d => 0.5 + x(d)).attr("x2", d => 0.5 + x(d))
        .attr("y1", marginTop).attr("y2", height - marginBottom))
    .call(g => g.append("g")
      .selectAll("line")
      .data(y.ticks())
      .join("line")
        .attr("y1", d => 0.5 + y(d)).attr("y2", d => 0.5 + y(d))
        .attr("x1", marginLeft).attr("x2", width - marginRight));

// Axes
const xAxis = svg.append("g")
    .attr("transform", `translate(0,${height - marginBottom})`)
    .call(d3.axisBottom(x).ticks(width / 80).tickSizeOuter(0));

xAxis.selectAll("text").style("fill", "#475569").style("font-weight", "500");
xAxis.select(".domain").style("stroke", "#94a3b8");
xAxis.selectAll(".tick line").style("stroke", "#94a3b8");

const yAxis = svg.append("g")
    .attr("transform", `translate(${marginLeft},0)`)
    .call(d3.axisLeft(y).ticks(8));

yAxis.selectAll("text").style("fill", "#475569").style("font-weight", "500");
yAxis.select(".domain").style("stroke", "#94a3b8");
yAxis.selectAll(".tick line").style("stroke", "#94a3b8");

yAxis.append("text")
    .attr("transform", "rotate(-90)")
    .attr("x", -(height - marginBottom - marginTop) / 2 - marginTop)
    .attr("y", -45)
    .attr("fill", "#1e293b")
    .attr("text-anchor", "middle")
    .attr("font-size", "14px")
    .attr("font-weight", "600")
    .text("Count of Incidents");

svg.append("text")
    .attr("x", width / 2)
    .attr("y", 30)
    .attr("text-anchor", "middle")
    .attr("fill", "#1e293b")
    .attr("font-size", "18px")
    .attr("font-weight", "700")
    .text("Incident Trends Over Time by Type");

// --- 3. Lines ---
const line = d3.line()
  .x(d => x(d.incident_date))
  .y(d => y(d.count))
  .curve(d3.curveMonotoneX);

const groupedData = d3.group(crime_data_by_date_categorised_arr_sorted, d => d.typ_desc);

const linesGroup = svg.append("g")
    .attr("fill", "none")
    .attr("stroke-width", 2.5)
    .attr("stroke-linejoin", "round")
    .attr("stroke-linecap", "round");

linesGroup.selectAll("path")
  .data(groupedData)
  .join("path")
    .attr("stroke", ([group, _]) => color(group))
    .attr("d", ([_, values]) => line(values))
    .attr("opacity", 0.85);

// --- 4. Legend ---
const legend = svg.append("g")
    .attr("transform", `translate(${width - marginRight + 20}, ${marginTop})`);

const legendItems = Array.from(groupedData.keys());
legendItems.forEach((item, i) => {
  const legendRow = legend.append("g").attr("transform", `translate(0, ${i * 28})`);
  legendRow.append("line")
      .attr("x1", 0).attr("x2", 20).attr("y1", 0).attr("y2", 0)
      .attr("stroke", color(item)).attr("stroke-width", 3);
  legendRow.append("text")
      .attr("x", 28).attr("y", 4)
      .attr("fill", "#334155")
      .attr("font-size", "11px")
      .text(item.length > 20 ? item.substring(0, 18) + '...' : item);
});

// --- 5. INTERACTION LOGIC ---

// A. Create the interaction container elements
const interactionGroup = svg.append("g")
  .style("pointer-events", "none")
  .attr("display", "none");

// The vertical line
const verticalLine = interactionGroup.append("line")
  .attr("stroke", "#475569")
  .attr("stroke-width", 1)
  .attr("stroke-dasharray", "4,4")
  .attr("y1", marginTop)
  .attr("y2", height - marginBottom);

// The date text at the top
const dateLabel = interactionGroup.append("text")
  .attr("y", marginTop - 10)
  .attr("text-anchor", "middle")
  .attr("font-size", "12px")
  .attr("font-weight", "bold")
  .attr("fill", "#1e293b");

// A group to hold the dynamic dots and count labels
const tooltipGroup = interactionGroup.append("g");

// B. Define the bisector
const bisect = d3.bisector(d => d.incident_date).center;

// C. Mouse Event Handlers with Bounds Check
svg.on("pointerenter", () => {
  // We can leave this empty or perform setup, 
  // but visibility is now controlled in pointermove
})
.on("pointerleave", () => {
  interactionGroup.attr("display", "none");
  linesGroup.selectAll("path").attr("opacity", 0.85).attr("stroke-width", 2.5);
})
.on("pointermove", (event) => {
  const [xPos, yPos] = d3.pointer(event);

  // --- BOUNDS CHECK START ---
  // If the mouse is outside the chart area (left, right, top, or bottom margins)
  if (xPos < marginLeft || xPos > width - marginRight || 
      yPos < marginTop || yPos > height - marginBottom) {
      
      interactionGroup.attr("display", "none");
      return; // Stop processing
  }
  // --- BOUNDS CHECK END ---

  // If we are inside, show the group
  interactionGroup.attr("display", null);
  
  // 1. Invert x-position to get the corresponding date
  const date = x.invert(xPos);

  // 2. Update vertical line position
  verticalLine.attr("x1", xPos).attr("x2", xPos);
  
  // 3. Update top date label
  dateLabel
    .attr("x", xPos)
    .text(d3.timeFormat("%b %d")(date));

  // 4. Update intersection dots and labels
  tooltipGroup.selectAll("*").remove();

  groupedData.forEach((values, key) => {
    const index = bisect(values, date);
    
    if (index < values.length && index >= 0) {
      const d = values[index];
      const px = x(d.incident_date);
      const py = y(d.count);

      // Only show dots if the data point is close to the mouse cursor X-wise
      if (Math.abs(px - xPos) < 20) {
        
        // Add dot
        tooltipGroup.append("circle")
          .attr("cx", px)
          .attr("cy", py)
          .attr("r", 4)
          .attr("fill", "white")
          .attr("stroke", color(key))
          .attr("stroke-width", 2);

        // Add count text
        const label = tooltipGroup.append("text")
          .attr("x", px + 8) 
          .attr("y", py + 4)
          .attr("font-size", "11px")
          .attr("font-weight", "bold")
          .attr("fill", color(key))
          .text(d.count);
          
        // Add white halo
        label.attr("stroke", "white")
             .attr("stroke-width", 3)
             .attr("stroke-linejoin", "round")
             .attr("paint-order", "stroke")
             .clone(true)
             .attr("fill", color(key))
             .attr("stroke", "none");
      }
    }
  });
});

svg.node();

```
</details>

<br>

### Analysis and Insights
<br>

#### 1. Crime Category

* **OTHER CRIMES** consistently dominates with the highest incident count, ranging between 700-950 incidents daily throughout the year followed by **LARCENY** maintains the second position with 450-700 incidents daily. 
* **ASSAULT** and **DISORDERLY** occupy the middle tier, both fluctuating between 250-450 incidents daily
* **BURGLARY** and **ROBBERY** remain at the bottom with 50-150 incidents daily
* **OTHER-CRIME INCIDENT** shows minimal activity, barely visible at the bottom of the chart

<div class="insight"><strong>Insight 💡</strong><br> The consistent layering suggests stable crime category proportions, OTHER CRIMES represents ~35-40% of all incidents, while property crimes (larceny + burglary) combined account for ~30-35% of daily incidents </div>


* Most crime categories show a synchronized rhythm of increasing during the summer and autumn and then valleying during the winters and spring.
* Most categories show elevated activity in late June/July and October, with OTHER CRIMES peaking near 950-1,000 incidents
* All categories trend downward from late November through December, with notable drops on holidays (28th November and 24th December).


<div class="insight"><strong>Insight 💡</strong><br> The winter drop may correlate with: less people staying outdoors, school being out of session (reducing certain incident types), vacation periods affecting reporting patterns, or holiday weekends. The late June/July and October spike could relate to return to school/work routines and seasonal factors </div>

* OTHER CRIMES and ASSAULT show the most dramatic day-to-day fluctuations (swings of 100-200 incidents) while BURGLARY and ROBBERY show remarkably stable, flat trends with minimal variation

<div class="insight"><strong>Insight 💡</strong><br> Property crimes (burglary, robbery) show consistent patterns suggesting professional/opportunistic crimes occur at steady rates, while person-based crimes (assault, disorderly conduct) are more sensitive to external factors like weather, events, and day-of-week effects </div>

* All categories exhibit regular wave patterns suggesting strong day-of-week effects with OTHER CRIMES and LARCENY shows the most pronounced weekly cycling (for example, look at October month's pattern)

<div class="insight"><strong>Insight 💡</strong><br> Regular weekly patterns indicate that certain crime types surge on specific days (likely weekends for disorderly conduct and assaults in entertainment districts, weekdays for larceny in commercial areas) </div>

#### 2. Potential Crime Category

* **INVESTIGATE/POSSIBLE CRIME** (red) overwhelmingly dominates with 1,400-2,200 incidents daily, representing approximately 60-70% of all potential crime incidents, followed by **DISPUTE** (orange) maintains steady second position at 600-900 incidents daily.
* **ALARMS** (blue) occupies third tier with 200-600 incidents daily, showing significant variability
* **All other categories** (PANIC ALARM, SUSP PACKAGE, SUSP LETTER, SUSP SUBSTANCE, etc.) remain compressed at the bottom with minimal activity (<100 incidents daily each)

<div class="insight"><strong>Insight 💡</strong><br> The dramatic dominance of INVESTIGATE/POSSIBLE CRIME suggests most potential incidents require initial investigation but may not develop into confirmed crimes, this represents the "triage" layer of police response </div>

* All categories show synchronized severe decline in the final two weeks of December, with INVESTIGATE/POSSIBLE CRIME dropping from ~1,800 to ~1,000 incidents
* There is a notable spike around July 4th period, followed by increased fluctuations

<div class="insight"><strong>Insight 💡</strong><br> The December collapse likely reflects: reduced civilian activity during holidays, fewer business operations, school closures, and potentially reduced reporting. The July spike may correlate with Independence Day celebrations generating more suspicious activity reports and fireworks-related calls </div>

* Similar to crime category, pronounced 7-day oscillation pattern visible across all categories, particularly dramatic in INVESTIGATE/POSSIBLE CRIME
* Exception to these are specialized categories (SUSP LETTER, EXPLOSIVE DEVICE, PANIC ALARM) remain remarkably flat with almost no visible variation

<div class="insight"><strong>Insight 💡</strong><br> The extreme sawtooth pattern in INVESTIGATE/POSSIBLE CRIME strongly suggests weekend vs. weekday effects, likely massive drops on weekends (fewer businesses open, less foot traffic) with sharp Monday increases as commercial activity resumes </div>

<br>

<!-- Question 3 -->
## How does the distribution of call types vary throughout the day? 

<br>

  ```js 
  const callCategory3 = view(Inputs.radio(focusOptions, {
    label: "Focus on call set",
    value: "Crime"
  }));
  ```

  <br>

<details>
  <summary> Data Loading </summary>

  Loading the count of type description grouped by incident time (data processed using <a href="https://github.com/LaxmanSRawat/NovaSight/blob/main/temporal_analysis_count_incident_agg_by_time.ipynb" rel="external">python jupyter notebook) </a>

  <br>

  ```js echo
  // Load the count of type description grouped by indcident time 
  const crime_data_by_time = FileAttachment("nyc_count_crime_type_by_time.csv").csv({typed: true})
  const potential_crime_data_by_time = FileAttachment("nyc_count_potential_crime_type_by_time.csv").csv({typed: true})
  
  const data3 = callCategory3 === "Crime" ? crime_data_by_time:potential_crime_data_by_time;

  ```

  <br>

  ```js echo
  const crime_data_by_time_categorised = data3.map(d=> {
                        return {typ_desc: d.typ_desc.split('(')[0].split(':')[0].trim(), incident_time: d.incident_time, count: d.count}
})

  // Process data: extract hour and aggregate by typ_desc and hour
  const processedData = d3.rollup(
    crime_data_by_time_categorised,
    v => d3.sum(v, d => d.count),
    d => d.typ_desc,
    d => {
      const hour = d.incident_time.split(':')[0];
      return parseInt(hour);
    }
  );

  // Convert to array format for heatmap
  const heatmapData = [];
  processedData.forEach((hours, typ_desc) => {
    hours.forEach((count, hour) => {
      heatmapData.push({ typ_desc, hour, count });
    });
  });

  // Get unique categories and hours
  const categories = Array.from(new Set(heatmapData.map(d => d.typ_desc))).sort();
  const hours = d3.range(0, 24);

  ```
</details>

```js 
Legend(d3.scaleSequential([0, d3.max(heatmapData.map(d => d.count))], d3.interpolateInferno),{
  title: "Count",
  width: 320
})
```

<!-- Plot 3 -->

```js
// Dimensions
const margin = { top: 80, right: 150, bottom: 80, left: 240 };
const cellWidth = 32;
const cellHeight = 32;
const width = cellWidth * hours.length + margin.left + margin.right;
const height = cellHeight * categories.length + margin.top + margin.bottom;

// Create SVG
const svg = d3.create("svg")
  .attr("width", width)
  .attr("height", height)
  .attr("viewBox", [0, 0, width, height])
  .attr("style", "max-width: 100%; height: auto; overflow: visible;");

// Add gradient background
const defs = svg.append("defs");
const bgGradient = defs.append("linearGradient")
  .attr("id", "heatmapBgGradient")
  .attr("x1", "0%")
  .attr("y1", "0%")
  .attr("x2", "0%")
  .attr("y2", "100%");

bgGradient.append("stop")
  .attr("offset", "0%")
  .attr("stop-color", "#f1f5f9");

bgGradient.append("stop")
  .attr("offset", "100%")
  .attr("stop-color", "#e2e8f0");

svg.append("rect")
  .attr("width", width)
  .attr("height", height)
  .attr("fill", "url(#heatmapBgGradient)");

const g = svg.append("g")
  .attr("transform", `translate(${margin.left},${margin.top})`);

// Color scale
const colorScale = d3.scaleSequential(d3.interpolateInferno)
  .domain([0, d3.max(heatmapData, d => d.count)]);

// X scale (hours)
const xScale = d3.scaleBand()
  .domain(hours)
  .range([0, cellWidth * hours.length])
  .padding(0.08);

// Y scale (typ_desc)
const yScale = d3.scaleBand()
  .domain(categories)
  .range([0, cellHeight * categories.length])
  .padding(0.08);

// Create heatmap cells 
const cells = g.selectAll("rect")
  .data(heatmapData)
  .join("rect")
  .attr("x", d => xScale(d.hour))
  .attr("y", d => yScale(d.typ_desc))
  .attr("width", xScale.bandwidth())
  .attr("height", yScale.bandwidth())
  .attr("fill", d => colorScale(d.count))
  .attr("rx", 4)
  .attr("ry", 4)
  .attr("stroke", "#fff")
  .attr("stroke-width", 1.5)
  .attr("filter", "url(#cell-shadow)")
  .style("cursor", "pointer")
  .style("transition", "all 0.2s")
  .on("mouseover", function(event, d) {
    d3.select(this)
      .attr("stroke", "#1e293b")
      .attr("stroke-width", 3)
    
    // Show tooltip
    tooltip.style("display", null)
      .attr("transform", `translate(${xScale(d.hour) + xScale.bandwidth() / 2},${yScale(d.typ_desc) - 10})`);
    
    tooltip.select(".tooltip-count")
      .text(`Count: ${d.count}`);
    
    tooltip.select(".tooltip-hour")
      .text(`Hour: ${d.hour}:00`);
  })
  .on("mouseout", function() {
    d3.select(this)
      .attr("stroke", "#fff")
      .attr("stroke-width", 1.5)
      .attr("transform", "scale(1)");
    
    tooltip.style("display", "none");
  });

// Create tooltip
const tooltip = g.append("g")
  .attr("class", "tooltip")
  .style("display", "none")
  .style("pointer-events", "none");

tooltip.append("rect")
  .attr("width", 100)
  .attr("height", 45)
  .attr("x", -50)
  .attr("y", -50)
  .attr("fill", "white")
  .attr("stroke", "#cbd5e1")
  .attr("stroke-width", 2)
  .attr("rx", 6)
  .attr("opacity", 0.95)
  .attr("filter", "url(#cell-shadow)");

tooltip.append("text")
  .attr("class", "tooltip-count")
  .attr("x", 0)
  .attr("y", -32)
  .attr("text-anchor", "middle")
  .attr("font-size", "12px")
  .attr("font-weight", "600")
  .attr("fill", "#1e293b")
  .attr("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

tooltip.append("text")
  .attr("class", "tooltip-hour")
  .attr("x", 0)
  .attr("y", -18)
  .attr("text-anchor", "middle")
  .attr("font-size", "11px")
  .attr("fill", "#475569")
  .attr("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

// X axis
const xAxis = g.append("g")
  .attr("transform", `translate(0,${cellHeight * categories.length})`)
  .call(d3.axisBottom(xScale).tickFormat(d => `${d}:00`));

xAxis.selectAll("text")
  .attr("transform", "rotate(-45)")
  .style("text-anchor", "end")
  .style("fill", "#475569")
  .style("font-size", "11px")
  .style("font-weight", "500")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

xAxis.select(".domain")
  .style("stroke", "#94a3b8")
  .style("stroke-width", "2px");

xAxis.selectAll(".tick line")
  .style("stroke", "#94a3b8");

// X-axis label
g.append("text")
  .attr("x", (cellWidth * hours.length) / 2)
  .attr("y", cellHeight * categories.length + 65)
  .attr("text-anchor", "middle")
  .style("font-size", "14px")
  .style("font-weight", "600")
  .style("fill", "#1e293b")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif")
  .text("Hour of Day");

// Y axis
const yAxis = g.append("g")
  .call(d3.axisLeft(yScale));

yAxis.selectAll("text")
  .style("fill", "#475569")
  .style("font-size", "11px")
  .style("font-weight", "500")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif");

yAxis.select(".domain")
  .style("stroke", "#94a3b8")
  .style("stroke-width", "2px");

yAxis.selectAll(".tick line")
  .style("stroke", "#94a3b8");

// Y-axis label
g.append("text")
  .attr("transform", "rotate(-90)")
  .attr("x", -(cellHeight * categories.length) / 2)
  .attr("y", -200)
  .attr("text-anchor", "middle")
  .style("font-size", "14px")
  .style("font-weight", "600")
  .style("fill", "#1e293b")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif")
  .text("Incident Type");

// Title
svg.append("text")
  .attr("x", width / 2)
  .attr("y", 35)
  .attr("text-anchor", "middle")
  .style("font-size", "20px")
  .style("font-weight", "700")
  .style("fill", "#1e293b")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif")
  .text("Incident Patterns by Hour of Day");

// Add subtitle
svg.append("text")
  .attr("x", width / 2)
  .attr("y", 55)
  .attr("text-anchor", "middle")
  .style("font-size", "13px")
  .style("font-weight", "400")
  .style("fill", "#64748b")
  .style("font-family", "'Segoe UI', Tahoma, Geneva, Verdana, sans-serif")
  .text("Heatmap showing incident frequency by type and time");

// Create color legend
const legendWidth = 20;
const legendHeight = cellHeight * categories.length;
const legendX = cellWidth * hours.length + 30;
const legendY = 0;

display(svg.node());

```

<details>
<summary> Code </summary>

```js echo
{
  // Dimensions
  const margin = { top: 50, right: 100, bottom: 50, left: 150 };
  const cellWidth = 30;
  const cellHeight = 30;
  const width = cellWidth * hours.length + margin.left + margin.right;
  const height = cellHeight * categories.length + margin.top + margin.bottom;

  // Create SVG
  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", [0, 0, width, height])
    .attr("style", "max-width: 100%; height: auto; overflow: visible; font: 10px arial;")
    .style("background","#dfdfd6");

  const g = svg.append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);

  // Color scale
  const colorScale = d3.scaleSequential(d3.interpolateInferno)
    .domain([0, d3.max(heatmapData, d => d.count)]);

  // X scale (hours)
  const xScale = d3.scaleBand()
    .domain(hours)
    .range([0, cellWidth * hours.length])
    .padding(0.05);

  // Y scale (typ_desc)
  const yScale = d3.scaleBand()
    .domain(categories)
    .range([0, cellHeight * categories.length])
    .padding(0.05);

  // Create heatmap cells
  g.selectAll("rect")
    .data(heatmapData)
    .join("rect")
    .attr("x", d => xScale(d.hour))
    .attr("y", d => yScale(d.typ_desc))
    .attr("width", xScale.bandwidth())
    .attr("height", yScale.bandwidth())
    .attr("fill", d => colorScale(d.count))
    .attr("rx", 2) // Rounded corners
    .attr("ry", 2)
    // .attr("stroke", "black")
    .attr("stroke-width", 0.5)
    .append("title")
    .text(d => `${d.typ_desc}\nHour: ${d.hour}:00\nCount: ${d.count}`);

  // X axis
  g.append("g")
    .attr("transform", `translate(0,${cellHeight * categories.length})`)
    .call(d3.axisBottom(xScale).tickFormat(d => `${d}:00`))
    .selectAll("text")
    .attr("transform", "rotate(-45)")
    .style("text-anchor", "end")
    .style("color","#000")


  // Y axis
  g.append("g")
    .call(d3.axisLeft(yScale))
    .style("color","#000")

  // Title
  svg.append("text")
    .attr("x", width / 2)
    .attr("y", margin.top / 2)
    .attr("text-anchor", "middle")
    .style("font-size", "16px")
    .style("font-weight", "bold")
    .text("Crime Type by Hour of Day");

  svg.node();
}
```

</details>


### Analysis and Insights
<br>

#### 1. Crime Category

* **OTHER CRIMES**: Most dramatic hourly variation - shows distinct hot zone from 14:00-20:00 (bright yellow/orange), then sharp decline after 21:00. **LARCENY** shows similar variation with evening elevation but less dramatic than Other Crimes.
* **ASSAULT** and **DISORDERLY** show relatively consistent purple throughout day with moderate elevation during evening hours (18:00-23:00)
* **BURGLARY** and **ROBBERY** show uniform dark coloring across all hours - minimal hourly variation

<div class="insight"><strong>Insight 💡</strong><br> Property crimes split into two patterns: LARCENY (opportunistic theft) spikes when crowds and commerce peak, while BURGLARY remains constant suggesting it occurs regardless of time (targeting unoccupied properties, following opportunity rather than clock) </div>

* All crime categories show maximum activity during late afternoon/evening (15:00-20:00) 5-hour window, with colors shifting to dark to brighter.
* OTHER CRIMES shows the most intense concentration during peak hours (15:00-19:00), with the brightest yellow coloring indicating 10,000-15,000+ incidents
* All categories show significantly reduced activity (dark purple/black) during early morning hours (00:00-06:00) Which gradually increase in activity as the city awakens, with noticeable transitions from dark to lighter purple (06:00-12:00)

<div class="insight"><strong>Insight 💡</strong><br> The 15:00-20:00 peak aligns with: evening commute periods, school dismissal times, retail peak hours, transition from work to leisure activities, and maximum street population density - creating optimal conditions for various crime types </div>

#### 2. Potential Crime Category 

* **INVESTIGATE/POSSIBLE CRIME**: Shows extreme hourly variation - brightest yellow during 12:00-18:00 (peak of 40,000-50,000 incidents), then dramatic decline to near-zero overnight
* **DISPUTE** shows relatively consistent purple throughout day with moderate elevation during afternoon/evening (14:00-20:00)
* **ALARMS** stays remarkably uniform purple coloring across most hours, minimal variation except slight elevation during the start of business hours
* **EXPLOSIVE DEVICE OR THREAT**, **SHOT SPOTTER**, and **SUSP PACKAGE** remains notably flat dark pattern with slight purple tint during business hours
* **SEX OFFENDER HA ADDRESS VERIFY** Shows unique isolated at 02:00, 14:00, and 20:00 - suggesting scheduled verification processes.

<div class="insight"><strong>Insight 💡</strong><br> The dramatic business-hour concentration suggests potential crime reports are heavily driven by civilian observation and reporting behavior - when people are active, alert, and in public/commercial spaces, they report more suspicious activity </div>

<br>

<details> 
<summary> Appendix </summary>



```js echo

// Helper Function for IQR
function calculateBoxPlotStats(values) {
            const sorted = values.sort(d3.ascending);
            const q1 = d3.quantile(sorted, 0.25);
            const median = d3.quantile(sorted, 0.5);
            const q3 = d3.quantile(sorted, 0.75);
            const iqr = q3 - q1;
            const min = d3.max([d3.min(sorted), q1 - 1.5 * iqr]);
            const max = d3.min([d3.max(sorted), q3 + 1.5 * iqr]);
            
            const outliers = sorted.filter(v => v < q1 - 1.5 * iqr || v > q3 + 1.5 * iqr);
            
            return { q1, median, q3, min, max, outliers };
        }
  
// Copyright 2021, Observable Inc.
// Released under the ISC license.
// https://observablehq.com/@d3/color-legend

function Legend(color, {
  title,
  tickSize = 6,
  width = 320, 
  height = 44 + tickSize,
  marginTop = 18,
  marginRight = 0,
  marginBottom = 16 + tickSize,
  marginLeft = 0,
  ticks = width / 64,
  tickFormat,
  tickValues
} = {}) {

  function ramp(color, n = 256) {
    const canvas = document.createElement("canvas");
    canvas.width = n;
    canvas.height = 1;
    const context = canvas.getContext("2d");
    for (let i = 0; i < n; ++i) {
      context.fillStyle = color(i / (n - 1));
      context.fillRect(i, 0, 1, 1);
    }
    return canvas;
  }

  const svg = d3.create("svg")
      .attr("width", width)
      .attr("height", height)
      .attr("viewBox", [0, 0, width, height])
      .style("overflow", "visible")
      .style("display", "block")
      

  let tickAdjust = g => g.selectAll(".tick line").attr("y1", marginTop + marginBottom - height);
  let x;

  // Continuous
  if (color.interpolate) {
    const n = Math.min(color.domain().length, color.range().length);

    x = color.copy().rangeRound(d3.quantize(d3.interpolate(marginLeft, width - marginRight), n));

    svg.append("image")
        .attr("x", marginLeft)
        .attr("y", marginTop)
        .attr("width", width - marginLeft - marginRight)
        .attr("height", height - marginTop - marginBottom)
        .attr("preserveAspectRatio", "none")
        .attr("xlink:href", ramp(color.copy().domain(d3.quantize(d3.interpolate(0, 1), n))).toDataURL());
  }

  // Sequential
  else if (color.interpolator) {
    x = Object.assign(color.copy()
        .interpolator(d3.interpolateRound(marginLeft, width - marginRight)),
        {range() { return [marginLeft, width - marginRight]; }});

    svg.append("image")
        .attr("x", marginLeft)
        .attr("y", marginTop)
        .attr("width", width - marginLeft - marginRight)
        .attr("height", height - marginTop - marginBottom)
        .attr("preserveAspectRatio", "none")
        .attr("xlink:href", ramp(color.interpolator()).toDataURL());

    // scaleSequentialQuantile doesn’t implement ticks or tickFormat.
    if (!x.ticks) {
      if (tickValues === undefined) {
        const n = Math.round(ticks + 1);
        tickValues = d3.range(n).map(i => d3.quantile(color.domain(), i / (n - 1)));
      }
      if (typeof tickFormat !== "function") {
        tickFormat = d3.format(tickFormat === undefined ? ",f" : tickFormat);
      }
    }
  }

  // Threshold
  else if (color.invertExtent) {
    const thresholds
        = color.thresholds ? color.thresholds() // scaleQuantize
        : color.quantiles ? color.quantiles() // scaleQuantile
        : color.domain(); // scaleThreshold

    const thresholdFormat
        = tickFormat === undefined ? d => d
        : typeof tickFormat === "string" ? d3.format(tickFormat)
        : tickFormat;

    x = d3.scaleLinear()
        .domain([-1, color.range().length - 1])
        .rangeRound([marginLeft, width - marginRight]);

    svg.append("g")
      .selectAll("rect")
      .data(color.range())
      .join("rect")
        .attr("x", (d, i) => x(i - 1))
        .attr("y", marginTop)
        .attr("width", (d, i) => x(i) - x(i - 1))
        .attr("height", height - marginTop - marginBottom)
        .attr("fill", d => d);

    tickValues = d3.range(thresholds.length);
    tickFormat = i => thresholdFormat(thresholds[i], i);
  }

  // Ordinal
  else {
    x = d3.scaleBand()
        .domain(color.domain())
        .rangeRound([marginLeft, width - marginRight]);

    svg.append("g")
      .selectAll("rect")
      .data(color.domain())
      .join("rect")
        .attr("x", x)
        .attr("y", marginTop)
        .attr("width", Math.max(0, x.bandwidth() - 1))
        .attr("height", height - marginTop - marginBottom)
        .attr("fill", color);

    tickAdjust = () => {};
  }

  svg.append("g")
      .attr("transform", `translate(0,${height - marginBottom})`)
      .call(d3.axisBottom(x)
        .ticks(ticks, typeof tickFormat === "string" ? tickFormat : undefined)
        .tickFormat(typeof tickFormat === "function" ? tickFormat : undefined)
        .tickSize(tickSize)
        .tickValues(tickValues))
      .call(tickAdjust)
      .call(g => g.select(".domain").remove())
      .call(g => g.append("text")
        .attr("x", marginLeft)
        .attr("y", marginTop + marginBottom - height - 6)
        .attr("fill", "currentColor")
        .attr("text-anchor", "start")
        .attr("font-weight", "bold")
        .attr("class", "title")
        .text(title));

  return svg.node();
}

// Copyright 2021, Observable Inc.
// Released under the ISC license.
// https://observablehq.com/@d3/color-legend
function Swatches(color, {
  columns = null,
  format,
  unknown: formatUnknown,
  swatchSize = 15,
  swatchWidth = swatchSize,
  swatchHeight = swatchSize,
  marginLeft = 0
} = {}) {
  const id = `-swatches-${Math.random().toString(16).slice(2)}`;
  const unknown = formatUnknown == null ? undefined : color.unknown();
  const unknowns = unknown == null || unknown === d3.scaleImplicit ? [] : [unknown];
  const domain = color.domain().concat(unknowns);
  if (format === undefined) format = x => x === unknown ? formatUnknown : x;

  function entity(character) {
    return `&#${character.charCodeAt(0).toString()};`;
  }

  if (columns !== null) return htl.html`<div style="display: flex; align-items: center; margin-left: ${+marginLeft}px; min-height: 33px; font: 10px sans-serif;">
  <style>

.${id}-item {
  break-inside: avoid;
  display: flex;
  align-items: center;
  padding-bottom: 1px;
}

.${id}-label {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: calc(100% - ${+swatchWidth}px - 0.5em);
}

.${id}-swatch {
  width: ${+swatchWidth}px;
  height: ${+swatchHeight}px;
  margin: 0 0.5em 0 0;
}

  </style>
  <div style=${{width: "100%", columns}}>${domain.map(value => {
    const label = `${format(value)}`;
    return htl.html`<div class=${id}-item>
      <div class=${id}-swatch style=${{background: color(value)}}></div>
      <div class=${id}-label title=${label}>${label}</div>
    </div>`;
  })}
  </div>
</div>`;

  return htl.html`<div style="display: flex; align-items: center; min-height: 33px; margin-left: ${+marginLeft}px; font: 10px sans-serif;">
  <style>

.${id} {
  display: inline-flex;
  align-items: center;
  margin-right: 1em;
}

.${id}::before {
  content: "";
  width: ${+swatchWidth}px;
  height: ${+swatchHeight}px;
  margin-right: 0.5em;
  background: var(--color);
}

  </style>
  <div>${domain.map(value => htl.html`<span class="${id}" style="--color: ${color(value)}">${format(value)}</span>`)}</div>`;
}

function swatches({color, ...options}) {
  return Swatches(color, options);
}

```
</details>