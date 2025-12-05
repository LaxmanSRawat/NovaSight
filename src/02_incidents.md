---
style: text-style.css
---

<!-- Questions and Findings - For each question:
Clear question statement
Polished D3 visualization
Analysis and interpretation
Insights and implications -->

# Behind Every Call: The Incident Reported
<br>

## Data Loading
<details>
Import processed data that consists of counts of Main category incidents.

Main Categories: ASSAULT, BURGLARY, DISORDERLY, LARCENY, OTHER CRIMES, OTHER-CRIME INCIDENT, ROBBERY

```js echo
import * as d3 from "npm:d3";
const crime_data = await FileAttachment("incident_counts.csv").csv()
display(crime_data)
```

All Categories (including sub-categories):
```js echo
const all_categories = await FileAttachment("all_incident_counts.csv").csv()
display(all_categories)
```
</details>

<br>

To understand what drives New York City’s emergency response system, we first analyzed the types of incidents that generate 911 calls. While the 911 system is often associated with serious, high-risk events, the data reveals a different reality: **the city’s call volume is overwhelmingly shaped by low-severity, high-frequency incidents.**

## What type of incidents are most common?

```js 
const rows = all_categories.map(d => ({ incident_main: d.incident_main.trim(), incident_sub: d.incident_sub.trim(), incident_subsub: d.incident_subsub.trim(), count: +d.count }));

const incidentHierarchy = (() => {
  const nested = d3.rollup(
    rows,
    v => d3.sum(v, d => d.count),
    d => d.incident_main,
    d => d.incident_sub,
    d => d.incident_subsub
  );

  function mapToTree(name, value) {
    if (value instanceof Map) {
      return {
        name,
        children: Array.from(value, ([k, v]) => mapToTree(k, v))
      };
    } else {
      return { name, value };
    }
  }

  return mapToTree("All incidents", nested);
})();

```
```js 
const chart = (() => {
  const width = 900;
  const height = width;
  const radius = width / 7;

  // Create the color scale.
  const color = d3.scaleOrdinal(
    d3.quantize(d3.interpolateWarm, incidentHierarchy.children.length + 1)
  );

  // Compute the layout.
  const hierarchy = d3.hierarchy(incidentHierarchy)
      .sum(d => d.value)
      .sort((a, b) => b.value - a.value);

  const root = d3.partition()
      .size([2 * Math.PI, hierarchy.height + 1])
    (hierarchy);

  root.each(d => d.current = d);

  const total = root.value;
  const format = d3.format(",d");

  // Arc generator.
  const arc = d3.arc()
      .startAngle(d => d.x0)
      .endAngle(d => d.x1)
      .padAngle(d => Math.min((d.x1 - d.x0) / 2, 0.005))
      .padRadius(radius * 1.5)
      .innerRadius(d => d.y0 * radius)
      .outerRadius(d => Math.max(d.y0 * radius, d.y1 * radius - 1));

  // SVG container.
  const svg = d3.create("svg")
      .attr("viewBox", [-width / 2, -height / 2, width, width])
      .style("font", "12px sans-serif")
      .style("font-weight",500)
      .style("font-family", "Arial")
      .style("background","#bcbcb4ff")
      .style("fill","#000")
      .style("color","#000")
      ;

  const path = svg.append("g")
    .selectAll("path")
    .data(root.descendants().slice(1))
    .join("path")
      .attr("fill", d => { while (d.depth > 1) d = d.parent; return color(d.data.name); })
      .attr("fill-opacity", d => arcVisible(d.current) ? (d.children ? 1 : 0.6) : 0)
      // .style("stroke", d => arcVisible(d.current) ? (d.children ? "#000" : "#dfdfd6") : "#dfdfd6")
      .style("stroke-width",1)
      .attr("pointer-events", d => arcVisible(d.current) ? "auto" : "none")
      .attr("d", d => arc(d.current))
      ;

  path.filter(d => d.children)
      .style("cursor", "pointer")
      .on("click", clicked);

  // Tooltip
  path.append("title")
    .text(d => {
      const pathNames = d.ancestors().map(d => d.data.name).reverse().join("/");
      const pct = (d.value / total) * 100;
      return `${pathNames}\n${format(d.value)} (${pct.toFixed(1)}%)`;
    });

  const label = svg.append("g")
      .attr("pointer-events", "none")
      .attr("text-anchor", "middle")
      .style("user-select", "none")
    .selectAll("text")
    .data(root.descendants().slice(1))
    .join("text")
      .attr("dy", "0.35em")
      .attr("fill-opacity", d => +labelVisible(d.current))
      .attr("transform", d => labelTransform(d.current))
      .text(d => {
      const pct = (d.value / total) * 100;
      return `${d.data.name} (${pct.toFixed(1)}%)`;
    });

  const parent = svg.append("circle")
      .datum(root)
      .attr("r", radius)
      .attr("fill", "none")
      .attr("pointer-events", "all")
      .on("click", clicked);

  function clicked(event, p) {
    parent.datum(p.parent || root);

    root.each(d => d.target = {
      x0: Math.max(0, Math.min(1, (d.x0 - p.x0) / (p.x1 - p.x0))) * 2 * Math.PI,
      x1: Math.max(0, Math.min(1, (d.x1 - p.x0) / (p.x1 - p.x0))) * 2 * Math.PI,
      y0: Math.max(0, d.y0 - p.depth),
      y1: Math.max(0, d.y1 - p.depth)
    });

    const t = svg.transition().duration(event.altKey ? 7500 : 750);

    path.transition(t)
        .tween("data", d => {
          const i = d3.interpolate(d.current, d.target);
          return t => d.current = i(t);
        })
      .filter(function(d) {
        return +this.getAttribute("fill-opacity") || arcVisible(d.target);
      })
        .attr("fill-opacity", d => arcVisible(d.target) ? (d.children ? 0.8 : 0.6) : 0)
        // .style("stroke", d => arcVisible(d.target) ? "#000" : "#dfdfd6")
        .attr("pointer-events", d => arcVisible(d.target) ? "auto" : "none")
        .attrTween("d", d => () => arc(d.current));

    label.filter(function(d) {
        return +this.getAttribute("fill-opacity") || labelVisible(d.target);
      }).transition(t)
        .attr("fill-opacity", d => +labelVisible(d.target))
        .attrTween("transform", d => () => labelTransform(d.current));
  }
  
  function arcVisible(d) {
    return d.y1 <= 3 && d.y0 >= 1 && d.x1 > d.x0;
  }

  function labelVisible(d) {
    return d.y1 <= 3 && d.y0 >= 1 && (d.y1 - d.y0) * (d.x1 - d.x0) > 0.03;
  }

  function labelTransform(d) {
    const x = (d.x0 + d.x1) / 2 * 180 / Math.PI;
    const y = (d.y0 + d.y1) / 2 * radius;
    return `rotate(${x - 90}) translate(${y},0) rotate(${x < 180 ? 0 : 180})`;
  }

  const node = svg.node();
  display(node);   
})();


```

