# CLAUDE.md

Guidance for Claude (and other agents) working in this repo.

## What this is

A single-page WebGL kaleidoscope effect tool, deployed as a static file via GitHub Pages. Everything lives in one file: `index.html`.

## Design constraints — hold these fixed

These are intentional product decisions, not technical debt. Do not propose reversing them without explicit approval.

- **Single file.** CSS, GLSL, HTML, and JS all live inside `index.html`. No external `.js`, `.css`, or `.glsl` files.
- **Zero runtime dependencies.** No framework, no bundler, no package manager, no CDN imports at runtime. Works entirely offline after first load.
- **No build step.** `index.html` is the source of truth and the deploy artifact. Do not introduce Webpack, Vite, Rollup, Parcel, tsc, esbuild, a task runner, or a dev-server dependency.
- **No TypeScript, no JSX, no transpilation.** Vanilla JS that runs directly in the browser.
- **No test framework.** If tests become worth having, they live in `index.html` behind a `?debug=1` guard or as a separate `tests.html`, not in `node_modules`.

When a change tempts you toward any of the above, the right answer is almost always to write the thing more concretely in vanilla JS and keep it in `index.html`.

## State contract — honor this

The tool is a non-destructive layered compositor. State lives in one plain
object in human units:

```js
const state = {
  activeLayerIndex: 0,
  layers: [
    { kind: 'base',   mode: 0, p1: 8, p2: 1.00, rot: 0, zoom: 1.00, cx: 50, cy: 50 },
    { kind: 'region', shape: 'rect'|'ellipse',
      bounds: { x, y, w, h },            // 0..1 canvas UV, top-left origin, frozen
      source: 'original'|'prior'|'crop',
      feather: 0..1,                     // fraction of region's min half-extent
      mode, p1, p2, rot, zoom, cx, cy },
    // …more region layers stacked on top
  ],
};
```

- Invariant: `layers[0].kind === 'base'` always. The base layer cannot be deleted or moved.
- `activeLayerIndex` names the layer that sliders, the mode picker, and drag gestures currently target.
- `state.layers[*].mode` is `0..4` (matches the `<select>` value).
- Slider values stored on layers are **human units**: degrees, ratios, percent. DOM sliders store integers; the `SLIDERS` config table maps DOM ↔ state via its `scale` field and operates on `activeLayer()`.
- For `source='crop'`, `cx` / `cy` are percentages of the **region**, not the canvas.
- `render()` is a pure function of `state` (plus texture). It does **not** read DOM values directly.
- All user-driven triggers (slider `input`, resize, canvas drag, mode change, layer list actions, region-draw gesture) go through `scheduleRender()` or explicit mutation helpers — each rAF-coalesces to at most one draw per frame.
- Direct `render()` calls are reserved for the synchronous export path and the explicit mutation helpers (`setActiveLayer`, `addRegion`, `removeLayer`, `reorderLayer`, `resetAll`) whose caller owns the frame cadence.

Every new feature should extend this model, not work around it:

- If the feature adds a per-layer tunable, add a field to every relevant layer kind, plus SLIDERS or the region-controls pathway, plus hash serialization.
- If the feature changes rendering, it reads from `state.layers` (never from DOM).
- If the feature is visible in the product, it should round-trip via the URL hash. `writeStateToHash` / `readStateFromHash` define the schema — extend them together.

## Rendering pipeline

Rendering walks `state.layers` through **two ping-pong FBOs** and blits the
final FBO to the default framebuffer. This keeps the export path's
`preserveDrawingBuffer=false` invariant: `gl.readPixels` on the default
framebuffer after the blit still produces the composite.

- FBO targets are clamped to `min(canvas.*, MAX_TEXTURE_SIZE, CONFIG.maxFboSize)` (default 4096). Two RGBA8 FBOs at the cap = ~128 MB VRAM.
- A base-only stack at a canvas size that exceeds the FBO cap bypasses the pipeline entirely (direct render to default framebuffer) so single-layer 4K+ exports keep native resolution.
- Layer N reads layer N-1's composite from TEXTURE1 (`u_prev_tex`); the source image stays bound to TEXTURE0. The main fragment shader branches on `u_source` (original / prior / crop) to pick which sampler feeds the kaleidoscope math, and on `u_region_kind` to decide whether to mix the tiled result against the prior backdrop through a feathered SDF mask.
- FBO allocation failure falls back to a single-layer base render with a user-visible status warning. Any region layers above it are silently dropped for that frame.

