# Stanley Kaleidoscope Web

A browser-based kaleidoscope image effect app — no installation, no backend, no dependencies.

**[Open the app →](https://abeage1.github.io/stanley-kaleidoscope-web/)**

## Features

- **5 kaleidoscope modes** — Radial, Rectangle, Triangle 45-45-90, Triangle 60-60-60, Triangle 30-60-90
- **Layered regional kaleidoscopes** — stack rect/ellipse region layers on top of the full-canvas base, each with its own mode, params, source (original / prior composite / region crop), and feather
- **WebGL rendering** — GPU-accelerated ping-pong FBO pipeline, updates instantly as you move sliders
- **Click/drag the preview** to set the sampling center point of the active layer
- **Full-resolution PNG export** — exports the full layer stack at native resolution
- **Drag-and-drop** image loading
- **Shareable URLs** — the entire layer stack round-trips through the URL hash
- **Zero dependencies** — single `index.html` file, works offline

## Controls

| Control | Radial | Tile modes |
|---|---|---|
| Segments / Tile Size | 1–24 segments (1 = passthrough) | 5%–200% of image |
| Tile Aspect *(Rectangle only)* | — | 0.25×–4.00× |
| Rotation | 0°–360° | 0°–360° |
| Zoom | 0.10×–3.00× | 0.10×–3.00× |
| Center X / Y | 0%–100% | 0%–100% |

## Sharing

Every control change, every region layer, and the active selection are reflected in the URL. Copy the URL, paste it anywhere — opening it reproduces the exact composition.

```
#a=1&L=b_m=2_p1=30_p2=1.5_rot=45_zoom=1.5_cx=50_cy=50|r_sh=ellipse_x=0.2_y=0.25_w=0.5_h=0.4_src=prior_f=0.3_m=0_p1=6_p2=1_rot=0_zoom=1_cx=50_cy=50
```

| key  | meaning                                                     |
|------|-------------------------------------------------------------|
| `a`  | active layer index                                          |
| `L`  | `|`-separated layer list; fields inside each layer are `_`-separated |
| `img` | *(optional)* URL of an image to load                       |

Inside a layer:

| key   | layers   | meaning                                               |
|-------|----------|-------------------------------------------------------|
| leading `b` / `r` | all | layer kind — base or region                    |
| `sh`  | region   | `rect` or `ellipse`                                  |
| `x,y,w,h` | region | region bounds (0..1 canvas UV, top-left origin)   |
| `src` | region   | `original`, `prior`, or `crop`                       |
| `f`   | region   | feather 0..1                                         |
| `m`   | all      | mode 0..4                                            |
| `p1,p2,rot,zoom,cx,cy` | all | kaleidoscope params                        |

Legacy flat URLs from earlier versions (`#m=2&p1=30&…`) still load — they are migrated to a single-base-layer v2 stack on read.

## Scripting & automation

The app exposes a control surface for scripting, browser automation, and agents. The state shape is layered (apiVersion 2).

```js
window.__kaleido.apiVersion            // 2
window.__kaleido.state
// → { activeLayerIndex, layers: [{ kind: 'base', mode, p1, p2, rot, zoom, cx, cy }, …] }

// Replace layers or switch active layer
window.__kaleido.setState({ activeLayerIndex: 0 })
window.__kaleido.setState({ layers: [...modifiedCopy] })

// Add a region over the top-left quadrant, sampling the prior composite
const idx = window.__kaleido.addRegion({
  shape: 'ellipse',
  bounds: { x: 0.1, y: 0.1, w: 0.4, h: 0.35 },
  source: 'prior',
  feather: 0.2,
})

window.__kaleido.setActiveLayer(idx)
window.__kaleido.reorderLayer(idx, 1)
window.__kaleido.removeLayer(idx)

await window.__kaleido.exportPNG()     // → Blob of the full stack
window.__kaleido.loadImageFromUrl(url, name)
window.__kaleido.reset()               // fresh single-base-layer stack

// Observe every mutation
document.addEventListener('kaleido:change', e => console.log(e.detail))
```

Every interactive element carries a stable `data-kaleido` attribute (`mode`, `slider-p1`…`slider-cy`, `slider-feather`, `btn-open`, `btn-save`, `btn-reset`, `btnAddRegion`, `btnShapeRect`, `btnShapeEllipse`, `btnCancelDraw`, `layers`, `layer-{idx}`, `region-controls`, `src-original`, `src-prior`, `src-crop`, `sample`, `marquee`), so tools like Playwright or Puppeteer can drive the UI without relying on visual or ID-based selectors.

When embedded in an iframe, the app also accepts a `postMessage` RPC:

```js
iframe.contentWindow.postMessage(
  { type: 'addRegion',
    payload: { shape: 'rect', bounds: { x: 0.2, y: 0.2, w: 0.4, h: 0.4 } },
    replyId: 1 },
  '*'
)
```

Supported types: `getState`, `setState`, `addRegion`, `removeLayer`, `setActiveLayer`, `reorderLayer`, `loadImage`, `exportPNG`.

## Running locally

Just open `index.html` in any modern browser — no server needed.

## Development

See [CLAUDE.md](CLAUDE.md) for design principles and the project's state contract.
