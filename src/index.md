---
theme: dashboard
title: Hydro Analytics - Centrale Lyon
toc: false
---

# Hydro Analytics

```js
const launches = FileAttachment("data/launches.csv").csv({typed: true});

const launches2 = FileAttachment("data/waterplease.csv").csv({
  typed: true,
  columns: {
    Year: Number,
    Value: Number,
    Area: String,
    VariableGroup: String,
    Subgroup: String,
    Variable: String,
    Unit: String,
    Symbol: String,
    IsAggregate: d => d === "False"
  }
});
const waterData = await launches2 

// Country selector
const countrySelector = Inputs.select(
  [...new Set(waterData.map(d => d.Area))].sort(),
  {
    label: "Select Country",
    value: [...new Set(waterData.map(d => d.Area))][0]
  }
);
```
<div class="grid grid-cols-4">
  <div class="card">
    <h2>Selected Country</h2>
    <span class="big">${countrySelector}</span>
  </div>
  <div class="card">
    <h2>Average Withdrawal</h2>
    <span class="big">${d3.format(".1f")(d3.mean(waterData.filter(d => d.Area === countrySelector), d => d.Value))}</span>
    <span class="muted">10^9 m3/year</span>
  </div>
  <div class="card">
    <h2>Years Covered</h2>
    <span class="big">${d3.extent(waterData.filter(d => d.Area === countrySelector), d => d.Year).join("-")}</span>
  </div>
  <div class="card">
    <h2>Total Records</h2>
    <span class="big">${waterData.filter(d => d.Area === countrySelector).length.toLocaleString("en-US")}</span>
  </div>
</div>


```js
// Line chart for selected country
function waterLineChart(data, country, {width} = {}) {
  const filteredData = data.filter(d => d.Area === country);
  
  return Plot.plot({
    title: `Water Withdrawal Trends for ${country}`,
    width,
    height: 400,
    y: {
      grid: true, 
      label: "Water Withdrawal (10^9 m3/year)"
    },
    x: {
      label: "Year",
      tickRotate: 45
    },
    color: {...colorData, legend: true},
    marks: [
      Plot.line(filteredData, {
        x: "Year",
        y: "Value",
        stroke: "Variable",
        strokeWidth: 2
      }),
      Plot.dot(filteredData, {
        x: "Year",
        y: "Value",
        fill: "Variable",
        stroke: "white",
        strokeWidth: 1,
        tip: true
      }),
      Plot.ruleY([0])
    ]
  });
}

// Bar chart showing distribution by variable
function waterBarChart(data, country, {width} = {}) {
  const filteredData = data.filter(d => d.Area === country);
  
  return Plot.plot({
    title: `Water Withdrawal by Category for ${country}`,
    width,
    height: 300,
    marginLeft: 150,
    x: {grid: true, label: "Average Withdrawal (10^9 m3/year)"},
    y: {label: null},
    color: {...colorData, legend: true},
    marks: [
      Plot.barX(filteredData, 
        Plot.groupY(
          {x: "mean"}, 
          {
            y: "Variable",
            fill: "Variable",
            sort: {y: "-x"},
            tip: true
          }
        )
      ),
      Plot.ruleX([0])
    ]
  });
}
```

<div class="grid grid-cols-1">
  <div class="card">
    ${resize((width) => waterLineChart(waterData, countrySelector, {width}))}
  </div>
</div>

<div class="grid grid-cols-1">
  <div class="card">
    ${resize((width) => waterBarChart(waterData, countrySelector, {width}))}
  </div>
</div>

<style>
.card {
  background: var(--theme-background-secondary);
  padding: 1rem;
  border-radius: 0.5rem;
  margin-bottom: 1rem;
}

.big {
  font-size: 2rem;
  font-weight: bold;
  display: block;
}

.muted {
  color: var(--theme-foreground-muted);
}
</style>
```js
const colorData = Plot.scale({
  color: {
    type: "categorical",
    domain: d3.groupSort(waterData, (D) => -D.length, (d) => d.Area).filter((d) => d !== "Other"),
    unknown: "var(--theme-foreground-muted)"
  }
});

const color = Plot.scale({
  color: {
    type: "categorical",
    domain: d3.groupSort(launches, (D) => -D.length, (d) => d.state).filter((d) => d !== "Other"),
    unknown: "var(--theme-foreground-muted)"
  }
});
```


<div class="grid grid-cols-4">
  <div class="card">
    <h2>Total Areas</h2>
    <span class="big">${new Set(waterData.map(d => d.Area)).size}</span>
  </div>
  <div class="card">
    <h2>Average Withdrawal</h2>
    <span class="big">${d3.format(".1f")(d3.mean(waterData, d => d.Value))}</span>
    <span class="muted">10^9 m3/year</span>
  </div>
  <div class="card">
    <h2>Years Covered</h2>
    <span class="big">${d3.extent(waterData, d => d.Year).join("-")}</span>
  </div>
  <div class="card">
    <h2>Total Records</h2>
    <span class="big">${waterData.length.toLocaleString("en-US")}</span>
  </div>
</div>

```js

// Timeline for water data
function waterTimeline(data, {width} = {}) {
  return Plot.plot({
    title: "Water Withdrawal Over Time",
    width,
    height: 300,
    y: {grid: true, label: "Water Withdrawal (10^9 m3/year)"},
    x: {label: "Year"},
    color: {...colorData, legend: true},
    marks: [
      Plot.rectY(data, Plot.binX({y: "sum"}, {
        x: "Year", 
        y: "Value", 
        fill: "Area", 
        tip: true
      })),
      Plot.ruleY([0])
    ]
  });
}

function launchTimeline(data, {width} = {}) {
  return Plot.plot({
    title: "Launches over the years",
    width,
    height: 300,
    y: {grid: true, label: "Launches"},
    color: {...color, legend: true},
    marks: [
      Plot.rectY(data, Plot.binX({y: "count"}, {x: "date", fill: "state", interval: "year", tip: true})),
      Plot.ruleY([0])
    ]
  });
}
```