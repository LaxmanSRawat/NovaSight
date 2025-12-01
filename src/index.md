---
style: text-style.css
---

<div class="hero">
  <h1>From 911 to Scene <h1>
  <h2>Understanding Emergency Response Patterns to Criminal Activities in NYC</h2>
  <h3>A study by team Novasight <h3>
</div>

<h2> Introduction <h2>

### Problem

TBD

### Background

New York City's 911 system receives millions of calls annually, serving as the primary emergency dispatch mechanism for the NYPD. 

### Motivation

Through comprehensive data analysis, we examine how emergency services respond to criminal activities across the city's five boroughs, revealing patterns in call distribution, response times, and incident resolution.

### Overview of Findings

TBD


<!-- TO DO ### Why This Matters -->
<!-- 
## Key Questions We Explore

<div class="grid grid-cols-2">
  <div class="card">
    <h3>📍 Geographic Patterns</h3>
    <p>Which boroughs and precincts experience the highest volume of emergency calls?</p>
  </div>
  
  <div class="card">
    <h3>🚨 Incident Types</h3>
    <p>What types of incidents are most common, and how do they vary by severity?</p>
  </div>
  
  <div class="card">
    <h3>⏱️ Response Efficiency</h3>
    <p>How long does it take from call initiation to incident closure?</p>
  </div>
  
  <div class="card">
    <h3>📈 Temporal Trends</h3>
    <p>How have call volumes and incident patterns changed over recent years?</p>
  </div>
</div>

## Navigate the Analysis

<div class="grid grid-cols-3">
  <a href="./01_data_overview" class="card">
    <h3>Data Overview</h3>
    <p>Explore the dataset, key metrics, and data quality</p>
  </a>
  
  <a href="./02_geography" class="card">
    <h3>Geographic Analysis</h3>
    <p>Visualize call patterns across boroughs and precincts</p>
  </a>
  
  <a href="./03_incidents" class="card">
    <h3>Incident Analysis</h3>
    <p>Understand incident types, priorities, and response durations</p>
  </a>
  
  <a href="./04_temporal" class="card">
    <h3>Temporal Patterns</h3>
    <p>Discover trends over seasons, and hours of the day</p>
  </a>
  
  <a href="./05_insights" class="card">
    <h3>Insights & Findings</h3>
    <p>Review key discoveries and recommendations</p>
  </a>
  
  <a href="./06_about" class="card">
    <h3>About</h3>
    <p>Learn about our methodology and team</p>
  </a>
</div> -->


<style>
  .{
    font-family: 'Arial';
  }

  .hero {
    display: flex;
    flex-direction: column;
    align-items: center;
    font-family: var(--sans-serif);
    margin: 4rem 0 8rem;
    text-wrap: balance;
    text-align: center;
  }

  .hero h1 {
    /* margin: 1rem 0;
    padding: 1rem 0; */
    max-width: none;
    font-size: 10vw;
    font-weight: 400;
    line-height: 1;
    background: linear-gradient(30deg, var(--theme-foreground-focus), currentColor);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  .hero h2 {
    margin: 0;
    max-width: 34em;
    font-size: 2vw;
    font-style: initial;
    font-weight: 500;
    line-height: 1.5;
    color: var(--theme-foreground-muted);
  }

  .hero h3 {
    margin: 0;
    max-width: 34em;
    font-size: 20px;
    font-style: initial;
    font-weight: 200;
    line-height: 1.5;
    color: var(--theme-foreground-muted);
  }

  @media (min-width: 640px) {
    .hero h1 {
      font-size: 90px;
    }
  }
  
</style>