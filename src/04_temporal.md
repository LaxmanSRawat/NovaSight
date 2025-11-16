# Temporal Analysis
<br>

## What is the average duration from call initiation to incident closure for the most common incident types? 

<br>

###  Possible Visualization: Horizontal bar chart showing top 10-15 radio codes with average total duration

<br>


![NYC Incident Average Call Duration](nyc-avg-duration.png)

</br>

### Loading the top 10 crime and potential crime category by median of call duration data processed using <a href="https://github.com/LaxmanSRawat/NovaSight/blob/main/temporal_analysis_top_10_category_by_average_total_duration.ipynb" rel="external">python jupyter notebook</a>

<br>

```js echo
// Load top 10 crime category by call duration data
const crime_top_10_categories_by_incident_duration = FileAttachment("nyc_data_crime_top_10_categories_by_call_duration.csv").csv({typed: true})
```

<br>

```js echo
crime_top_10_categories_by_incident_duration
```

<br>

```js echo

  // Set up dimensions
  const margin = { top: 40, right: 60, bottom: 60, left: 330 };
  const width = 900 - margin.left - margin.right;
  const height = 500 - margin.top - margin.bottom;

  // Group data by typ_desc
  const groupedData = d3.group(crime_top_10_categories_by_incident_duration, d => d.typ_desc);
  const boxPlotData = Array.from(groupedData, ([key, values]) => ({
    type: key,
    stats: calculateBoxPlotStats(values.map(d => d.call_duration_minutes)),
    values: values.map(d => d.call_duration_minutes)
  }));

  // Sort by median in descending order
  boxPlotData.sort((a, b) =>b.stats.median - a.stats.median);

  // Create SVG
  const svg = d3.create("svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", height + margin.top + margin.bottom)
    .attr("viewBox", [0, 0, width + margin.left + margin.right, height + margin.top + margin.bottom])
    .style("max-width", "100%")
    .style("height", "auto")
    .style("background", "#253f4b");

  const g = svg.append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);

  // Create scales (swapped x and y)
  const yScale = d3.scaleBand()
    .domain(boxPlotData.map(d => d.type))
    .range([0, height])
    .padding(0.3);

  const xScale = d3.scaleLinear()
    .domain([0, d3.max(boxPlotData, d => d3.max([d.stats.max, ...d.stats.outliers]))])
    .nice()
    .range([0, width]);

  console.log(d3.max(boxPlotData, d => d3.max([d.stats.max, ...d.stats.outliers])))

  // Create axes
  g.append("g")
    .attr("class", "x-axis")
    .attr("transform", `translate(0,${height})`)
    .call(d3.axisBottom(xScale))
    .selectAll("text")
    .style("font-size", "12px")
    .style("color", "white")
    .style("font-family", "Arial");

  g.append("g")
    .attr("class", "y-axis")
    .call(d3.axisLeft(yScale))
    .selectAll("text")
    .style("font-size", "12px")
    .style("font-family", "Arial");

  // Add axis labels
  g.append("text")
    .attr("text-anchor", "middle")
    .attr("x", width / 2)
    .attr("y", height + margin.bottom - 10)
    .style("font-size", "14px")
    .style("font-weight", "bold")
    .text("INCIDENT DURATION (MINUTES)")
    .style("fill", "#dfdfd6")
    .style("font-family", "Arial");

  g.append("text")
    .attr("text-anchor", "middle")
    .attr("transform", "rotate(-90)")
    .attr("x", -height / 2)
    .attr("y", -margin.left + 15)
    .style("font-size", "14px")
    .style("font-weight", "bold")
    .text("INCIDENT TYPE")
    .style("fill", "#dfdfd6")
    .style("font-family", "Arial");

  // Draw box plots
  const boxHeight = yScale.bandwidth();

  boxPlotData.forEach(d => {
    const y = yScale(d.type);
    const center = y + boxHeight / 2;

    const boxGroup = g.append("g");

    // Horizontal line from min to Q1
    boxGroup.append("line")
      .attr("x1", xScale(d.stats.min))
      .attr("x2", xScale(d.stats.q1))
      .attr("y1", center)
      .attr("y2", center)
      .attr("stroke", "#000")
      .attr("stroke-width", 1.5);

    // Horizontal line from Q3 to max
    boxGroup.append("line")
      .attr("x1", xScale(d.stats.q3))
      .attr("x2", xScale(d.stats.max))
      .attr("y1", center)
      .attr("y2", center)
      .attr("stroke", "#000")
      .attr("stroke-width", 1.5);

    // Min vertical line
    boxGroup.append("line")
      .attr("x1", xScale(d.stats.min))
      .attr("x2", xScale(d.stats.min))
      .attr("y1", center - boxHeight / 4)
      .attr("y2", center + boxHeight / 4)
      .attr("stroke", "#000")
      .attr("stroke-width", 1.5);

    // Max vertical line
    boxGroup.append("line")
      .attr("x1", xScale(d.stats.max))
      .attr("x2", xScale(d.stats.max))
      .attr("y1", center - boxHeight / 4)
      .attr("y2", center + boxHeight / 4)
      .attr("stroke", "#000")
      .attr("stroke-width", 1.5);

    // Box (IQR)
    boxGroup.append("rect")
      .attr("x", xScale(d.stats.q1))
      .attr("y", y)
      .attr("width", xScale(d.stats.q3) - xScale(d.stats.q1))
      .attr("height", boxHeight)
      .attr("fill", "steelblue")
      .attr("opacity", 0.7)
      .attr("stroke", "#000")
      .attr("stroke-width", 1);

    // Median line
    boxGroup.append("line")
      .attr("x1", xScale(d.stats.median))
      .attr("x2", xScale(d.stats.median))
      .attr("y1", y)
      .attr("y2", y + boxHeight)
      .attr("stroke", "#000")
      .attr("stroke-width", 2);

    // Outliers
    d.stats.outliers.forEach(outlier => {
      boxGroup.append("circle")
        .attr("cx", xScale(outlier))
        .attr("cy", center)
        .attr("r", 3)
        .attr("fill", "#e74c3c")
        .attr("opacity", 0.6);
    });
  });

  display(svg.node());
```

<br>

## How have 911 call volumes and incident types changed over the year? 

### Possible Visualization:  Time series line chart showing monthly trends by major incident category 

<br>

![NYC Incident Time Series](nyc-incident-time-series.png)

<br>

## How does the distribution of call types vary throughout the day? 


### Possible Visualization: heatmap showing incident categories by hour of day

<br>

![!NYC 24hr Incident Heatmap](nyc-24hr-incident-heatmap.png)

#### Appendix


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
```