<details>
<summary>Code</summary>

```js echo
const chart = (() => {
  const width = 900;
  const height = width;
  const radius = width / 7;

  // Create the color scale.
  const color = d3.scaleOrdinal(
    d3.quantize(d3.interpolateSpectral, incidentHierarchy.children.length + 1)
  );

  // Compute the layout.
  const hierarchy = d3.hierarchy(incidentHierarchy)
      .sum(d => d.value)
      .sort((a, b) => b.value - a.value);

  const root = d3.partition()
      .size([2 * Math.PI, hierarchy.height + 1])
    (hierarchy);

  root.each(d => d.current = d);

  const total = root.value;
  const format = d3.format(",d");

  // Arc generator.
  const arc = d3.arc()
      .startAngle(d => d.x0)
      .endAngle(d => d.x1)
      .padAngle(d => Math.min((d.x1 - d.x0) / 2, 0.005))
      .padRadius(radius * 1.5)
      .innerRadius(d => d.y0 * radius)
      .outerRadius(d => Math.max(d.y0 * radius, d.y1 * radius - 1));

  // SVG container.
  const svg = d3.create("svg")
      .attr("viewBox", [-width / 2, -height / 2, width, width])
      .style("font", "10px sans-serif")
      .style("font-family", "Arial")
      .style("background","#dfdfd6")
      .style("fill","#000")
      .style("color","#000")
      ;

  const path = svg.append("g")
    .selectAll("path")
    .data(root.descendants().slice(1))
    .join("path")
      .attr("fill", d => { while (d.depth > 1) d = d.parent; return color(d.data.name); })
      .attr("fill-opacity", d => arcVisible(d.current) ? (d.children ? 1 : 0.6) : 0)
      .style("stroke", d => arcVisible(d.current) ? (d.children ? "#000" : "#dfdfd6") : "#dfdfd6")
      .style("stroke-width",1)
      .attr("pointer-events", d => arcVisible(d.current) ? "auto" : "none")
      .attr("d", d => arc(d.current))
      ;

  path.filter(d => d.children)
      .style("cursor", "pointer")
      .on("click", clicked);

  // Tooltip
  path.append("title")
    .text(d => {
      const pathNames = d.ancestors().map(d => d.data.name).reverse().join("/");
      const pct = (d.value / total) * 100;
      return `${pathNames}\n${format(d.value)} (${pct.toFixed(1)}%)`;
    });

  const label = svg.append("g")
      .attr("pointer-events", "none")
      .attr("text-anchor", "middle")
      .style("user-select", "none")
    .selectAll("text")
    .data(root.descendants().slice(1))
    .join("text")
      .attr("dy", "0.35em")
      .attr("fill-opacity", d => +labelVisible(d.current))
      .attr("transform", d => labelTransform(d.current))
      .text(d => {
      const pct = (d.value / total) * 100;
      return `${d.data.name} (${pct.toFixed(1)}%)`;
    });

  const parent = svg.append("circle")
      .datum(root)
      .attr("r", radius)
      .attr("fill", "none")
      .attr("pointer-events", "all")
      .on("click", clicked);

  function clicked(event, p) {
    parent.datum(p.parent || root);

    root.each(d => d.target = {
      x0: Math.max(0, Math.min(1, (d.x0 - p.x0) / (p.x1 - p.x0))) * 2 * Math.PI,
      x1: Math.max(0, Math.min(1, (d.x1 - p.x0) / (p.x1 - p.x0))) * 2 * Math.PI,
      y0: Math.max(0, d.y0 - p.depth),
      y1: Math.max(0, d.y1 - p.depth)
    });

    const t = svg.transition().duration(event.altKey ? 7500 : 750);

    path.transition(t)
        .tween("data", d => {
          const i = d3.interpolate(d.current, d.target);
          return t => d.current = i(t);
        })
      .filter(function(d) {
        return +this.getAttribute("fill-opacity") || arcVisible(d.target);
      })
        .attr("fill-opacity", d => arcVisible(d.target) ? (d.children ? 0.8 : 0.6) : 0)
        .style("stroke", d => arcVisible(d.target) ? "#000" : "#dfdfd6")
        .attr("pointer-events", d => arcVisible(d.target) ? "auto" : "none")
        .attrTween("d", d => () => arc(d.current));

    label.filter(function(d) {
        return +this.getAttribute("fill-opacity") || labelVisible(d.target);
      }).transition(t)
        .attr("fill-opacity", d => +labelVisible(d.target))
        .attrTween("transform", d => () => labelTransform(d.current));
  }
  
  function arcVisible(d) {
    return d.y1 <= 3 && d.y0 >= 1 && d.x1 > d.x0;
  }

  function labelVisible(d) {
    return d.y1 <= 3 && d.y0 >= 1 && (d.y1 - d.y0) * (d.x1 - d.x0) > 0.03;
  }

  function labelTransform(d) {
    const x = (d.x0 + d.x1) / 2 * 180 / Math.PI;
    const y = (d.y0 + d.y1) / 2 * radius;
    return `rotate(${x - 90}) translate(${y},0) rotate(${x < 180 ? 0 : 180})`;
  }

  const node = svg.node();
  return node;   
})();
```
</details>

