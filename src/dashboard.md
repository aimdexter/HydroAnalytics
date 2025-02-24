---
theme: dashboard
title: Dashboard
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

```js
const countrySelector = view(
  Inputs.select([...new Set(waterData.map((d) => d.Area))].sort(), {
    label: "Select Country",
    value: [...new Set(waterData.map((d) => d.Area))][0],
    width: "100%",
  })
);
const subgroupeSelector = view(
  Inputs.select([...new Set(waterData.map((d) => d.Subgroup))].sort(), {
    label: "Select SubGroup",
    value: "Water withdrawal by sector",
    width: "100%",
  })
);
const unitSelector = view(
  Inputs.select([...new Set(waterData.map((d) => d.Unit))].sort(), {
    label: "Select Unit",
    value: "10^9 m3/year",
    width: "100%",
  })
);

const excludeEnvFlow = view(
  Inputs.radio([true, false], {
    label: "Include Environmental Flow Requirements",
    value: true,
    width: "100%",
  })
);

const years = Array.from(new Set(waterData.map((d) => d.Year))).sort();
const selectedYear = view(
  Inputs.range([Math.min(...years), Math.max(...years)], {
    label: "Select a Year",
    step: 1,
    value: Math.min(...years),
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
  selectedCountry,
  selectedSubgroup,
  selectedUnit,
  excludeEFR
) {
  // Définition des dimensions
  const width = 800,
    height = 500,
    margin = { top: 40, right: 200, bottom: 70, left: 100 };

  // === 1. Filtrer les données ===
  let filteredData = waterData.filter((d) => d.Area === selectedCountry);

  // Filtrer par unité
  if (selectedUnit === "%") {
    filteredData = filteredData.filter((d) => d.Variable.includes("%"));
  } else if (selectedUnit === "m3/inhab/year") {
    filteredData = filteredData.filter((d) =>
      d.Variable.includes("per capita")
    );
  } else {
    filteredData = filteredData.filter(
      (d) => !d.Variable.includes("%") && !d.Variable.includes("per capita")
    );
  }

  // Exclure "Environmental Flow Requirements" si demandé
  if (excludeEFR) {
    filteredData = filteredData.filter(
      (d) => d.Variable !== "Environmental Flow Requirements"
    );
  }

  // Filtrer par sous-groupe
  filteredData = filteredData.filter((d) => d.Subgroup === selectedSubgroup);

  // === 2. Regrouper les données par année ===
  const groupedData = d3
    .groups(filteredData, (d) => d.Year)
    .map(([year, values]) => ({
      year: +year,
      values: values.map((d) => ({ variable: d.Variable, value: +d.Value })),
    }));

  // Sélection des années et variables
  const years = Array.from(new Set(filteredData.map((d) => d.Year))).sort(
    d3.ascending
  );
  const variables = Array.from(new Set(filteredData.map((d) => d.Variable)));

  // === 3. Définir les échelles des axes ===
  const x = d3
    .scaleLinear()
    .domain(d3.extent(groupedData, (d) => d.year))
    .range([margin.left, width - margin.right]);

  const y = d3
    .scaleLinear()
    .domain([
      0,
      d3.max(groupedData, (d) => d3.max(d.values, (v) => v.value)) || 1,
    ])
    .nice()
    .range([height - margin.bottom, margin.top]);

  const color = d3.scaleOrdinal(d3.schemeCategory10).domain(variables);

  // === 4. Créer le SVG ===
  const svg = d3.create("svg").attr("width", width).attr("height", height);

  // === 5. Ajouter les axes ===
  svg
    .append("g")
    .attr("transform", `translate(0, ${height - margin.bottom})`)
    .call(d3.axisBottom(x).tickFormat(d3.format("d")).ticks(years.length))
    .selectAll("text")
    .attr("transform", "rotate(45)")
    .style("text-anchor", "start");

  svg
    .append("text")
    .attr("x", width / 2)
    .attr("y", height - 20)
    .attr("text-anchor", "middle")
    .style("font-size", "14px")
    .text("Year")
    .style("fill", "white");

  svg
    .append("g")
    .attr("transform", `translate(${margin.left}, 0)`)
    .call(d3.axisLeft(y));

  svg
    .append("text")
    .attr("transform", "rotate(-90)")
    .attr("x", -height / 2)
    .attr("y", margin.left - 60)
    .attr("text-anchor", "middle")
    .style("font-size", "14px")
    .text(`Total Value (${selectedUnit})`)
    .style("fill", "white");

  // === 6. Ajouter la légende ===
  const legend = svg
    .append("g")
    .attr(
      "transform",
      `translate(${width - margin.right + 20}, ${margin.top})`
    );

  variables.forEach((variable, i) => {
    const row = legend.append("g").attr("transform", `translate(0, ${i * 20})`);
    row
      .append("rect")
      .attr("width", 15)
      .attr("height", 15)
      .attr("fill", color(variable));
    row
      .append("text")
      .attr("x", 20)
      .attr("y", 10)
      .text(variable)
      .style("font-size", "12px")
      .style("fill", "white");
  });

  // === 7. Ajouter un tooltip ===
  const tooltip = d3
    .select("body")
    .append("div")
    .style("position", "absolute")
    .style("background", "#fff")
    .style("color", "black")
    .style("padding", "5px")
    .style("border", "1px solid #ccc")
    .style("border-radius", "4px")
    .style("box-shadow", "0px 0px 5px rgba(0,0,0,0.3)")
    .style("pointer-events", "none")
    .style("opacity", 0);

  // === 8. Tracer les lignes ===
  const line = d3
    .line()
    .x((d) => x(d.year))
    .y((d) => y(d.value));

  variables.forEach((variable) => {
    const dataPoints = groupedData
      .map((d) => ({
        year: d.year,
        value: (
          d.values.find((v) => v.variable === variable) || { value: null }
        ).value,
      }))
      .filter((d) => d.value !== null);

    if (dataPoints.length > 0) {
      svg
        .append("path")
        .datum(dataPoints)
        .attr("fill", "none")
        .attr("stroke", color(variable))
        .attr("stroke-width", 2)
        .attr("d", line);

      // === 9. Ajouter des points interactifs ===
      svg
        .selectAll(`.dot-${variable.replace(/[^a-zA-Z0-9]/g, "-")}`)
        .data(dataPoints)
        .enter()
        .append("circle")
        .attr("class", "dot")
        .attr("cx", (d) => x(d.year))
        .attr("cy", (d) => y(d.value))
        .attr("r", 5)
        .attr("fill", color(variable))
        .style("cursor", "pointer")
        .on("mouseover", (event, d) => {
          tooltip
            .style("opacity", 1)
            .html(
              `Country: <b>${selectedCountry}</b><br>
                   Year: <b>${d.year}</b><br>
                   Value: <b>${d3.format(".2f")(d.value)} ${selectedUnit}</b>`
            )
            .style("left", `${event.pageX + 10}px`)
            .style("top", `${event.pageY - 20}px`);
        })
        .on("mousemove", (event) => {
          tooltip
            .style("left", `${event.pageX + 10}px`)
            .style("top", `${event.pageY - 20}px`);
        })
        .on("mouseout", () => {
          tooltip.style("opacity", 0);
        });
    }
  });

  return svg.node();
}
```

