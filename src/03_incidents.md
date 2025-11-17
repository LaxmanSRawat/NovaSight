# Incident Analysis
### 1. Import processed data that consists of counts of Main category incidents
Main Categories: ASSAULT, BURGLARY, DISORDERLY, LARCENY, OTHER CRIMES, OTHER-CRIME INCIDENT, ROBBERY
```js echo
import * as d3 from "npm:d3";
const crime_data = await FileAttachment("incident_counts.csv").csv()
display(crime_data)
```
```js 
const countsFixed = crime_data.map(d => ({
  incident_main: d.incident_main,
  cip_jobs: d.cip_jobs,         
  count: +d.count || 0          
}));

const cipCategories = Array.from(new Set(countsFixed.map(d => d.cip_jobs))).sort(d3.ascending);
const stackedData = (() => {
  const byIncident = d3.rollups(
    countsFixed,
    v => Object.fromEntries(v.map(d => [d.cip_jobs, d.count])),
    d => d.incident_main
  );
  return byIncident.map(([incident_main, obj]) => {
    const row = { incident_main };
    for (const c of cipCategories) row[c] = obj[c] ?? 0;
    return row;
  });
})();
```
```js
const chart = (() => {
  const width = 900, height = 450;
  const margin = { top: 60, right: 40, bottom: 80, left: 70 };

  const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .attr("width", width)
    .attr("height", height);

  const x = d3.scaleBand()
    .domain(stackedData.map(d => d.incident_main))
    .range([margin.left, width - margin.right])
    .padding(0.3);

  const series = d3.stack().keys(cipCategories)(stackedData);

  const ymax = d3.max(series, s => d3.max(s, d => d[1])) ?? 0;
  const y = d3.scaleLinear()
    .domain([0, ymax]).nice()
    .range([height - margin.bottom, margin.top]);

  const color = d3.scaleOrdinal()
    .domain(cipCategories)
    .range(d3.schemeTableau10.slice(0, Math.max(1, cipCategories.length)));

  // legend
  const legend = svg.append("g")
    .attr("transform", `translate(${width - margin.right - 100}, ${margin.top})`); 

  cipCategories.forEach((key, i) => {
    const g = legend.append("g")
      .attr("transform", `translate(0, ${i * 22})`);

    g.append("rect")
      .attr("width", 14)
      .attr("height", 14)
      .attr("fill", color(key));

    g.append("text")
      .attr("x", 20)
      .attr("y", 11)
      .attr("font-size", 12)
      .attr("fill", "white")
      .text(key);
  });

  // bars
  const MIN_PX = 2;
  svg.selectAll("g.layer")
    .data(series)
    .join("g")
      .attr("class", "layer")
      .attr("fill", d => color(d.key))
    .selectAll("rect")
    .data(d => d.map(v => ({
      key: d.key,
      incident_main: v.data.incident_main,
      y0: v[0],
      y1: v[1]
    })))
    .join("rect")
      .attr("x", d => x(d.incident_main))
      .attr("y", d => {
        const h = y(d.y0) - y(d.y1);
        return h < MIN_PX ? y(d.y1) - (MIN_PX - h) : y(d.y1);
      })
      .attr("height", d => {
        const h = y(d.y0) - y(d.y1);
        return h < MIN_PX ? MIN_PX : h;
      })
      .attr("width", x.bandwidth());

  // axes
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(x))
    .selectAll("text")
      .attr("text-anchor", "end")
      .attr("transform", "rotate(-35)")
      .attr("dx", "-0.4em")
      .attr("dy", "0.2em");

  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).tickFormat(d3.format(",d")));

  // title
  svg.append("text")
    .attr("x", width / 2)
    .attr("y", margin.top - 30)
    .attr("text-anchor", "middle")
    .attr("font-size", 16)
    .attr("font-weight", "bold")
    .attr("fill", "#FFFFFF")
    .text("Incidents by Main Category");

  display(svg.node());
})();

```


<br>

## What types of incidents are most common, and how do they vary by severity level? 

<br>

### Possible Visualization: stacked bar chart showing incident categories, broken down by priority level

<br>

![NYC Incident Frequency](nyc-incident-frequency.png)