<br>

### Analysis & Insights

#### The following categories make up the overwhelming majority of NYC’s 911 call activity:
- **Other Crimes (~36.7%):** This category alone accounts for more than one-third of all recorded incidents. Its size reflects the broad range of relatively lower-severity complaints that residents report.
- **Larceny (~27.0%):** Theft-related calls form the second-largest block, driven by both home thefts and outside thefts (non-transit).
- **Disorderly (~17.9%):** Disturbance-related behavior—fights, noise complaints and public disorder.

<div class="insight"><strong>Insight 💡</strong><br> NYC’s emergency call system is disproportionately shaped by frequent but relatively non-critical events. The prominence of Other Crimes reflects that 911 is often used for low-level disputes and public-order issues, not clear-cut criminal events. This is similar with Disorderly and Assault categories which represent mild disturbances to physical conflict. Assault, despite being serious, appears far less frequently than “Other Crimes,” underscoring how much 911 traffic is driven by low-intensity social disruptions. This indicates that NYPD might be functioning as a default responder for ambiguous or socially complex situations that may not require armed law enforcement. </div>

#### **Notable subcategory patterns**
- **Harassment** is the largest subcategory overall (≈ 21.3% of all incidents) and is the most common in *Other Crimes*.  
- **Person-related larcenies** and **vehicle-related larcenies** together make Larceny one of the biggest major categories.  
- **Disorderly incidents** are made up mostly of *Person* (13.0%) and *Group* (5.0%) disturbances.  
- **Assault** is mostly categorised into 'Other' which further breaks into the most common assault types being Inside and Family Related.

