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
    IsAggregate: (d) => d === "False",
  },
});
const waterData = await launches2;
```

<!-- Country selector -->

```js
const countrySelector = view(
  Inputs.select([...new Set(waterData.map((d) => d.Area))].sort(), {
    label: "Select Country",
    value: [...new Set(waterData.map((d) => d.Area))][0],
  })
);
const subgroupeSelector = view(
  Inputs.select([...new Set(waterData.map((d) => d.Subgroup))].sort(), {
    label: "Select SubGroup",
    value: "Water withdrawal by sector", // Set a default value
  })
);
const unitSelector = view(
  Inputs.select([...new Set(waterData.map((d) => d.Unit))].sort(), {
    label: "Select Unit",
    value: "10^9 m3/year", // Set a default value
  })
);

const excludeEnvFlow = view(
  Inputs.select(["True", "False"], {
    label: "Exclude Environmental Flow Requirements",
    value: "True", // Set a default value
  })
);
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

<!-- waterLineChart Graphic -->

```js
const selectedCountryData = {
  async value() {
    return waterData.filter((d) => d.Area === countrySelector);
  },
};
function waterLineChart(
  data,
  selectedCountry,
  selectedSubgroup,
  selectedUnit,
  excludeEnv,
  { width } = {}
) {
  const filteredData = data.filter(
    (d) =>
      d.Area === selectedCountry &&
      (selectedSubgroup === "All" || d.Subgroup === selectedSubgroup) &&
      (selectedUnit === "All" || d.Unit === selectedUnit) &&
      (excludeEnv != "True" || d.Variable !== "Environmental Flow Requirements")
  );

  console.log(excludeEnv);

  const maxValue = Math.max(...filteredData.map((d) => d.Value));
  const yMax = Math.ceil(maxValue * 1.1);
  const years = filteredData.map((d) => d.Year);
  const xMin = Math.min(...years);
  const xMax = Math.max(...years);

  return Plot.plot({
    title: `Water Withdrawal by 10^9 m3/year for ${selectedCountry}`,
    width,
    height: 500,
    y: {
      grid: true,
      label: "Total Value (10^9 m3/year)",
      domain: [0, yMax],
      nice: true,
    },
    x: {
      label: "Year",
      domain: [xMin, xMax],
      nice: true,
    },
    marks: [
      Plot.line(filteredData, {
        x: "Year",
        y: "Value",
        stroke: "Variable",
        strokeWidth: 2,
      }),
      Plot.dot(filteredData, {
        x: "Year",
        y: "Value",
        fill: "Variable",
        stroke: "white",
        strokeWidth: 1,
        tip: true,
      }),
      Plot.ruleY([0]),
    ],
    color: {
      legend: true,
      scheme: "tableau10",
    },
  });
}
```

<!-- Display layout -->
<div class="grid grid-cols-2">
  <div class="card">
    <h2>Selected Country</h2>
    <span class="big">${countrySelector}</span>
  </div>
  <div class="card">
    <h2>Selected Subgroup</h2>
    <span class="big">${subgroupeSelector}</span>
  </div>
  <div class="card">
    <h2>Selected Unit</h2>
    <span class="big">${unitSelector}</span>
  </div>
  <div class="card">
    <h2>Selected Unit</h2>
    <span class="big">${excludeEnvFlow}</span>
  </div>
</div>

<div class="grid grid-cols-1">
  <div class="card">
    ${resize((width) => waterLineChart(waterData, countrySelector, subgroupeSelector,unitSelector,excludeEnvFlow, {width}))}
      </div>
</div>