## Agent surface — keep stable

`window.__kaleido` is a public API contract. Consumers include browser automation, MCP tools, iframe embeds, and future Claude sessions. Changes to its shape are breaking changes.

Current contract (apiVersion 2):

- `apiVersion: 2`
- `state` getter → deep copy of `{ activeLayerIndex, layers }`
- `setState(patch)` → merges `activeLayerIndex?` and `layers?` (full array replacement; `layers[0].kind` must be `'base'`). v1 flat patches like `{ mode: 2 }` are rejected with a `console.error` pointing here.
- `addRegion({ shape, bounds, source?, feather? })` → pushes a new region layer, makes it active, returns its index (or `-1` on validation failure).
- `removeLayer(index)` (index ≥ 1)
- `setActiveLayer(index)`
- `reorderLayer(from, to)` (from, to ≥ 1)
- `loadImageFromUrl(url, name)`, `reset()`, `exportPNG() → Blob`, `MODES`, `ready: true`

`data-kaleido="…"` attributes are also part of the contract — renaming or removing one breaks external selectors. Layer rows in the sidebar carry `data-kaleido="layer-{idx}"`.

The `postMessage` bridge only activates when `window.parent !== window`. Its message types mirror the methods above (`getState`, `setState`, `addRegion`, `removeLayer`, `setActiveLayer`, `reorderLayer`, `loadImage`, `exportPNG`).

The `kaleido:change` CustomEvent fires on every render and every structural mutation with `{ detail: cloneState() }` (the full v2 shape).

## GPU safety

Any WebGL path that takes an image size or a viewport size must clamp:

- Upload: `maybeDownscale` against `MAX_TEXTURE_SIZE`; reject inputs over `CONFIG.maxInputPixels`.
- FBO allocation: `ensureFBOs` clamps to `min(canvas.*, MAX_TEXTURE_SIZE, CONFIG.maxFboSize)` and handles allocation failure explicitly.
- Export: clamp target `w × h` to `MAX_VP_SIZE` and surface the clamp in the status bar.

Do not add a new code path that calls `gl.texImage2D` or changes `canvas.width/height` without going through these checks. Do not allocate a framebuffer without going through `createFBO` / `ensureFBOs`.

## Export path quirk

Export does **not** use `preserveDrawingBuffer: true`. It calls `gl.readPixels` on the default framebuffer (which holds the blit output from the pipeline) into a `Uint8ClampedArray`, flips Y in place, and draws that into an offscreen 2D canvas before `toBlob`. If you touch `exportPNG`, preserve this shape — the alternative (turning `preserveDrawingBuffer` back on) pays a per-frame FPS tax on Apple GPUs.

## CSP

There is a `<meta http-equiv="Content-Security-Policy">` at the top of the file. New behavior that requires a different `connect-src`, `img-src`, `frame-ancestors`, etc., updates the meta tag in the same change.

Keep `'unsafe-inline'` on script/style — the single-file design needs it.

## Code style

- Short, direct vanilla JS. No class hierarchies for things that want to be plain objects.
- Comments explain *why* (non-obvious invariant, hidden constraint), not *what*. Decorative section banners are fine sparingly.
- Magic numbers live in `CONFIG` at the top of the script.
- Identifiers follow the existing pattern: `btn*` for buttons, `slider-*` / `val-*` for data-kaleido attrs.

## When in doubt

- Read `todos/` for the prioritized backlog with solution options already laid out.
- Read `docs/solutions/performance-issues/webgl-single-file-spa-hardening-patterns.md` for the six reusable patterns this codebase already applies.
- Read the `<!-- AGENT API -->` HTML comment at the top of the `<script>` block in `index.html` — it's the canonical API reference.