<!-- top10BarChart -->

```js
function top10BarChart(selectedVariable, selectedYear) {
  // Définition des dimensions
  const width = 800,
    height = 500,
    margin = { top: 50, right: 30, bottom: 100, left: 250 }; // 📌 Augmentation de la marge gauche

  // === 1. Filtrer les données ===
  let filteredData = waterData.filter(
    (d) => d.Variable === selectedVariable && +d.Year === selectedYear
  );

  // Trouver l'unité associée à la variable sélectionnée
  let unit = filteredData.length > 0 ? filteredData[0].Unit : "Unknown Unit";

  // Grouper par région et calculer la somme des valeurs
  let aggregatedData = d3.rollups(
    filteredData,
    (v) => d3.sum(v, (d) => +d.Value),
    (d) => d.Area
  );

  // Trier et garder les 10 premières valeurs
  let topRegionsData = aggregatedData.sort((a, b) => b[1] - a[1]).slice(0, 10);

  // Sélectionner les régions et valeurs
  const regions = topRegionsData.map((d) => d[0]);
  const values = topRegionsData.map((d) => d[1]);

  // === 2. Définir les échelles des axes ===
  const x = d3
    .scaleLinear()
    .domain([0, d3.max(values) || 1])
    .nice()
    .range([margin.left, width - margin.right]);

  const y = d3
    .scaleBand()
    .domain(regions)
    .range([margin.top, height - margin.bottom])
    .padding(0.3);

  // === 3. Créer le SVG ===
  const svg = d3.create("svg").attr("width", width).attr("height", height);

  // === 4. Ajouter les axes ===
  svg
    .append("g")
    .attr("transform", `translate(0, ${height - margin.bottom})`)
    .call(d3.axisBottom(x))
    .selectAll("text")
    .style("text-anchor", "middle");

  // 📌 Mise à jour dynamique de l'unité sur l'axe X
  svg
    .append("text")
    .attr("x", width / 2)
    .attr("y", height - 50)
    .attr("text-anchor", "middle")
    .style("font-size", "14px")
    .text(`Total Value (${unit})`)
    .style("fill", "white");

  svg
    .append("g")
    .attr("transform", `translate(${margin.left}, 0)`) // 📌 Décalage correct du graphe vers la droite
    .call(d3.axisLeft(y));

  // 📌 Position normale du titre de l'axe Y
  svg
    .append("text")
    .attr("transform", "rotate(-90)")
    .attr("x", -height / 2)
    .attr("y", margin.left - 230) // Position standard sans superposition
    .attr("text-anchor", "middle")
    .style("font-size", "14px")
    .text("Country")
    .style("fill", "white");

  // === 5. Ajouter les barres ===
  svg
    .selectAll(".bar")
    .data(topRegionsData)
    .enter()
    .append("rect")
    .attr("class", "bar")
    .attr("x", margin.left)
    .attr("y", (d) => y(d[0]))
    .attr("width", (d) => x(d[1]) - margin.left)
    .attr("height", y.bandwidth())
    .attr("fill", d3.scaleOrdinal(d3.schemeCategory10).domain(topRegionsData));

  // === 6. Ajouter les labels des valeurs sur les barres ===
  svg
    .selectAll(".label")
    .data(topRegionsData)
    .enter()
    .append("text")
    .attr("x", (d) => x(d[1]) + 5)
    .attr("y", (d) => y(d[0]) + y.bandwidth() / 2)
    .attr("dy", "0.35em")
    .text((d) => d3.format(".2f")(d[1]))
    .style("fill", "white");

  return svg.node();
}
```

