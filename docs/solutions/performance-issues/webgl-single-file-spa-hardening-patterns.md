---
title: WebGL single-file SPA hardening patterns
category: performance-issues
date: 2026-04-23
source_pr: https://github.com/abeage1/stanley-kaleidoscope-web/pull/1
source_commits: [3206e99, 84d90f7]
tags:
  - webgl
  - single-file
  - static-site
  - architecture
  - performance
  - agent-native
  - csp
  - github-pages
---

# WebGL single-file SPA hardening patterns

Reusable patterns surfaced while auditing and refactoring a zero-dependency
single-file WebGL SPA (vanilla JS + GLSL, deployed via GitHub Pages, no build
step). Applied all six in a single pass on `index.html` (779 → 963 lines).

The problems each pattern addresses are easy to miss on small projects
because nothing obviously breaks in day-to-day use — they bite on edge
devices (older iOS), rapid input (slider scrubs), or when the app has to
grow beyond its initial feature set.

## 1. Lift DOM-coupled state into a plain object

**Symptom.** Adding undo/redo, presets, URL-shareable state, or agent
automation on top of a DOM-coupled render loop requires rewriting the render
loop. "Every feature pays the tax."

**Root cause.** `render()` reads values directly from slider `.value` at
draw time. State is implicit in the DOM. No way to snapshot, serialize,
round-trip, or mutate programmatically.

**Solution.** Treat state as data. One plain object holds every user-facing
value in human units; a config table maps DOM↔state.

```js
const state = { mode: 0, p1: 8, p2: 1.00, rot: 0, zoom: 1.00, cx: 50, cy: 50 };

// scale = DOM integer → state divisor (state in human units)
const SLIDERS = {
  p1:   { el: $('p1'),   scale: 1,   fmt: (v, m) => MODES[m].fmt(v) },
  rot:  { el: $('rot'),  scale: 10,  fmt: v => v.toFixed(1) + '°' },
  zoom: { el: $('zoom'), scale: 100, fmt: v => v.toFixed(2) + '×' },
  // ...
};

function syncStateFromDom() {
  state.mode = parseInt(modeEl.value, 10);
  for (const k of Object.keys(SLIDERS)) state[k] = +SLIDERS[k].el.value / SLIDERS[k].scale;
}
function render() { /* reads state, not DOM */ }
```

**Prevention.** On any DOM-coupled vanilla JS app you plan to extend,
this is P1 *before feature work* — every later feature (undo, presets,
URL share, agent API, tests) compounds off the same refactor. Don't do it
piecemeal.

## 2. Always clamp to GPU limits in WebGL apps

**Symptom.** On older iOS devices (`MAX_TEXTURE_SIZE` = 4096), uploading a
6000 px Unsplash sample produces silently clipped or all-black output. On
desktop, a 20000×20000 PNG (~1.6 GB decoded) OOM-crashes the tab.

**Root cause.** `gl.texImage2D` and `gl.drawArrays` both have device-
specific size limits. Exceeding them is not reported as an error — the
driver truncates or produces undefined output. Nobody checks.

**Solution.** Query limits at init, clamp on both upload *and* export, and
add a soft pixel cap to reject pathological inputs before they hit the GPU.

```js
const MAX_TEX     = gl.getParameter(gl.MAX_TEXTURE_SIZE);
const MAX_VP      = gl.getParameter(gl.MAX_VIEWPORT_DIMS);
const MAX_VP_SIZE = Math.min(MAX_VP[0], MAX_VP[1]);

function uploadToGL(source, name) {
  const w = source.naturalWidth  || source.width;
  const h = source.naturalHeight || source.height;
  if (w * h > 64_000_000) {              // soft cap rejects OOM inputs
    setStatus(`Image too large: ${w}×${h}.`); return;
  }
  const img = (w > MAX_TEX || h > MAX_TEX) ? downscaleTo(MAX_TEX, source) : source;
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, img);
  // …
}

// On export, clamp the export dimensions too:
if (Math.max(w, h) > MAX_VP_SIZE) { /* scale w,h down, warn the user */ }
```

**Prevention.** Every WebGL app — no matter how toy — should have these
four lines at init and two clamp sites. Silent truncation on one minority
device class is a real bug, not theoretical.

## 3. Coalesce rapid input into a single rAF draw

**Symptom.** Scrubbing a slider with a high-polling mouse or dragging a
window edge triggers many synchronous draws per animation frame. Redundant
GPU work, blocking the pipeline; window resize also reallocates the
drawing buffer each time.

**Root cause.** DOM `input`/`resize` events fire faster than vsync. Each
handler calls `render()` immediately.

**Solution.** One helper, used everywhere user input triggers a draw.

```js
let pending = false;
function scheduleRender() {
  if (pending) return;
  pending = true;
  requestAnimationFrame(() => {
    pending = false;
    syncStateFromDom(); updateLabels(); render();
  });
}

sliders.forEach(s => s.addEventListener('input', scheduleRender));
window.addEventListener('resize', scheduleRender);
canvas.addEventListener('mousemove', e => { if (dragging) { /* ... */ scheduleRender(); } });
```

