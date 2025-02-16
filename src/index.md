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
const countrySelector = view(Inputs.select(
  [...new Set(waterData.map(d => d.Area))].sort(),
  {
    label: "Select Country",
    value: [...new Set(waterData.map(d => d.Area))][0]
  }
));
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


<!-- waterLineChart Graphic -->
```js
const currentChart = derived(countrySelector, (country) => ({
  render: (width) => waterLineChart(waterData, country, {width})
}));

function waterLineChart(data, selectedCountry, {width} = {}) {
  return Plot.plot({
    title: `Water Withdrawal by 10^9 m3/year for ${selectedCountry}`,
    width,
    height: 500,
    y: {
      grid: true,
      label: "Total Value (10^9 m3/year)",
      domain: [0, 100]
    },
    x: {
      label: "Year"
    },
    marks: [
      Plot.line(data.filter(d => d.Area === selectedCountry), {
        x: "Year",
        y: "Value",
        stroke: "Variable",
        strokeWidth: 2
      }),
      Plot.dot(data.filter(d => d.Area === selectedCountry), {
        x: "Year",
        y: "Value",
        fill: "Variable",
        stroke: "white",
        strokeWidth: 1,
        tip: true
      }),
      Plot.ruleY([0])
    ],
    color: {
      legend: true,
      scheme: "tableau10"
    }
  });
}
```
// Display layout
<div class="grid grid-cols-1">
  <div class="card">
    <h2>Selected Country</h2>
    <span class="big">${countrySelector}</span>
  </div>
</div>

<div class="grid grid-cols-1">
  <div class="card">
    ${resize((width) => currentChart.value.render(width))}
  </div>
</div>