# Tooltip Annotation

## Overview

Tooltip annotations let users click data points and create persistent Plotly annotations from the modebar. The feature supports three customization layers:

- `tooltiptemplate` for formatting the annotation text
- `tooltip` for annotation styling
- `tooltipfunction(ctx)` for runtime customization, remapping, or cancellation

This feature is designed for Plotly.js usage in JavaScript. The callback form is runtime-only and is attached directly to the plotted trace object.

## Requirements And Activation

To use tooltip annotations:

1. Add the tooltip modebar button with `config.modeBarButtonsToAdd: ['tooltip']`
2. Prefer `editable: true` so created annotations can be edited directly on the graph
3. Click the tooltip modebar button in the UI to enable "add on click"
4. Click a point or bin on the plot to create the annotation

Example:

```js
Plotly.newPlot('graph', data, layout, {
  editable: true,
  modeBarButtonsToAdd: ['tooltip']
});
```

`editable: true` is recommended because it makes tooltip annotations easier to work with after they are created:

- drag the annotation to a new position
- edit the text inline
- clear the text to remove the annotation

## Basic Usage With `tooltiptemplate`

`tooltiptemplate` uses the same interpolation style as `hovertemplate`. It controls the text shown inside the created annotation.

If no `tooltiptemplate` is supplied, Plotly falls back to a built-in default based on the clicked point data, typically `x`, `y`, and `z` when available.

Basic scatter example:

```js
const data = [{
  type: 'scatter',
  mode: 'markers',
  x: [1, 2, 3],
  y: [2, 1, 4],
  tooltiptemplate: 'x: %{x}<br>y: %{y}'
}];

Plotly.newPlot('graph', data, {}, {
  editable: true,
  modeBarButtonsToAdd: ['tooltip']
});
```

Heatmap or histogram2d example:

```js
const data = [{
  type: 'heatmap',
  z: [
    [1, 2, 3],
    [2, 5, 1],
    [0, 1, 4]
  ],
  tooltiptemplate: 'x: %{x}<br>y: %{y}<br>z: %{z}'
}];
```

## Styling With `tooltip`

`tooltip` accepts annotation-style properties. It is merged with Plotly's built-in defaults for tooltip annotations.

Common styling fields include:

- `bgcolor`
- `bordercolor`
- `font`
- `arrowcolor`

Example:

```js
const data = [{
  type: 'scatter',
  mode: 'markers',
  x: [1, 2, 3],
  y: [2, 1, 4],
  tooltiptemplate: 'Point<br>x: %{x}<br>y: %{y}',
  tooltip: {
    bgcolor: 'rgba(15, 23, 42, 0.92)',
    bordercolor: '#22c55e',
    arrowcolor: '#22c55e',
    font: {
      color: '#f8fafc',
      family: 'Segoe UI, Arial, sans-serif',
      size: 12
    }
  }
}];
```

## Runtime Customization With `tooltipfunction(ctx)`

`tooltipfunction(ctx)` is the executable customization layer. It is:

- JavaScript-only
- runtime-only
- attached directly to `gd.data[traceIndex]`
- not part of serialized figure JSON

Attach it after the plot exists:

```js
const gd = document.getElementById('graph');

gd.data[0].tooltipfunction = function(ctx) {
  return {
    point: {
      extraValue: 42
    }
  };
};
```

### Return Contract

The callback may return:

- `false` or `null` to cancel tooltip creation
- a `string` to replace the template text input
- an `object` with any of these fields:
  - `point`
  - `text`
  - `annotation`
  - `style`

Meaning of each object field:

- `point`: merged into the clicked point data before `tooltiptemplate` interpolation
- `text`: replaces the resolved template string
- `annotation`: overrides the created annotation object
- `style`: adds or overrides annotation styling

Example returning a string:

```js
gd.data[0].tooltipfunction = function(ctx) {
  return 'Clicked x=' + ctx.point.x + '<br>Clicked y=' + ctx.point.y;
};
```

Example cancelling the tooltip:

```js
gd.data[0].tooltipfunction = function(ctx) {
  if(ctx.point.y < 0) return false;
  return null; // also cancels
};
```

Example injecting extra fields used by `tooltiptemplate`:

```js
gd.data[0].tooltiptemplate = 'x: %{x}<br>y: %{y}<br>note: %{note}';

gd.data[0].tooltipfunction = function(ctx) {
  return {
    point: {
      note: 'nearest sample'
    }
  };
};
```

## `ctx` Reference

The callback receives a single `ctx` object with these fields:

- `gd`
- `eventData`
- `event`
- `point`
- `trace`
- `fullTrace`
- `calcdata`
- `fullLayout`
- `xaxis`
- `yaxis`

Practical usage of each field:

### `ctx.gd`

The graph div. Use this when you need direct access to the rendered plot object.

```js
console.log(ctx.gd.data.length);
```

### `ctx.eventData`

The `plotly_click` payload. Useful when you want the full click event object, including all clicked points.

```js
console.log(ctx.eventData.points);
```

### `ctx.event`

The raw mouse event. Useful for screen-space calculations such as `clientX`, `clientY`, or DOM target geometry.

```js
const rect = ctx.event.target.getBoundingClientRect();
console.log(ctx.event.clientX - rect.left, ctx.event.clientY - rect.top);
```

### `ctx.point`

The clicked point or bin event data. This is usually the main input for runtime tooltip logic.

Typical values include:

- `x`
- `y`
- `z`
- `pointIndex` or `pointIndices`
- `pointNumbers`
- `customdata` when available

```js
console.log(ctx.point.x, ctx.point.y, ctx.point.z);
```

### `ctx.trace`

The original input trace from `gd.data`. This is where runtime-only additions such as `tooltipfunction` live.