**Prevention.** Never call `render()` directly from a per-event handler
that can fire faster than vsync. Export/save paths can still call the raw
`render()` synchronously when you need a specific frame.

## 4. Agent-native parity for client-only SPAs = three small pieces

**Symptom.** The app can only be driven by simulating DOM events on
unlabeled elements; no URL sharing, no programmatic setState, no reliable
selectors for automation.

**Root cause.** The app was built for human users. Agents (Playwright,
Puppeteer, MCP tools, iframe hosts) have no stable contract.

**Solution.** Three pieces, ~45 LOC total:

```js
// (a) Global control surface
window.__kaleido = {
  get state() { return { ...state }; },
  setState(patch)          { /* validate, write state, sync DOM, render */ },
  loadImageFromUrl,
  exportPNG,               // returns Promise<Blob>
  ready: true,
};

// (b) URL-hash state — bookmarkable / shareable / agent-navigable
history.replaceState(null, '', '#m=0&p1=8&rot=45&zoom=1.5&cx=50&cy=50');

// (c) data-kaleido attributes on every interactive control
// <input type="range" data-kaleido="slider-rot" …>
// stable selectors even if IDs change

// Optional: postMessage bridge, only when embedded
if (window.parent !== window) {
  window.addEventListener('message', e => {
    if (e.data?.type === 'setState') window.__kaleido.setState(e.data.payload);
    // …with replyId echo for request/reply
  });
}
```

**Prevention.** Prerequisite: pattern #1 (state as data). Once you have
that, agent parity is nearly free. Treat every new feature as state-first —
if it can't be represented in the hash URL, it probably shouldn't exist.

## 5. Drop `preserveDrawingBuffer: true`; read pixels on export instead

**Symptom.** Every preview frame pays a measurable FPS cost on Apple GPUs
and Adreno, just so `canvas.toBlob` in the export handler can capture the
frame.

**Root cause.** `preserveDrawingBuffer: true` prevents the driver from
clearing the backbuffer after present, enabling async capture — but it's
paid every frame forever for a feature used only on explicit save.

**Solution.** Export via `readPixels` into a 2D canvas. Works regardless
of `preserveDrawingBuffer` because `readPixels` runs before present.

```js
async function exportPNG() {
  const prevW = canvas.width, prevH = canvas.height;
  canvas.width = w; canvas.height = h;
  render();                                                    // synchronous

  const px = new Uint8ClampedArray(w * h * 4);
  gl.readPixels(0, 0, w, h, gl.RGBA, gl.UNSIGNED_BYTE, px);    // read before present

  // Flip Y in place (GL y-up → canvas y-down)
  const row = w * 4, tmp = new Uint8ClampedArray(row);
  for (let y = 0; y < (h >> 1); y++) {
    const t = y * row, b = (h - 1 - y) * row;
    tmp.set(px.subarray(t, t + row));
    px.copyWithin(t, b, b + row);
    px.set(tmp, b);
  }

  canvas.width = prevW; canvas.height = prevH;
  render();

  const out = document.createElement('canvas');
  out.width = w; out.height = h;
  out.getContext('2d').putImageData(new ImageData(px, w, h), 0, 0);
  return new Promise(r => out.toBlob(r, 'image/png'));
}
```

**Prevention.** Reach for `preserveDrawingBuffer: true` only if you've
measured and need it. The `readPixels` path is safer (browser-agnostic),
faster (no per-frame tax), and free of race conditions around when the
browser presents.

## 6. CSP on a zero-dependency single-file GitHub Pages app

**Symptom.** GitHub Pages serves no `Content-Security-Policy`,
`X-Frame-Options`, or `Referrer-Policy`. The app has no sensitive surface,
but a future `innerHTML` regression would have no guardrail.

**Root cause.** No server config to add headers; static hosting only.

**Solution.** Meta tags. Keep `'unsafe-inline'` for script/style because
the whole app is intentionally in one file.

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  img-src 'self' blob: data: https://images.unsplash.com;
  style-src 'self' 'unsafe-inline';
  script-src 'self' 'unsafe-inline';
  connect-src 'none';
  frame-ancestors *;
  base-uri 'none';
">
<meta name="referrer" content="no-referrer">
```

Notes:
- `connect-src 'none'` is correct when all external resources load via
  `<img>` — those use `img-src`, not `connect-src`.
- `frame-ancestors *` if you want the app embeddable; otherwise `'none'`.
- Keep `'unsafe-inline'` only because "single file" is a deliberate design
  constraint. For apps that can move script/style to external files, drop it.

**Prevention.** Ship a CSP meta tag from day one on any static app, even
when the current threat surface is zero. It's ~5 lines and catches future
regressions automatically.

## See also

- PR: https://github.com/abeage1/stanley-kaleidoscope-web/pull/1
- Review artifacts: `todos/` in the `review/initial-audit` branch
- Commits: `3206e99` (12 triage todos), `84d90f7` (implementation)
