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

See Plotly's hovertemplate documentation:
https://plotly.com/javascript/hover-text-and-formatting/

Numeric formatting inside `%{...}` uses D3 format syntax, for example `%{x:.2f}` or `%{z:.3e}`.
See the D3 format syntax reference:
https://d3js.org/d3-format

If no `tooltiptemplate` is supplied, Plotly falls back to a built-in default based on the clicked point data, typically `x`, `y`, and `z` when available.

Basic scatter example:

```js
const data = [{
  type: 'scatter',
  mode: 'markers',
  x: [1, 2, 3],
  y: [2, 1, 4],
  tooltiptemplate: 'x: %{x:.2f}<br>y: %{y:.3f}'
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
  tooltiptemplate: 'x: %{x:.1f}<br>y: %{y:.1f}<br>z: %{z:.3f}'
}];
```

## Styling With `tooltip`

`tooltip` accepts annotation-style properties. It is merged with Plotly's built-in defaults for tooltip annotations.

These are the same annotation-style properties documented in Plotly's text and annotations page:
https://plotly.com/javascript/text-and-annotations/

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
  tooltiptemplate: 'Point<br>x: %{x:.2f}<br>y: %{y:.2f}',
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
- `text`: replaces the resolved template string before interpolation
- `annotation`: overrides fields on the annotation object that Plotly is about to create
- `style`: adds or overrides annotation styling after the trace-level `tooltip` style is read

Examples for each field:

Example using `point`:

```js
gd.data[0].tooltiptemplate = 'x: %{x}<br>y: %{y}<br>score: %{score:.2f}';

gd.data[0].tooltipfunction = function(ctx) {
  return {
    point: {
      score: ctx.point.y * 10
    }
  };
};
```

Example returning a string:

```js
gd.data[0].tooltipfunction = function(ctx) {
  return 'Clicked x=' + ctx.point.x.toFixed(2) + '<br>Clicked y=' + ctx.point.y.toFixed(2);
};
```

Example using `text`:

```js
gd.data[0].tooltipfunction = function(ctx) {
  return {
    text: 'Formatted value<br>y = ' + ctx.point.y.toExponential(3)
  };
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

Example using `annotation`:

```js
gd.data[0].tooltipfunction = function(ctx) {
  return {
    annotation: {
      ax: 30,
      ay: -40,
      xanchor: 'left'
    }
  };
};
```

Use `annotation` when you want to control the annotation object itself, for example:

- force a different arrow offset with `ax` and `ay`
- change the box anchor with `xanchor` or `yanchor`
- override the annotation position independently from the formatted point fields

Example using `style`:

```js
gd.data[0].tooltipfunction = function(ctx) {
  return {
    style: {
      arrowcolor: 'crimson',
      bordercolor: 'crimson'
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

The fully processed trace from `gd._fullData`. It is the plotted version of the trace after Plotly has applied defaults and internal preprocessing. Use it when you need the trace as Plotly is actually using it, not only the raw input you provided in `gd.data`.

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

The fully processed layout object from the rendered plot. Use it when you need the active layout state that Plotly is currently using, such as the current locale, axis objects, hover mode, subplot internals, or computed defaults.

```js
console.log(ctx.fullLayout.hovermode);
console.log(ctx.fullLayout._d3locale);
```

### `ctx.xaxis` and `ctx.yaxis`

The clicked subplot axes. These are Plotly axis objects with conversion helpers.

Common internal helpers include:

- `c2p`: data coordinate to pixel
- `p2c`: pixel to data coordinate
- `d2c`: displayed value to coordinate, often useful on categorical axes

These helpers are internal Plotly axis methods rather than public top-level API methods, so they are best treated as advanced usage helpers inferred from the current implementation.

```js
const xPixel = ctx.xaxis.c2p(ctx.point.x);
const yPixel = ctx.yaxis.c2p(ctx.point.y);
console.log(xPixel, yPixel);
```

See Plotly's event data documentation for the general click-event payload shape:
https://plotly.com/javascript/plotlyjs-events/

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
  tooltiptemplate: 'x: %{x:.2f}<br>y: %{y:.2f}',
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
  'x: %{x:.2f}<br>y: %{y:.2f}<br>score: %{score:.1f}';

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

In this example:

- `kernelSizeX` and `kernelSizeY` are in plot data units, not pixels
- the callback searches nearby heatmap or histogram2d cells
- returning new `point.x`, `point.y`, and `point.z` changes both the formatted values and the default arrow anchor position

If you only return `point.x` and `point.y`, the arrow already moves to that remapped point. You only need `annotation` as well if you want the annotation object itself to differ, for example with a custom `ax`, `ay`, `xanchor`, or a deliberately different `x` / `y` than the remapped point.

```js
const gd = document.getElementById('graph');

gd.data[0].tooltiptemplate =
  'Local max: %{z:.4f}<br>x: %{x:.3f}<br>y: %{y:.3f}<br>kernel: %{kernelSizeX} x %{kernelSizeY}';

gd.data[0].tooltipfunction = function(ctx) {
  // Kernel size in plot units, not pixels.
  const kernelSizeX = 3;
  const kernelSizeY = 3;

  // For heatmap-like traces, Plotly stores the plotted z matrix in calcdata.
  // If _x/_y are present on the full trace, they are the plotted x/y centers.
  const z = ctx.fullTrace._z || ctx.calcdata[0].z;
  const xs = (ctx.fullTrace._x && ctx.fullTrace._x.length) ?
    ctx.fullTrace._x :
    ctx.calcdata[0].xRanges.map(r => (r[0] + r[1]) / 2);
  const ys = (ctx.fullTrace._y && ctx.fullTrace._y.length) ?
    ctx.fullTrace._y :
    ctx.calcdata[0].yRanges.map(r => (r[0] + r[1]) / 2);

  // Search bounds centered on the clicked point.
  const minX = ctx.point.x - kernelSizeX / 2;
  const maxX = ctx.point.x + kernelSizeX / 2;
  const minY = ctx.point.y - kernelSizeY / 2;
  const maxY = ctx.point.y + kernelSizeY / 2;

  let best = -Infinity;
  let bestX = ctx.point.x;
  let bestY = ctx.point.y;

  // Scan the bins whose centers fall inside the kernel rectangle.
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
    // These remapped point fields drive both template interpolation and
    // the default arrow anchor position.
    point: {
      x: bestX,
      y: bestY,
      z: best,
      kernelSizeX,
      kernelSizeY
    }
  };
};
```

If you want the annotation object itself to differ from the remapped point, add `annotation` explicitly:

```js
return {
  point: {
    x: bestX,
    y: bestY,
    z: best,
    kernelSizeX,
    kernelSizeY
  },
  annotation: {
    ax: 40,
    ay: -30
  }
};
```

### Example 5: Cancellation

Returning `false` or `null` is useful for conditional display when only some points or bins should produce tooltip annotations.

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
