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
 In recent years, the NYPD has experienced a dramatic drop in staffing due to high rates of retirement and resignations: losing roughly 300 officers per month on average. This has led to the lowest headcount in decades, making precincts across the boroughs understaffed. This attrition has coincided with rising response times and increasing calls per officer [1].  ￼
 
At the same time, it is known that several 911 calls are attributed to non-critical incidents (an average of 62.6% in nationwide calls) [2]. This mismatch between high call volume (driven by social behaviour and public-safety concerns) and dwindling policing capacity creates a structural strain on emergency response, prompting a critical need to understand call patterns, allocate resources efficiently, and explore alternative response models.

Sources: 
1. https://www.amny.com/news/nypd-attrition-crisis-longer-police-response-times/
2. https://www.vera.org/news/most-911-calls-have-nothing-to-do-with-crime-why-are-we-still-sending-police

### Background

New York City’s 911 system receives millions of calls annually and serves as the city’s primary pathway for emergency assistance this ranges from concerned regarding crime, disorder, medical distress, and even general public safety concerns. As a primary dispatch system for the NYPD, the system classifies each call, assesses for severity and deploys resources to meet demand. Emergency calling behaviour varies widely across neighborhoods due to differences in population density, socioeconomic factors, community-police relations, and perceived threat levels. Understanding these dynamics is crucial for effective staffing, resource allocation, and the development of alternative response strategies. With rising national pressure on emergency systems and increasing reliance on police for non-criminal issues, analyzing NYC’s 911 trends provides valuable insights into both citizen behavior and operational strain.

### Motivation

In light of increasing response times, shrinking police staffing, and rising non-critical call volume, this project aims to use data-driven analysis to uncover how 911 calls are distributed across New York City. By examining call types, severity, geographic concentration, and temporal trends, we seek to identify structural inefficiencies and highlight opportunities for more effective resource deployment and alternative response strategies.

### Overview of Findings

Our analysis reveals that New York City’s 911 system is driven more by high call volume than by high-severity emergencies. The majority of incidents fall into non-critical categories, with interpersonal disputes, harassment, disorderly conduct, and minor thefts representing a substantial share of all calls. These patterns vary significantly across boroughs: Brooklyn and the Bronx carry the heaviest burdens, with the Bronx exhibiting the longest call durations and Brooklyn generating a disproportionately high number of potential crime reports. Temporal analysis indicates highly predictable weekly cycles and consistent year-round demand, while precinct-level comparisons show that a small number of precincts account for a large share of incidents. Together, these findings highlight potential for targeted staffing, improved triage, and alternative response models.

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