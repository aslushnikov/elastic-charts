<h1 align="center">
  Elastic Charts
</h1>
<p align="center">
  <a href="(https://travis-ci.org/elastic/elastic-charts"><img alt="Build Status" src="https://travis-ci.org/elastic/elastic-charts.svg?branch=master"></a>
  <a href="http://commitizen.github.io/cz-cli/"><img alt="Commitizen friendly" src="https://img.shields.io/badge/commitizen-friendly-brightgreen.svg"></a>
</p>

🚨 **WARNING** While open source, the intended consumers of this repository are Elastic products. Read the [FAQ][faq] for details.

---

This library is a complete rewrite of the current vislib in Kibana and EUISeriesChart in EUI.
The rationale behind this refactoring is the need for a testable and decoupled architecture for displaying data within charts. The current EUI implementation is based on ReactVis, which directly manipulates the data series inside components without a clear rendering pipeline and without a clean way to extend it. Some of the downsides of using ReactVis are:

- the main chart component, before rendering children, looks at their props and recomputes them, injecting new props. Some configuration is accessed through references to children components.
- the components are themselves SVG components that render bars, lines, and axes. The problem with this is that not all components can be rendered at the same time, but there is the need for a rendering pipeline to allow a correct placement for all the geometries, especially when we face the need for having auto-scaled axis dimensions.
- the only way to test the chart is testing the resulting SVG component. If rendered through canvas the test can be only a visual regression test.
- no possibility of manage x-indexing of elements

This new implementation revisits the concept of a charting library and tries to apply a unidirectional rendering flow to the concept of charting. The rendering flow is the following:

