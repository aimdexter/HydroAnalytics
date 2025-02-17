---
theme: dashboard
title: Bar Chart
toc: false
---

# Bar Chart

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
    IsAggregate: (d) => d === "False",
  },
});
const waterData = await launches2;
```

<!-- Difine Colors -->

```js
const colorData = Plot.scale({
  color: {
    type: "categorical",
    domain: d3
      .groupSort(
        waterData,
        (D) => -D.length,
        (d) => d.Area
      )
      .filter((d) => d !== "Other"),
    unknown: "var(--theme-foreground-muted)",
  },
});
```

```js
function top10BarChart(data, selectedVariable, { width } = {}) {
  const maxYear = Math.max(...data.map((d) => d.Year));
  const filteredData = data.filter(
    (d) => d.Variable === selectedVariable && d.Year === maxYear
  );

  const top10Data = filteredData.sort((a, b) => b.Value - a.Value).slice(0, 10);

  return Plot.plot({
    title: `Top 5 Regions for ${selectedVariable}`,
    width,
    height: 500,
    marginLeft: 120,
    x: {
      label: "Region",
      domain: top10Data.map((d) => d.Area),
    },
    y: {
      grid: true,
      label: `Total Value (10^9 m3/year)`,
    },
    marks: [
      Plot.barY(top10Data, {
        x: "Area",
        y: "Value",
        fill: "Area",
        sort: { x: "-y" },
      }),
      Plot.ruleY([0]),
    ],
    color: {
      scheme: "tableau10",
    },
  });
}
```

```js
const variableSelector = view(
  Inputs.select([...new Set(waterData.map((d) => d.Variable))].sort(), {
    label: "Select Variable for Top 5",
    value: "Agricultural water withdrawal",
    width: "100%",
  })
);
```

<div class="grid grid-cols-1">
  <div class="card">
    ${resize((width) => top10BarChart(waterData, variableSelector, {width}))}
  </div>
</div>