```js
console.log(ctx.trace.name);
console.log(ctx.trace.customdata);
```

### `ctx.fullTrace`

The computed/defaulted trace from `gd._fullData`. Use this for values that may have been defaulted or expanded during plotting.

```js
console.log(ctx.fullTrace.tooltiptemplate);
console.log(ctx.fullTrace.type);
```

### `ctx.calcdata`

Trace-specific calcdata. Useful for advanced logic on binned traces such as `histogram2d`, where you may want access to computed bins and values.

```js
const cd0 = ctx.calcdata[0];
console.log(cd0.z);
```

### `ctx.fullLayout`

The computed layout object. Useful for locale, mode, and subplot information.

```js
console.log(ctx.fullLayout.hovermode);
```

### `ctx.xaxis` and `ctx.yaxis`

The clicked subplot axes. Useful for coordinate conversions such as `c2p`, `p2c`, and related axis helpers.

```js
const xPixel = ctx.xaxis.c2p(ctx.point.x);
const yPixel = ctx.yaxis.c2p(ctx.point.y);
console.log(xPixel, yPixel);
```

## `customdata` In The Callback

For direct point-based traces, `customdata` is often available directly on the clicked point:

```js
gd.data[0].tooltipfunction = function(ctx) {
  console.log(ctx.point.customdata);
  return {
    point: {
      label: ctx.point.customdata
    }
  };
};
```

For aggregated traces like `histogram2d`, a clicked bin usually corresponds to many source samples. In that case, use `ctx.point.pointIndices` with `ctx.trace.customdata`:

```js
gd.data[0].tooltipfunction = function(ctx) {
  const indices = ctx.point.pointIndices || [];
  const rawCustom = indices.map(function(i) {
    return ctx.trace.customdata[i];
  });

  return {
    point: {
      sampleCount: indices.length,
      firstCustomValue: rawCustom[0]
    }
  };
};
```

## Examples

### Example 1: Basic Scatter Tooltip Annotation

```js
const trace = {
  type: 'scatter',
  mode: 'markers',
  x: [1, 2, 3],
  y: [2, 1, 4],
  tooltiptemplate: 'x: %{x}<br>y: %{y}',
  tooltip: {
    arrowcolor: 'blue'
  }
};

Plotly.newPlot('graph', [trace], {}, {
  editable: true,
  modeBarButtonsToAdd: ['tooltip']
});
```

### Example 2: Programmatic Attachment On An Existing Plot

```js
const gd = document.getElementById('graph');

gd.data[0].tooltiptemplate =
  'x: %{x}<br>y: %{y}<br>score: %{score}';

gd.data[0].tooltipfunction = function(ctx) {
  return {
    point: {
      score: ctx.point.y * 10
    }
  };
};
```

### Example 3: Styling A Tooltip Annotation

```js
const gd = document.getElementById('graph');

gd.data[0].tooltip = {
  bgcolor: 'rgba(255, 255, 255, 0.95)',
  bordercolor: '#2563eb',
  arrowcolor: '#2563eb',
  font: {
    color: '#111827',
    family: 'Georgia, serif',
    size: 13
  }
};
```

### Example 4: Histogram2d Or Heatmap Local Maximum

This pattern remaps the tooltip to the strongest value inside a rectangular kernel.

```js
const gd = document.getElementById('graph');

gd.data[0].tooltiptemplate =
  'Local max: %{z:.4f}<br>x: %{x:.3f}<br>y: %{y:.3f}<br>kernel: %{kernelSizeX} x %{kernelSizeY}';

gd.data[0].tooltipfunction = function(ctx) {
  const kernelSizeX = 3;
  const kernelSizeY = 3;
  const z = ctx.fullTrace._z || ctx.calcdata[0].z;
  const xs = (ctx.fullTrace._x && ctx.fullTrace._x.length) ?
    ctx.fullTrace._x :
    ctx.calcdata[0].xRanges.map(r => (r[0] + r[1]) / 2);
  const ys = (ctx.fullTrace._y && ctx.fullTrace._y.length) ?
    ctx.fullTrace._y :
    ctx.calcdata[0].yRanges.map(r => (r[0] + r[1]) / 2);

  const minX = ctx.point.x - kernelSizeX / 2;
  const maxX = ctx.point.x + kernelSizeX / 2;
  const minY = ctx.point.y - kernelSizeY / 2;
  const maxY = ctx.point.y + kernelSizeY / 2;

  let best = -Infinity;
  let bestX = ctx.point.x;
  let bestY = ctx.point.y;

  for(let iy = 0; iy < ys.length; iy++) {
    if(ys[iy] < minY || ys[iy] > maxY) continue;
    for(let ix = 0; ix < xs.length; ix++) {
      if(xs[ix] < minX || xs[ix] > maxX) continue;
      const value = z[iy][ix];
      if(value > best) {
        best = value;
        bestX = xs[ix];
        bestY = ys[iy];
      }
    }
  }

  return {
    point: {
      x: bestX,
      y: bestY,
      z: best,
      kernelSizeX,
      kernelSizeY
    },
    annotation: {
      x: bestX,
      y: bestY
    }
  };
};
```

### Example 5: Cancellation

```js
gd.data[0].tooltipfunction = function(ctx) {
  if(ctx.point.z !== undefined && ctx.point.z < 0) {
    return false;
  }

  return {
    point: {
      accepted: true
    }
  };
};
```

## Caveats And Notes

- `tooltipfunction(ctx)` is synchronous only
- the callback is runtime-only and not JSON-serializable
- wrapper languages such as Python or R need custom injected JavaScript if they want callback behavior
- tooltip annotations are real layout annotations, so adding them may trigger noticeable redraw latency on very large heatmaps or other heavy figures
- the modebar tooltip button must be enabled and activated by the user before click-to-add behavior starts
