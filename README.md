# Stanley Kaleidoscope Web

A browser-based kaleidoscope image effect app — no installation, no backend, no dependencies.

**[Open the app →](https://abeage1.github.io/stanley-kaleidoscope-web/)**

## Features

- **5 kaleidoscope modes** — Radial, Rectangle, Triangle 45-45-90, Triangle 60-60-60, Triangle 30-60-90
- **WebGL rendering** — GPU-accelerated, updates instantly as you move sliders
- **Click/drag the preview** to set the sampling center point
- **Full-resolution PNG export** — exports at the original image's native resolution
- **Drag-and-drop** image loading
- **Shareable URLs** — the current mode + controls live in the URL hash
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

Every control change is reflected in the URL. Copy the URL, paste it anywhere — opening it reproduces the exact state.

```
#m=2&p1=30&rot=45&zoom=1.5&cx=50&cy=50
```

| key  | meaning                          |
|------|----------------------------------|
| `m`  | mode (0–4)                       |
| `p1` | segments (radial) / tile-size %  |
| `p2` | rectangle aspect ratio           |
| `rot`| rotation in degrees              |
| `zoom`| zoom ratio                      |
| `cx` / `cy` | center percent             |
| `img` | *(optional)* URL of an image to load |

## Scripting & automation

The app exposes a small control surface for scripting, browser-automation tools, and agents.

```js
// Read / write state
window.__kaleido.state
// → { mode, p1, p2, rot, zoom, cx, cy }

window.__kaleido.setState({ mode: 2, rot: 45, zoom: 1.5 })
await window.__kaleido.exportPNG()         // → Blob
window.__kaleido.loadImageFromUrl(url, name)
window.__kaleido.reset()

// Observe state changes
document.addEventListener('kaleido:change', e => console.log(e.detail))
```

Every interactive element also carries a stable `data-kaleido` attribute (`mode`, `slider-p1`…`slider-cy`, `btn-open`, `btn-save`, `btn-reset`, `sample`), so tools like Playwright or Puppeteer can drive the UI without relying on visual or ID-based selectors.

When embedded in an iframe, the app also accepts a small `postMessage` RPC:

```js
iframe.contentWindow.postMessage(
  { type: 'setState', payload: { mode: 2, rot: 45 }, replyId: 1 },
  '*'
)
```

Supported types: `getState`, `setState`, `loadImage`, `exportPNG`.

## Running locally

Just open `index.html` in any modern browser — no server needed.

## Development

See [CLAUDE.md](CLAUDE.md) for design principles and the project's state contract.