<div class="insight"><strong>Insight 💡</strong><br> A substantial portion of 911 calls relate to ongoing interpersonal tension rather than acute emergencies (Harassment). Disordely incidents are largely composed of Person and Group disturbances which can be suggestive of overall public behavior and street-level tensions. Such calls may benefit more from civilian crisis teams, or social workers than police intervention. </div>
<br>

Having understood how incident types are distributed, the next question is: how severe are these incidents? To explore this, we examine the severity levels associated with each main category.


## How do incidents vary by severity level? 

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
  const width = 900, height = 600;
  const margin = { top: 60, right: 40, bottom: 100, left: 70 };

  const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .attr("width", width)
    .attr("height", height)
    .style("background","#dfdfd6")
    .style("font-family", "Arial");

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
      .attr("fill", color(key))
      .style;

    g.append("text")
      .attr("x", 20)
      .attr("y", 11)
      .attr("font-size", 12)
      .attr("fill", "black")
      .text(key);
  });

  // bars
  const bars = svg.selectAll("g.layer")
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
      .attr("y", d => y(d.y1))
      .attr("height", d => y(d.y0) - y(d.y1))
      .attr("width", x.bandwidth());

const segments = series.flatMap(s =>
  s.map(v => ({
    key: s.key,
    incident_main: v.data.incident_main,
    y0: v[0],
    y1: v[1],
    value: v[1] - v[0]
  }))
);

const MIN_LABEL_PX = 12;

// labels for large segments
svg.append("g")
  .selectAll("text.inside-label")
  .data(segments.filter(d => (y(d.y0) - y(d.y1)) >= MIN_LABEL_PX))
  .join("text")
    .attr("class", "inside-label")
    .attr("x", d => x(d.incident_main) + x.bandwidth() / 2)
    .attr("y", d => (y(d.y0) + y(d.y1)) / 2 + 3)
    .attr("text-anchor", "middle")
    .attr("font-size", 10)
    .attr("fill", "black")
    .text(d => d.value);

// labels for small segments
svg.append("g")
  .selectAll("text.outside-label")
  .data(segments.filter(d => (y(d.y0) - y(d.y1)) < MIN_LABEL_PX && d.value > 0))
  .join("text")
    .attr("class", "outside-label")
    .attr("x", d => x(d.incident_main) + 8)
    .attr("y", d => y(d.y1) - 2)
    .attr("text-anchor", "start")
    .attr("font-size", 10)
    .attr("fill", "black")
    .text(d => `${d.value} (${d.key})`);
  
  // axes
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(x))
    .style("color","#000")
    .selectAll("text")
      .attr("text-anchor", "end")
      .attr("transform", "rotate(-35)")
      .attr("dx", "-0.4em")
      .attr("dy", "0.2em");
  
  const ticks = d3.range(0, ymax + 5000, 50000)
  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).tickValues(ticks).tickFormat(d3.format(",d")))
    .style("color","#000");

  // title
  svg.append("text")
    .attr("x", width / 2)
    .attr("y", margin.top - 30)
    .attr("text-anchor", "middle")
    .attr("font-size", 16)
    .attr("font-weight", "bold")
    .style("fill","#000")
    .style("color","#000")
    .text("Severity Levels of Main Incident Types");

  display(svg.node());
})();

