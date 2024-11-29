---
theme: alt
toc: true
---

# Radial climate plot in Observable Plot

I saw this sort of graph on [Dr. Dominic Royé's blog](https://dominicroye.github.io/en/2021/climate-circles/), and wanted to recreate it in [Observable Plot](https://observablehq.com/plot/).

So let's make a graph of the climate in a city. Here's what we'll end up with:

```js
display(
  Plot.plot({
    title: "Portland, Maine climate",
    subtitle: "average degrees F, 1940-2024",
    projection: {
      type: "azimuthal-equidistant",
      rotate: [0, -90],
      domain: d3.geoCircle().center([0, 90]).radius(0.625)(),
    },
    color: { scheme: "Plasma" },
    marks: [
      // grey circles at 0°, 30°, 60°, 90°
      Plot.geo([0, 30, 60, 90], {
        geometry: (d) =>
          d3
            .geoCircle()
            .center([0, 90])
            .radius(lat(d) - 90)(),
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
      // axis labels
      Plot.text(
        // prettier-ignore
        ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"],
        {
          x: (d) => lon(new Date(`${d} 1, 2024`)),
          y: lat(110),
        },
      ),
      // the lines representing each day
      Plot.geo(rules, { stroke: "temperature" }),
    ],
  }),
);
```

## Get Climate Data

Getting climate data for a US city from the NOAA isn't super complicated, but
it does have a lot of steps.

If you want to skip this rigamarole and just [download the data I used](https://github.com/llimllib/climatecycles/raw/refs/heads/main/docs/data/portland-1940-2023-noaa-data.csv), go ahead.

If you want to get data for your own city:

- Go to [NOAA's Daily Observational Data](https://www.ncei.noaa.gov/maps/daily/) site
- Click on the wrench next to `GHCN Daily` to open up the "map tools" pane
- Click the "identify" button in the "map tools" pane
- Click on a dot to select a weather station
- Click the `View Station Details` button in the left pane
- Click on `Add to cart`
- Click on the orange shopping cart button
- Select a CSV output
- Choose the maximum date extent available for your chosen station
- Click "continue"
- Select all data types for custom output
- Click "continue"
- Enter your email address twice
- Finally, click "submit order"
- In your email, you should get a confirmation of order email followed shortly by an email with a download link. For me, it only took a minute or two for the download link to arrive.

I'd love to find a way to get this from an API instead, because this is painful, so if you know how please let me know.

## Save the data

Once you get an email with your data link, or you download the [data I provided](https://github.com/llimllib/climatecycles/raw/refs/heads/main/docs/data/portland-1940-2023-noaa-data.csv), save it somewhere that you can access it from Plot.

I'm using observable framework to write this, so I saved it to the `docs/data` directory and opened it as a `FileAttachment`:

```js echo
const data = await FileAttachment("data/portland-1940-2023-noaa-data.csv").csv({
  typed: true,
});
display(data);
```

Now that we have the data, I like to do something simple to graph it and make sure it makes sense. Here's the daily low from 1940 to 2024; it's not a useful graph but it shows that we seem to have data that oscillates with the seasons.

```js echo display
display(Plot.plot({ marks: [Plot.line(data, { x: "DATE", y: "TMIN" })] }));
```

## Group the data

In order to graph the data the way we want, we'll need to group the data by date. After a bunch of hacking around, I ended up with this `d3.rollup` command.

- group the data by month and day using `toLocaleString`
- for each group, get the date and the average high, low, and average temperature

```js echo
const dataByDay = [
  ...d3
    .rollup(
      data,
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
display(dataByDay);
```

## Graph the grouped data

Let's do another graph to make sure we're on the right track. Looks good!

```js echo
display(
  Plot.plot({
    title: "Portland climate",
    subtitle: "average degrees F, 1940-2024",
    // https://observablehq.com/@d3/color-schemes
    color: { scheme: "Plasma" },
    marks: [
      Plot.ruleX(dataByDay, {
        x: "date",
        y1: "low",
        y2: "high",
        stroke: "mean",
      }),
    ],
  }),
);
```

## Make a polar graph

Observable plot [doesn't have polar plots](https://github.com/observablehq/plot/issues/133), so we're going to have to be a bit creative to get the output we want.

After looking at [a few](https://github.com/observablehq/plot/issues/133) [examples](https://github.com/observablehq/plot/issues/133) of somewhat similar plots on the web, I figured out that people had been using plot's geographical features to create radial plots.

Essentially, we're going to create a map [like this view of Antarctica](https://observablehq.com/plot/features/projections#:~:text=below%2C%20a%20map%20of%20antarctica%20in%20a%20polar%20aspect%20of%20the%20azimuthal-equidistant%20projection) in the docs, except that instead of drawing Antarctica on it, we're going to draw our climate data.

First, we need to create some scales:

```js echo
// map dates to a longitude
const timeExtent = [new Date(2024, 1, 1), new Date(2024, 12, 31)];
const lon = d3.scaleTime(timeExtent, [-180, 180]);
// map degrees, from -20 to 100, to an output in the range of 90 - 90.5
const lat = d3.scaleLinear([-20, 100], [90, 90.5]);
```

Then, we're going to create a geoJSON point for each day of the year, starting at `[date, low temperature]`, and going to `[date, high temperature]`:

```js echo
// for each day, create a line starting at
// [date, low] and going to [date, high]
const rules = dataByDay.map((d) => ({
  type: "LineString",
  coordinates: [
    [lon(d.date), lat(d.low)],
    [lon(d.date), lat(d.high)],
  ],
  temperature: d.mean,
}));
```

Finally, we can create our graph:

```js echo
display(
  Plot.plot({
    title: "Portland, Maine climate",
    subtitle: "average degrees F, 1940-2024",
    projection: {
      type: "azimuthal-equidistant",
      rotate: [0, -90],
      domain: d3.geoCircle().center([0, 90]).radius(0.625)(),
    },
    color: { scheme: "Plasma" },
    marks: [
      // grey circles at 0°, 30°, 60°, 90°
      Plot.geo([0, 30, 60, 90], {
        geometry: (d) =>
          d3
            .geoCircle()
            .center([0, 90])
            .radius(lat(d) - 90)(),
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
      // axis labels
      Plot.text(
        // prettier-ignore
        ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"],
        {
          x: (d) => lon(new Date(`${d} 1, 2024`)),
          y: lat(110),
        },
      ),
      // the lines representing each day
      Plot.geo(rules, { stroke: "temperature" }),
    ],
  }),
);
```