<!-- variableSelector -->

```js
const variableSelector = view(
  Inputs.select([...new Set(waterData.map((d) => d.Variable))].sort(), {
    label: "Select Variable for Top 5",
    value: "Agricultural water withdrawal",
    width: "100%",
  })
);
```

```js
const worldData = await d3.json(
  "https://raw.githubusercontent.com/datasets/geo-countries/master/data/countries.geojson"
);

function createWorldMap(
  selectedYear,
  selectedSubgroup,
  selectedUnit,
  excludeEFR
) {
  const width = 1200,
    height = 1200;
  const svg = d3
    .create("svg")
    .attr("width", width)
    .attr("height", height)
    .style("border", "1px solid #ccc")
    .style("background-color", "#000"); // Mode sombre

  const projection = d3
    .geoMercator()
    .scale(250)
    .translate([width / 2, height / 1.4]);

  const path = d3.geoPath(projection);
  const g = svg.append("g");

  g.selectAll("path")
    .data(worldData.features)
    .join("path")
    .attr("d", path)
    .attr("fill", "#333")
    .attr("stroke", "#fff")
    .attr("stroke-width", 0.5);

  // === 1. Ajouter un Tooltip avec le nom du pays ===
  const tooltip = d3
    .select("body")
    .append("div")
    .style("position", "absolute")
    .style("background", "rgba(255,255,255,0.9)")
    .style("border", "1px solid #ccc")
    .style("padding", "8px")
    .style("border-radius", "4px")
    .style("pointer-events", "none")
    .style("font-size", "12px")
    .style("display", "none");

  // === 2. Filtrer les données ===
  let filteredData = waterData.filter(
    (d) =>
      +d.Year === selectedYear &&
      d.Subgroup === selectedSubgroup &&
      d.Unit === selectedUnit
  );

  // Exclure "Environmental Flow Requirements" si demandé
  if (!excludeEFR) {
    filteredData = filteredData.filter(
      (d) => d.Variable !== "Environmental Flow Requirements"
    );
  }

  const data = Object.create(null);
  for (const row of filteredData) {
    if (!data[row.Area]) {
      data[row.Area] = [];
    }
    data[row.Area].push({ name: row.Variable, value: +row.Value });
  }

  // === 3. Calculer la surface des pays et ajouter le nom ===
  const countryAreas = worldData.features
    .map((d) => ({
      id: d.id,
      name: d.properties.ADMIN, // Nom du pays
      centroid: projection(d3.geoCentroid(d)),
      area: path.area(d),
    }))
    .filter((d) => d.centroid && data[d.name]);

  // === 4. Échelle de taille proportionnelle à la surface du pays ===
  const minArea = d3.min(countryAreas, (d) => d.area);
  const maxArea = d3.max(countryAreas, (d) => d.area);

  // Taille limitée en fonction de la grandeur du pays
  const sizeScale = d3.scaleSqrt().domain([minArea, maxArea]).range([5, 40]); // Adapter pour éviter l'encombrement

  const pie = d3.pie().value((d) => d.value);
  const colorScale = d3.scaleOrdinal(d3.schemeCategory10);

  // === 5. Créer les pie charts en fonction de la taille du pays ===
  const pieGroups = g
    .selectAll(".pie")
    .data(countryAreas)
    .enter()
    .append("g")
    .attr("class", "pie")
    .attr("transform", (d) => `translate(${d.centroid[0]}, ${d.centroid[1]})`);

  pieGroups.each(function (d) {
    const pieData = data[d.name] || [{ name: "No data", value: 1 }];
    const arc = d3.arc().innerRadius(0).outerRadius(sizeScale(d.area)); // Taille proportionnelle

    d3.select(this)
      .selectAll("path")
      .data(pie(pieData))
      .join("path")
      .attr("d", arc)
      .attr("fill", (d) => colorScale(d.data.name))
      .attr("stroke", "#000")
      .attr("stroke-width", 0.5)
      .on("mouseover", function (event, d) {
        d3.select(this)
          .transition()
          .duration(200)
          .attr("transform", "scale(1.1)");
        tooltip
          .style("display", "block")
          .html(
            `<strong>Country: ${this.parentNode.__data__.name}</strong><br>
                   <strong>${d.data.name}</strong><br>
                   Value: ${d.data.value}`
          )
          .style("left", `${event.pageX + 10}px`)
          .style("top", `${event.pageY + 10}px`);
      })
      .on("mousemove", function (event) {
        tooltip
          .style("left", `${event.pageX + 10}px`)
          .style("top", `${event.pageY + 10}px`);
      })
      .on("mouseout", function () {
        d3.select(this)
          .transition()
          .duration(200)
          .attr("transform", "scale(1)");
        tooltip.style("display", "none");
      });
  });

  // === 6. Zoom dynamique avec limitation du volume des pie charts ===
  const zoomBehavior = d3
    .zoom()
    .scaleExtent([1, 8])
    .on("zoom", (event) => {
      g.attr("transform", event.transform);

      const newScale = event.transform.k;
      pieGroups.each(function (d) {
        let adjustedSize = sizeScale(d.area) / newScale;

        // Appliquer une limite stricte selon la taille du pays
        adjustedSize = Math.max(5, Math.min(adjustedSize, sizeScale(d.area)));

        const arc = d3.arc().innerRadius(0).outerRadius(adjustedSize);
        d3.select(this).selectAll("path").attr("d", arc);
      });
    });

  svg.call(zoomBehavior);

  return svg.node();
}
```

<!-- Display layout -->
<div class="grid grid-cols-4">
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

<div class="grid grid-cols-2">
  <div class="card">
    ${resize((width) => waterLineChart(countrySelector, subgroupeSelector, unitSelector, excludeEnvFlow))}
  </div>
  <div class="card">
    ${resize((width) => top10BarChart(variableSelector, selectedYear))}
  </div>
</div>
<div class="grid grid-cols-1">
  <div class="card">
    ${createWorldMap(selectedYear, subgroupeSelector, unitSelector, excludeEnvFlow)}
  </div>
</div>