![rendering-pipeline](https://user-images.githubusercontent.com/1421091/49724064-bba8cb80-fc68-11e8-8378-9d59b941f15d.png)

This controlled flow allows us to achieve the following points:

- the computation of series dimensions (x and y domains of the datasets, for example) are required to precompute the space required for the axis labeling. Axis rendering is dependent on the axis displayed values, and thus on the data series provided. This is a required passage to accommodate automatically spaced axes . Other libraries just live the developer with the needs to specify static margin space to render the axis labels, and this can result in truncated labels.
- put a testing probe just before rendering the chart, the computed geometries are the exact values that need to be used to render on SVG, canvas or WebGL on that exact portion of the screen. No further calculation is needed on the rendered component. x, y, width, height, color, and transforms are computed before the rendering phase.
- reduce the rendering operation to the minimum required. Resizing, for example, will only require the last 3 phases to complete.
- decouple the chart from its rendering medium: can be SVG, canvas or WebGL, using React or any other DOM library.
- part of this computation can also be processed server side or on a WebWorker.

The rendering pipeline is achieved by revisiting the way a chart library is built. Instead of creating a chart library around a set of rendering components: bar series, axis etc., this new implementation decouples the specification of the chart from the rendering components. The rendering components are part of the internals of the library. We are exposing `empty` react components to the developer, using the JSX format just as a declarative language to describe the specification of your chart and not as a set of real react components that will render something.
That is achieved using the following render function on the main `Chart` component:

```jsx
<Provider chartStore={this.chartSpecStore}>
  <Fragment>
    <SpecsParser>{this.props.children}</SpecsParser>
    <ChartResizer />
    {renderer === 'svg' && <SVGChart />}
    {renderer === 'canvas' && <CanvasChart />}
    {renderer === 'canvas_native' && <NativeChart />}
    <Tooltips />
  </Fragment>
</Provider>
```

Where all the children passed are rendered inside the `SpecsParser`, that signal a state manager that we are updating the specification of the chart, but doesn't render anything.
The actual rendering is done by one of the rendered like the `ReactChart` that is rendered after the rendering pipeline produced and saved the geometries on the state manager.

A spec can be something like the following:

```js
<Chart renderer={renderer}>
  <Settings rotation={rotation} animateData={true} />
  <Axis id={getAxisId('bottom')} position={AxisPosition.Bottom} title={`Rendering test`} />
  <Axis id={getAxisId('left')} position={AxisPosition.Left} />
  <LineSeries
    id={getSpecId('1')}
    yScaleType={ScaleType.Linear}
    xScaleType={ScaleType.Linear}
    xAccessor="x"
    yAccessors={['y']}
    data={BARCHART_1Y0G}
  />
  <BarSeries
    id={getSpecId('2')}
    yScaleType={ScaleType.Linear}
    xScaleType={ScaleType.Linear}
    xAccessor="x"
    yAccessors={['y1', 'y2']}
    splitSeriesAccessors={['g']}
    stackAccessors={['x', 'g']}
    data={BARCHART_2Y1G}
  />
</Chart>
```

## Setting Up Your Development Environment

Fork, then clone the `elastic-chart` repo and change directory into it

```bash
git clone git@github.com:<YOUR_GITHUB_NAME>/elastic-charts.git elastic-charts
cd kibana
```

Install the latest version of [yarn](https://yarnpkg.com)

We depend upon the version of node defined in [.nvmrc](.nvmrc).

You will probably want to install a node version manager. [nvm](https://github.com/creationix/nvm) is recommended.

To install and use the correct node version with `nvm`:

```bash
nvm install
```

Install all the dependencies

```bash
yarn install
```

### Storybook

We develop using [storybook](https://storybook.js.org) to document API, edge-cases, and usage of the library.
A hosted version is available at [https://elastic.github.io/elastic-charts](https://elastic.github.io/elastic-charts).
You can run locally at [http://localhost:9001/](http://localhost:9001/) by running:

```
yarn storybook
```

## Installation

**note:** there is no published package on NPM at the moment (30/01/2018). Will be part of the Roadmap #1.

To install the Elastic Charts into an existing project, use the `yarn` CLI (`npm` is not supported).

```
yarn add @elastic/charts
```

## Contributing

We are trying to enforce some good practices in this library:

- All commits must follow the [conventional commits](https://www.conventionalcommits.org/en/v1.0.0-beta.2/)
- [semantic-release](https://semantic-release.gitbook.io) is used as an automated release manager suite.
  This will automatically publish on NPM on every push on master, will automatically create the changelog and bump the correct semver depending on the commits. To avoid too many new releases, especially in this initial phase of the project, we are going to work against a `dev` branch and then merge on master periodically.
- Every commit count in the version bump: this means that we can merge a PR with two methods:
  - merge all the PR commit history (please follow the commit convention or squash partial commits)
  - squash and merge all commits using a single commit that follows the conventions.

The following tools are used to ensure this convention:

- `commitlint` hook ensure that you are following correctly the convention
- `yarn cz` can be used to start `commitizen` as a cli tool that prompts you with selections and questions
  to help you in writing a conventional commit
- `commitlint-bot` is a github app that checks PR commits to help you on writing the correct commit message (not currently enabled on github)

## Concepts

### Axes

The concept of an axis in this library follow the following constraints:

- there is no distinction between x-axis or y-axis.
- the distinction is between the position of the axis on the chart (top, bottom, left, or right) in relation to the rotation of the chart: A standard horizontal bar chart with a Y axis (for a dependent variable whose values rise to the top) can be supported by left and right axis that will show the Y values, and the bottom and top axes will show the X axis for an independent variable. In contrast, a 90-degree (clockwise-rotated) bar chart, with Y values that spread from left to right, will have a horizontal (bottom/top) axis that shows the Y independent variable and the left/right vertical axis that shows the X variable.

As a constraint, we allow only one X-axis, but we provide the ability to add multiple Y-axes (also if it's a discouraged practice (see https://blog.datawrapper.de/dualaxis/ or http://www.storytellingwithdata.com/blog/2016/2/1/be-gone-dual-y-axis)).

### Dataset Domains:

Related to a dataset is the extent of a variable. It usually used to draw the position of the specific value/datum along one axis (vertical or horizontal).
On a series chart, we always need to have at least two domains, representing the 2 dimensions of the data we are drawing.

### Data

It's an array of values, that will be used to compute the chart. Usually, each element must have at least 2 values to be charted. Multiple values can be used to specify how we want to split the chart by series and by y values.

Examples of datasets:

```ts
// 1 metric (y) and no groups/split series ===> 1 single series
const BARCHART_1Y0G = [{ x: 0, y: 1 }, { x: 1, y: 2 }, { x: 2, y: 10 }, { x: 3, y: 6 }];

// 2 metrics (2y) and 1 group/split series ===> 4 different data series
const BARCHART_2Y1G = [
  { x: 0, g: 'a', y1: 1, y2: 4 },
  { x: 0, g: 'b', y1: 3, y2: 6 },
  { x: 1, g: 'a', y1: 2, y2: 1 },
  { x: 1, g: 'b', y1: 2, y2: 5 },
  { x: 2, g: 'a', y1: 10, y2: 5 },
  { x: 2, g: 'b', y1: 3, y2: 1 },
  { x: 3, g: 'a', y1: 7, y2: 3 },
  { x: 3, g: 'b', y1: 6, y2: 4 },
];
```

These datasets can be used as input for any type of chart specification. These are the interfaces that make up a `BasicSpec` (some sort of abstract specification)

```ts
export interface SeriesSpec {
  /* The ID of the spec, generated via getSpecId method */
  id: SpecId;
  /* The ID of the spec, generated via getGroupId method, default to __global__ */
  groupId: GroupId;
  /* An array of data */
  data: Datum[];
  /* If specified, it constrant the x domain to these values */
  xDomain?: Domain;
  /* If specified, it constrant the y Domain to these values */
  yDomain?: Domain;
  /* The type of series you are looking to render */
  seriesType: 'bar' | 'line' | 'area' | 'basic';
}

export interface SeriesAccessors {
  /* The field name of the x value on Datum object */
  xAccessor: Accessor;
  /* An array of field names one per y metric value */
  yAccessors: Accessor[];
  /* An array of fields thats indicates the datum series membership */
  splitSeriesAccessors?: Accessor[];
  /* An array of fields thats indicates the stack membership */
  stackAccessors?: Accessor[];
  /* An optional array of field name thats indicates the stack membership */
  colorAccessors?: Accessor[];
}

export interface SeriesScales {
  /* The x axis scale type */
  xScaleType: ScaleType;
  /* The y axis scale type */
  yScaleType: ScaleContinuousType;
  /** if true, the min y value is set to the minimum domain value, 0 otherwise */
  yScaleToDataExtent: boolean;
}
```

A `BarSeriesSpec` for example is the following union type:

```ts
export type BarSeriesSpec = SeriesSpec &
  SeriesAccessors &
  SeriesScales & {
    seriesType: 'bar';
  };
```

A chart can be feed with data in the following ways:

- one series type specification with one `data` props configured.
- a set of series types with `data` props configured. In these case the data arrays are merged together as the following rules:
  - x values are merged together. If the chart has multiple different `xScaleType`s, the main x scale type is coerced to `ScaleType.Linear` if all the scales are continuous or to `ScaleType.Ordinal` if one scale type is ordinal. Also, a temporal scale is, in specific cases, coerced to linear, so be careful to configure correctly the scales.
  - if there is a specified x domain on the spec, this is used as x domain for that series, and it's merged together with the existing x domains.
  - specifications with `splitAccessors` are split into different series. Each specification is treated in a separated manner: if you have one chart with 3 series merged to one chart with 1 series, this results in a chart that has each x value split in two (the number of specification used, two charts) than on split is used to accommodate 3 series and the other to accommodate the remaining series. If you want to treat each series in the same way, split your chart before and create 4 different BarSeries specs, so that these are rendered evenly on the x-axis.
  - bar, area, line series with a `stackAccessor` are stacked together each stacked above their respective group (areas with areas, bars with bars, lines with lines. You cannot mix stacking bars above lines above areas).
  - bar series without `stackAccessors` are clustered together for the same x value
  - area and line series, without `stackAccessors` are just drawn each one on their own layer (not stacked nor clustered).
  - the y value is influenced by the following aspects:
    - if there is a specified y domain on the spec, this is used as y domain for that series
    - if no or only one y-axis is specified, each y value is treated as part of the same domain.
    - if there is more than one y-axis (visible or not), the y domains are merged in respect of the same `groupId`. For e.g. two bar charts, and two y-axes, each for a spec, one per bar value. The rendered bar heights are independent of each other, because of the two axes.
    - if the data are stacked or not. Stacked produce a rendering where the lower bottom of the chart is the previous series y value.

On the current `Visualize Editor`, you can stack or cluster in the following cases:

- when adding multiple Y values: each Y value can be stacked (every type) or clustered (only bars)
- when splitting a series, each series can be stacked (every type) or clustered (only bars)

### Multiple charts/Split charts/Small Multiples (phase 2)

Small multiples are created using the `<SmallMultiples>` component, that takes multiple `<SplittedSeries>` component with the usual element inside. `<SplittedSeries>` can contain only `BarSeries` `AreaSeries` and `LineSeries` Axis and other configuration need to be configured outside the `SplittedSeries` element.

In the case of small multiples, each `SplittedSeries` computes its own x and y domains. Then the x domains are merged and expanded. The same happens with the main Y domains; they are merged together.

### Colors

Each data series can have its own color.
The color is assigned through the `colorAccessors` prop that specifies which data attributes are used to define the color,
for example:

- a dataset without any split accessor or fields that can be used to identify a color will have a single color.
- a dataset that has 1 variable to split the dataset into 3 different series, will have 3 different colors if that variable is specified through the `colorAccessors`.
- a dataset with 2 split variables, each with 3 different values (a dataset with 3 \* 2 series) will have 6 different colors if the two variables are defined in `colorAccessors`
- a dataset with 2 split variables, each with 3 different values, but only one specified in the `colorAccessors` will have only 3 colors.
- if no `colorAccessors` is specified, `splitAccessors` will be used to identify how to coloring the series


## License

[Apache Licensed.][license] Read the [FAQ][faq] for details.

[license]: LICENSE.md
[faq]: FAQ.md