```
<details>
<summary> Code </summary>

```js echo
const chart = (() => {
  const width = 900, height = 600;
  const margin = { top: 60, right: 40, bottom: 100, left: 70 };

  const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .attr("width", width)
    .attr("height", height)
    .style("background","#dfdfd6")
    .style("font-family", "Arial");

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
      .attr("fill", color(key))
      .style;

    g.append("text")
      .attr("x", 20)
      .attr("y", 11)
      .attr("font-size", 12)
      .attr("fill", "black")
      .text(key);
  });

  // bars
  const bars = svg.selectAll("g.layer")
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
      .attr("y", d => y(d.y1))
      .attr("height", d => y(d.y0) - y(d.y1))
      .attr("width", x.bandwidth());

const segments = series.flatMap(s =>
  s.map(v => ({
    key: s.key,
    incident_main: v.data.incident_main,
    y0: v[0],
    y1: v[1],
    value: v[1] - v[0]
  }))
);

const MIN_LABEL_PX = 12;

// labels for large segments
svg.append("g")
  .selectAll("text.inside-label")
  .data(segments.filter(d => (y(d.y0) - y(d.y1)) >= MIN_LABEL_PX))
  .join("text")
    .attr("class", "inside-label")
    .attr("x", d => x(d.incident_main) + x.bandwidth() / 2)
    .attr("y", d => (y(d.y0) + y(d.y1)) / 2 + 3)
    .attr("text-anchor", "middle")
    .attr("font-size", 10)
    .attr("fill", "black")
    .text(d => d.value);

// labels for small segments
svg.append("g")
  .selectAll("text.outside-label")
  .data(segments.filter(d => (y(d.y0) - y(d.y1)) < MIN_LABEL_PX && d.value > 0))
  .join("text")
    .attr("class", "outside-label")
    .attr("x", d => x(d.incident_main) + 8)
    .attr("y", d => y(d.y1) - 2)
    .attr("text-anchor", "start")
    .attr("font-size", 10)
    .attr("fill", "black")
    .text(d => `${d.value} (${d.key})`);
  
  // axes
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(x))
    .style("color","#000")
    .selectAll("text")
      .attr("text-anchor", "end")
      .attr("transform", "rotate(-35)")
      .attr("dx", "-0.4em")
      .attr("dy", "0.2em");
  
  const ticks = d3.range(0, ymax + 5000, 50000)
  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).tickValues(ticks).tickFormat(d3.format(",d")))
    .style("color","#000");

  // title
  svg.append("text")
    .attr("x", width / 2)
    .attr("y", margin.top - 30)
    .attr("text-anchor", "middle")
    .attr("font-size", 16)
    .attr("font-weight", "bold")
    .style("fill","#000")
    .style("color","#000")
    .text("Severity Levels of Main Incident Types");

  svg.node();
})();

```
</details>
<br>

### Analysis & Insights
#### **Most severe incident categories**
- Larceny — Highest number of Critical + Serious Incidents
- Assault
- Burglary  

<div class="insight"><strong>Insight 💡</strong><br> 1. Disorderly and Other-Crime Incident stand out as the least severe categories (similar to what we inferred earlier).
Every incident within these groups is classified as Non-CIP (not in progress), meaning they typically reflect situations that have already ended, minor disturbances, or low-threat public concerns.

2. Assault, Burglary, Larceny, and Robbery contain Critical Incidents, marking them as the primary contributors to high-urgency response workloads. These incidents are imminent or ongoing threats and therefore place the greatest demands on rapid police intervention.

3. Larceny, Disorderly, and Other Crimes have the highest number of Non-CIP reports. This suggests these categories often involve victims or witnesses calling after the event has concluded. These call types inflate total workload without addressing immediate danger which can call for differentiated staffing strategies.
</div>
