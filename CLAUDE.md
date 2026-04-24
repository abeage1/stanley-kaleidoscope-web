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

All user-facing state lives in one plain object in human units:

```js
const state = { mode: 0, p1: 8, p2: 1.00, rot: 0, zoom: 1.00, cx: 50, cy: 50 };
```

- `state.mode` is `0..4` (matches the `<select>` value).
- Slider values in `state` are **human units**: degrees, ratios, percent. DOM sliders store integers; the `SLIDERS` config table maps DOM ↔ state via its `scale` field.
- `render(state)` is a pure function of `state` (plus texture). It does **not** read DOM values directly.
- All user-driven triggers (slider `input`, resize, canvas drag, mode change) go through `scheduleRender()`, which rAF-coalesces to at most one draw per frame.
- Direct `render()` calls are reserved for the synchronous export path.

Every new feature should extend this model, not work around it:

- If the feature adds a tunable, add an entry to `SLIDERS` and a field to `state`.
- If the feature changes rendering, it reads from `state`.
- If the feature is visible in the product, it should be representable in the URL hash (so it round-trips via share-and-paste). `writeStateToHash` and `readStateFromHash` define the schema — extend them together.

## Agent surface — keep stable

`window.__kaleido` is a public API contract. Consumers may include browser automation, MCP tools, iframe embeds, and future Claude sessions. Changes to its shape are breaking changes:

- `state` getter, `setState(patch)`, `loadImageFromUrl`, `reset`, `exportPNG`, `MODES`, `ready`.

`data-kaleido="…"` attributes are also part of the contract — renaming or removing one breaks external selectors.

The `postMessage` bridge only activates when `window.parent !== window`.

The `kaleido:change` CustomEvent fires on every render with `{ detail: { ...state } }`.

## GPU safety

Any WebGL path that takes an image size or a viewport size must clamp:

- Upload: `maybeDownscale` against `MAX_TEXTURE_SIZE`; reject inputs over `CONFIG.maxInputPixels`.
- Export: clamp target `w × h` to `MAX_VP_SIZE` and surface the clamp in the status bar.

Do not add a new code path that calls `gl.texImage2D` or changes `canvas.width/height` without going through these checks.

## Export path quirk

Export does **not** use `preserveDrawingBuffer: true`. It calls `gl.readPixels` into a `Uint8ClampedArray`, flips Y in place, and draws that into an offscreen 2D canvas before `toBlob`. If you touch `exportPNG`, preserve this shape — the alternative (turning `preserveDrawingBuffer` back on) pays a per-frame FPS tax on Apple GPUs.

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
