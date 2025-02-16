---
theme: dashboard
title: Hydro Analytics - Centrale Lyon
toc: false
---

# Hydro Analytics

<!-- Read Data from csv file -->
```js
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
```

<!-- Country selector -->
```js
const countrySelector = Inputs.select(
  [...new Set(waterData.map(d => d.Area))].sort(),
  {
    label: "Select Country",
    value: [...new Set(waterData.map(d => d.Area))][0]
  }
);
```

<!-- Difine Colors -->
```js
const colorData = Plot.scale({
  color: {
    type: "categorical",
    domain: d3.groupSort(waterData, (D) => -D.length, (d) => d.Area).filter((d) => d !== "Other"),
    unknown: "var(--theme-foreground-muted)"
  }
});
```



```js
// Line chart for selected country
function waterLineChart(data, country, {width} = {}) {
  const filteredData = data.filter(d => d.Area === country);
  
  return Plot.plot({
    title: `Water Withdrawal by 10^9 m3/year for ${country}`,
    width,
    height: 500,
    y: {
      grid: true,
      label: "Total Value (10^9 m3/year)"
    },
    x: {
      label: "Year"
    },
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
        tip: {
          format: {
            y: ".1f"
          }
        }
      }),
      Plot.ruleY([0])
    ],
    color: {
      legend: true,
      domain: [...new Set(filteredData.map(d => d.Variable))],
      scheme: "tableau10"
    }
  });
}
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>Selected Country</h2>
    <span class="big">${countrySelector}</span>
  </div>
</div>

<div class="grid grid-cols-1">
  <div class="card">
    ${resize((width) => waterLineChart(waterData, countrySelector.value, {width}))}
  </div>
</div>
