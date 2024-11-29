---
theme: alt
toc: false
---

```js
function convertNum(obj) {
  const result = {};

  for (const key in obj) {
    const value = obj[key];
    // Number("") == 0, which is insane
    if (value == "") {
      result[key] = null;
    } else {
      const n = Number(value);
      result[key] = isNaN(n) ? value : n;
    }
  }

  return result;
}

const data = (
  await FileAttachment("data/portland-2023-noaa-data.csv").csv()
).map(convertNum);
data.forEach((d) => (d.DATE = new Date(d.DATE)));
const alldata = (
  await FileAttachment("data/portland-1940-2023-noaa-data.csv").csv()
).map(convertNum);
alldata.forEach((d) => (d.DATE = new Date(d.DATE)));
```

```js
const dataByDay = [
  ...d3
    .rollup(
      alldata,
      // a date without a year isn't parsed by `new Date`, so I had to add
      // 2024. Just remove the year when formatting output
      (v) => ({
        date: new Date(
          d3.min(v, (d) =>
            d.DATE.toLocaleString("en-US", { month: "short", day: "numeric" }),
          ) + " 2024",
        ),
        mean: d3.mean(v, (d) => d.TAVG),
        high: d3.mean(v, (d) => d.TMAX),
        low: d3.mean(v, (d) => d.TMIN),
      }),
      (d) => d.DATE.toLocaleString("en-US", { month: "short", day: "numeric" }),
    )
    .values(),
];
```

```js
const graph = Plot.plot({
  title: "Portland climate",
  subtitle: "average degrees F, 1940-2024",
  // https://observablehq.com/@d3/color-schemes
  color: { scheme: "Plasma" },
  marks: [
    // Plot.ruleX(data, { x: "DATE", y1: "TMIN", y2: "TMAX", stroke: "TAVG" }),
    Plot.ruleX(dataByDay, { x: "date", y1: "low", y2: "high", stroke: "mean" }),
  ],
});
display(graph);
```

```js
// map the dates to a longitude
const timeExtent = [new Date(2024, 1, 1), new Date(2024, 12, 31)];
const monthsSince = (d) => d3.utcMonth.count(timeExtent[0], d);
const lon = d3.scaleTime(timeExtent, [-180, 180]);
// lat: domain is temperature, output is 90+[0,.5]
const lat = d3.scaleLinear([-20, 100], [90, 90.5]);
const rules = dataByDay.map((d) => ({
  type: "LineString",
  coordinates: [
    [lon(d.date), lat(d.low)],
    [lon(d.date), lat(d.high)],
  ],
  temperature: d.mean,
}));

// working off:
// https://observablehq.com/@observablehq/plot-radar-chart
// https://observablehq.observablehq.cloud/pangea/plot/spiral-heatmap
const graph = Plot.plot({
  title: "Portland, Maine climate",
  subtitle: "average degrees F, 1940-2024",
  width: 450,
  projection: {
    type: "azimuthal-equidistant",
    rotate: [0, -90],
    // Note: 0.625° corresponds to max. length (here, 0.5), plus enough room for the labels
    // Q: why did they set the center of the circle to [0,90] instead of [0,0]?
    // A: if you set it to [0,0], you get squished ovals instead of circles
    // subQ: why?
    domain: d3.geoCircle().center([0, 90]).radius(0.625)(),
  },
  color: { scheme: "Plasma" },
  marks: [
    // grey discs
    Plot.geo([0.5, 0.4, 0.3, 0.2, 0.1], {
      geometry: (r) => d3.geoCircle().center([0, 90]).radius(r)(),
      stroke: "black",
      fill: "white",
      strokeOpacity: 0.3,
      fillOpacity: 0.03,
      strokeWidth: 0.5,
    }),
    // white axes
    Plot.link(
      // prettier-ignore
      ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"],
      {
        x1: (d) => lon(new Date(`${d} 1, 2024`)),
        y1: 90,
        x2: (d) => lon(new Date(`${d} 1, 2024`)),
        y2: 90.5,
        stroke: "white",
        strokeOpacity: 0.2,
        strokeWidth: 1,
      },
    ),
    Plot.geo(rules, { stroke: "temperature" }),
    Plot.text(
      // prettier-ignore
      ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"],
      {
        x: (d) => lon(new Date(`${d} 1, 2024`)),
        y: lat(110),
      },
    ),
  ],
});
display(graph);
```
