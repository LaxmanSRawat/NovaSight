---
style: text-style.css
---

# Data Description

Our analysis uses the NYPD Calls for Service dataset from NYC Open Data, collected through the Computer Aided Dispatch (CAD) system. The data includes incident timestamps, location coordinates, radio codes, priority levels, and response unit information—updated quarterly as part of NYC's transparency initiative.

</br>

<!-- Sources, collection methods, attributes used -->

### Loading the crime data downloaded from NYC Open data using <a href="https://github.com/LaxmanSRawat/NovaSight/blob/main/download_nyc_data_crime.ipynb" rel="external">python jupyter notebook</a>

</br>

### Crime Data

```js echo
// Load the parquet file. This returns a promise to an Arrow table.
const crime_data_promise = await FileAttachment("nyc_data_crime.parquet").parquet()
```

```js echo
// Show the returned promise to the arrow file
crime_data_promise
```

```js echo
// Inspect as array of rows
[...crime_data_promise]
```

```js echo
// Inspect as a table
Inputs.table(crime_data_promise)
```


<!-- 
```js echo
Plot.plot({
  marks: [
    Plot.barY(crime_data_promise, Plot.groupX({y: "count"}, {x: "boro_nm"}))
  ],
  x: {label: "Borough"},
  y: {label: "Count"}
})
```

**Wait!** Why does the null still shows as mark on the x-axis even though we filtered it out while downloading the data? 
Let's investigate it more.

```js echo
// create array of objects for ease of manipulation
crime_data = crime_data
``` -->

### Potential Crime Data

```js echo
// Load the parquet file. This returns a promise to an Arrow table.
const potential_crime_data_promise = await FileAttachment("nyc_data_potential_crime.parquet").parquet()
```

```js echo
// Show the returned promise to the arrow file
potential_crime_data_promise
```

```js echo
// Inspect as array of rows
[...potential_crime_data_promise]
```

```js echo
// Inspect as a table
Inputs.table(potential_crime_data_promise)
```
