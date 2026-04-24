---
title: Layered Regional Kaleidoscope
type: feat
status: completed
date: 2026-04-24
origin: docs/brainstorms/2026-04-24-regional-selection-brainstorm.md
branch: feat/regional-selection
---

# Layered Regional Kaleidoscope

## Overview

Turn the kaleidoscope tool into a non-destructive layered compositor. The base full-canvas kaleidoscope becomes layer 0; users can stack rectangular or elliptical **region layers** on top, each with its own mode, params, source image, and optional feathered edge. The whole stack is editable forever (params only — bounds freeze on commit), round-trips through the URL hash, and renders via a ping-pong framebuffer pipeline. The public `window.__kaleido` API migrates to a new layered shape in a single breaking-change commit alongside a CLAUDE.md update.

(See brainstorm: `docs/brainstorms/2026-04-24-regional-selection-brainstorm.md`.)

## Problem Statement

Today's tool applies a single kaleidoscope across the entire canvas. Users who want to combine kaleidoscope effects — for example, folding only a face in a photo while leaving the rest intact, or stamping a small badge-of-kaleidoscope over a finished composition — must leave the tool and composite manually in other software.

The ask (from the feature request): *"select a part of the image and do these kaleidoscope operations just on that part while leaving the region outside of selected unchanged."* Brainstorm refined this to a **layered** model where "outside" means "whatever was shown just before you drew this region" — a paint-over-what-you-made mental model, not an ROI mask against the original image.

## Proposed Solution

A layered non-destructive pipeline:

- `state.layers[0]` is a **base** layer — a full-canvas kaleidoscope, equivalent to today's single output.
- `state.layers[1..N]` are **region** layers — axis-aligned rect or ellipse with their own kaleidoscope params. Each region has a configurable `source` (`original` / `prior` / `crop`, default `prior`) and a `feather` value (0 = hard edge, up to 1 = soft).
- Rendering walks the stack through **two ping-pong FBOs**: each pass reads the prior composite and writes the next. The final FBO is blitted to the default framebuffer (the visible canvas).
- UI: a sidebar **Layers list** (select / delete / reorder) + a **"+ Region"** button that enters a draw gesture (dashed marquee, no live preview, kaleidoscope appears on pointer-up).
- Sliders always operate on the currently active layer. No "commit" button is needed under the hood — commit just means "deselect and add a new layer"; bounds are read-only once the layer is created.
- URL hash serialises the full stack.
- `window.__kaleido` is versioned to the new shape (breaking change; CLAUDE.md updated in the same commit).

(See brainstorm — *Why This Approach* and *Key Decisions* sections — for the decision trail.)

## Technical Approach

### Architecture

#### Rendering pipeline

Current pipeline (`index.html:556-569`):

```
original texture (u_tex)
  → fragment shader (mode, p1, p2, rot, zoom, cx, cy)
  → default framebuffer (canvas)
```

New pipeline (N layers, ping-pong):

```
FBO_A, FBO_B : color textures sized to canvas.{width,height}, clamped to MAX_TEX

layer 0 (base) : render to FBO_A
  uniforms: source=original, region_kind=none
  samples u_tex only

for i in 1..N-1:
  src, dst = (FBO_A, FBO_B) if i odd else (FBO_B, FBO_A)
  render layer[i] to dst
  uniforms:
    u_prev_tex = src
    u_tex      = original image
    region     = shape/bounds/feather
    source     = original | prior | crop

final: blit last FBO to default framebuffer (trivial pass-through shader)
```

**Single shader program, many uniforms.** No separate compositor program. The fragment shader branches on `u_region_kind` to decide whether to do full-canvas (base) or masked (region) output, and on `u_source` to decide which sampler feeds the kaleidoscope math.

**Blit pass.** One extra tiny fragment shader: `gl_FragColor = texture2D(u_tex, uv)`. Keeps the `preserveDrawingBuffer=false` invariant (`CLAUDE.md` "Export path quirk") — export still uses `readPixels` on the default framebuffer after the blit.

#### Fragment shader extensions

Build on the existing shader (`index.html:332-438`). Add uniforms:

```glsl
uniform sampler2D u_prev_tex;   // prior composite (unused when region_kind==0)
uniform int       u_region_kind; // 0 base, 1 rect, 2 ellipse
uniform vec4      u_region;      // x, y, w, h in canvas UV (0..1)
uniform float     u_feather;     // 0..1, fraction of region's min half-extent
uniform int       u_source;      // 0 original, 1 prior, 2 crop
```

New helper: compute mask `m ∈ [0,1]` as a feathered signed-distance-field to the region boundary. Rect SDF = `max(abs(p - centre) - halfExtent, 0.0)`. Ellipse SDF = `length((p - centre) / halfExtent) - 1.0`, scaled.

Sampling logic:

```glsl
vec4 inside;
if (u_source == 0) {
  inside = kaleidoscope(u_tex, uv_canvas_to_image, params);        // original image
} else if (u_source == 1) {
  inside = kaleidoscope(u_prev_tex, uv_canvas_to_canvas, params);  // prior composite
} else {
  // crop: treat the region as the entire "virtual canvas" and "virtual texture"
  vec2 region_uv = (uv_canvas - u_region.xy) / u_region.zw;        // 0..1 within region
  inside = kaleidoscopeCropped(u_prev_tex, region_uv, u_region, params);
}

vec4 outside = texture2D(u_prev_tex, uv);
gl_FragColor = (u_region_kind == 0) ? inside : mix(outside, inside, m);
```

**Coordinate-system note.** Today `cx`/`cy` index the source texture (`index.html:360-361`: `ctr = (cx/100) * u_tsz`). That interpretation is preserved for `source=original`. For `source=prior`, `cx`/`cy` are percentages of the **canvas** (the prior composite sits on the canvas grid). For `source=crop`, `cx`/`cy` are percentages of the **region** (region-local). All three cases collapse to the same math once we pass in the correct effective `u_tsz` and `ctr` per layer on the JS side — no shader branching on coordinate convention needed.

#### State model

Replace the flat state (`index.html:526`) with:

```js
state = {
  activeLayerIndex: 0,
  layers: [
    { kind: 'base',
      mode: 0, p1: 8, p2: 1.00, rot: 0, zoom: 1.00, cx: 50, cy: 50 },
    // region layers appended here
  ]
}
```

Region layer shape:

```js
{
  kind: 'region',
  shape: 'rect' | 'ellipse',
  bounds: { x, y, w, h },        // 0..1 in canvas UV, axis-aligned
  source: 'original' | 'prior' | 'crop',
  feather: 0,                    // 0..1
  mode, p1, p2, rot, zoom, cx, cy
}
```

Invariants:
- `layers[0].kind === 'base'` always. Cannot be deleted or moved.
- `activeLayerIndex ∈ [0, layers.length)`.
- For `source='crop'`, `cx`/`cy` are region-local percentages.

#### SLIDERS + DOM ↔ state sync

`SLIDERS` (`index.html:510-517`) keeps its shape; `syncStateFromDom` / `syncDomFromState` (`index.html:528-539`) change to operate on `state.layers[state.activeLayerIndex]`:

```js
function activeLayer() { return state.layers[state.activeLayerIndex]; }
function syncStateFromDom() { const L = activeLayer(); for (const k of SLIDER_KEYS) L[k] = SLIDERS[k].el.value/SLIDERS[k].scale; }
function syncDomFromState() { const L = activeLayer(); for (const k of SLIDER_KEYS) SLIDERS[k].el.value = Math.round(L[k]*SLIDERS[k].scale); applyModeUI(L.mode); }
```

Sliders always reflect and mutate the active layer. Switching the active layer calls `syncDomFromState()` to reset slider positions.

#### Region drawing gesture

New gesture mode: `'draw-region'`. Added alongside `dragMode ∈ {'center', 'rotate', null}` (`index.html:679-680`).

State machine triggered by `+ Region` button:

```
idle → (click "+ Region") → picking-shape
  → (select shape) → armed
    → (pointerdown on canvas) → drawing
      → (pointermove) → update marquee overlay
      → (pointerup) → create region layer at bounds, set active, render, return to idle
    → (Escape / click Cancel) → return to idle
```

Marquee is a `<div class="marquee">` overlay on top of the stage, absolutely positioned, with `border: 1px dashed var(--accent)`. Cheaper than a WebGL preview pass and matches the brainstorm ("Dashed outline only").

Drop the region if `bounds.w < MIN_REGION_SIZE` or `bounds.h < MIN_REGION_SIZE` (suggest 2% of canvas). Clamp to canvas bounds.

#### Layers list UI

Extend the sidebar (`index.html:201-278`) with a new `.ctrl-group`:

```html
<div class="ctrl-group">
  <div class="ctrl-label">Layers</div>
  <div class="layers-list" data-kaleido="layers">
    <!-- one row per layer, top of stack first -->
  </div>
  <div class="layer-actions">
    <button data-kaleido="btnAddRegion">+ Region</button>
    <div class="shape-picker hidden">
      <button data-kaleido="btnShapeRect">Rect</button>
      <button data-kaleido="btnShapeEllipse">Ellipse</button>
      <button data-kaleido="btnCancelDraw">Cancel</button>
    </div>
  </div>
</div>
```

Each layer row shows: shape indicator ("⬛ Base" / "▭ Rect 1" / "⬭ Ellipse 2"), active indicator, delete button, reorder arrows. Clicking a row sets `activeLayerIndex`. `data-kaleido="layer-{idx}"` on each row for agent targeting.

Region-only controls (Source radio + Feather slider) live in a collapsible row inside the existing Controls group and are `.hidden` when the active layer is `base`.

#### Hash schema

The hash becomes a compact per-layer encoding. Design:

- Top-level: `a=<activeIdx>&L=<layer0>|<layer1>|...`
- Base layer: `b.m=0.p1=8.p2=1.rot=0.zoom=1.cx=50.cy=50`
- Region layer: `r.sh=rect.x=20.y=30.w=40.h=25.src=prior.f=0.m=1.p1=50.p2=1.rot=0.zoom=1.cx=50.cy=50`

Delimiter chars chosen to avoid URL-encoding: `|` between layers, `.` between fields, `=` for key/value. Legacy hash (no `L=`) migrates to a one-layer base-only stack.

Alternative: base64-encoded JSON. Rejected for v1 — unreadable, harder to hand-edit, CLAUDE.md values direct hackability.

(See brainstorm *Persistence & sharing* — full stack in the hash.)

#### FBO lifecycle

Two FBOs at canvas size, RGBA8, `LINEAR` filtering, `CLAMP_TO_EDGE`. Created on first render; reallocated whenever `canvas.width` or `canvas.height` changes (resize, export). Freed via `gl.deleteTexture` + `gl.deleteFramebuffer` before reallocation.

Size clamp: `min(canvas.{w,h}, MAX_TEX, MAX_VP_SIZE)`. If either FBO allocation fails, surface in the status bar and fall back to single-layer rendering of the base layer (rest of the stack silently dropped for that frame, with a user-visible warning).

**VRAM budget.** Two RGBA8 FBOs at 4096×4096 = 128 MB. Cap FBO size at `min(canvas.*, 4096)` by default; expose `CONFIG.maxFboSize` for tuning. Document in CLAUDE.md's *GPU safety* section.

#### Agent API (breaking)

`window.__kaleido` v2 shape:

```js
{
  apiVersion: 2,
  state: { activeLayerIndex, layers },    // getter returns deep copy
  setState(patch),                         // merges into state; validates shape
  addRegion({ shape, bounds, source?, feather? }),  // returns new layer index
  removeLayer(index),                      // index !== 0
  setActiveLayer(index),
  reorderLayer(from, to),
  loadImageFromUrl, reset, exportPNG, MODES, ready: true,
}
```

`kaleido:change` CustomEvent payload: `{ detail: { activeLayerIndex, layers } }`.

postMessage bridge (`index.html:967-992`) migrates to mirror the new API (messages: `setState`, `addRegion`, `removeLayer`, `setActiveLayer`, `reorderLayer`, `exportPNG`).

CLAUDE.md **Agent surface — keep stable** section is updated in the same commit to document v2.

#### Export

No change to the overall `exportPNG` shape (`index.html:767-811`). The only difference: the per-frame `render()` inside export now runs the full layer-stack pipeline. Since the blit pass writes to the default framebuffer, `readPixels` + Y-flip + 2D canvas + `toBlob` continue to work unchanged.

FBOs are reallocated at the export viewport size (with `MAX_VP_SIZE` clamp — inherited from today's export path). If the layered render at export size exceeds FBO budget, surface a status message: `"exported at <WxH> (clamped — <N> layers composited)"`.

### Implementation Phases

#### Phase 1 — FBO pipeline + shader refactor (foundation)

Deliverables:
- `createFBO(w, h)` / `deleteFBO(fbo)` / `resizeFBOs()` helpers, tracking two ping-pong FBOs.
- Blit shader program (trivial pass-through).
- Extend the main fragment shader with the new uniforms and branching (`u_prev_tex`, `u_region_kind`, `u_region`, `u_feather`, `u_source`).
- Rewrite `render()` to walk `state.layers`, ping-pong FBOs, and blit.
- All existing user flows (single base layer only) render pixel-identical to today.

Success criteria:
- Single-layer output visually and numerically matches current output (SSIM ≥ 0.99 against a reference capture for each of the 5 modes).
- No `preserveDrawingBuffer` regressions; export still works.
- FBO allocation failure is handled gracefully.

Estimated scope: the biggest single piece. ~150-250 LOC.

#### Phase 2 — State migration

Deliverables:
- Migrate `state` to `{activeLayerIndex, layers}`.
- `activeLayer()` helper; `syncStateFromDom`/`syncDomFromState` updated.
- `resetAll` creates a single-base-layer stack.
- `writeStateToHash`/`readStateFromHash` rewritten; legacy hash migrates to v2.
- `setState` validates layered patches.
- Update `kaleido:change` payload.

Success criteria:
- URL round-trip works for single-layer compositions.
- Legacy hash links still load correctly.
- Sliders still control the base layer as today.

#### Phase 3 — Region drawing gesture

Deliverables:
- "+ Region" button in sidebar; shape-picker (rect/ellipse); Cancel.
- Marquee overlay element + CSS.
- Pointer/touch handlers for the `'draw-region'` gesture mode.
- On pointer-up: create region layer with `source='prior'`, `feather=0`, default params copied from base; set active; render.
- Minimum-size threshold; bounds clamp; Escape cancels.

Success criteria:
- User can draw a rectangle region; new kaleidoscope appears inside, prior composite outside.
- Same with ellipse.
- Drawing on top of existing regions produces a visible stack.

#### Phase 4 — Layers list UI + reorder/delete

Deliverables:
- Layers list panel in sidebar with select/delete/reorder.
- Layer row renders shape icon and label.
- `data-kaleido="layer-{idx}"` on each row.
- Reorder triggers re-render and hash write.

Success criteria:
- Can add 5+ layers, select each, edit each, delete each.
- Reordering changes visible stacking.

#### Phase 5 — Per-layer source & feather controls

Deliverables:
- Source radio (Original / Prior / Crop), visible only when active layer is a region.
- Feather slider (0..100%), visible only when active layer is a region.
- Shader math for each source (incl. region-local `cx`/`cy` for `source='crop'`).
- Feathered edge rendering for both rect and ellipse.

Success criteria:
- Each source produces the expected visual result.
- Feather at 0 is indistinguishable from no feather; at high values edges are clearly softened.

#### Phase 6 — Agent API v2 + CLAUDE.md

Deliverables:
- `window.__kaleido` v2 shape.
- postMessage bridge migration.
- Update CLAUDE.md's *Agent surface* and *State contract* sections.
- Update README's scripting surface description if present.

Success criteria:
- `window.__kaleido.apiVersion === 2`.
- Scripted test: add a region, set its mode, remove it, export — all via `window.__kaleido`.

#### Phase 7 — Export + polish

Deliverables:
- Export runs the full stack at export resolution with FBO reallocation.
- Status-bar messages for clamp / layer count.
- Empty-state handling (no image → "+ Region" disabled).
- Keyboard: Escape cancels draw.
- Update `<!-- AGENT API -->` HTML comment in `index.html` to describe v2.

Success criteria:
- Multi-layer export produces the expected composite at the correct resolution.
- Export clamp message mirrors current single-layer behaviour.

## Alternative Approaches Considered

- **Single-pass mega-shader** with fixed layer uniforms. Rejected: GLSL ES 1.0 arrays are awkward; hard layer cap; `source=prior canvas` is hard to implement without a second texture sample of a not-yet-rendered thing. (See brainstorm *Approach B*.)
- **Destructive commits** — each committed layer becomes frozen pixels, so only a single FBO is needed. Rejected: user explicitly requested fully non-destructive params (brainstorm *Edit history*).
- **Freehand / polygon selection shapes.** Deferred — rectangle + ellipse covers the creative use cases; freehand adds significant UX and hash-encoding complexity.
- **Selection handles for post-commit bound editing.** Deferred — user chose "read-only bounds after commit" for simpler UX (brainstorm *Resolved Questions*).

## System-Wide Impact

### Interaction Graph

- `+ Region` → `enterDrawMode(shape)` → pointerdown on canvas → marquee updates on pointermove → pointerup → `addRegion(bounds, shape)` → mutates `state.layers` → `syncDomFromState()` → `scheduleRender()` → full-stack render via FBO ping-pong → `writeStateToHash()` → `kaleido:change` event.
- Slider input → `scheduleRender()` → `syncStateFromDom()` (writes to active layer) → full-stack render → hash write → event.
- Layer row click → `setActiveLayer(i)` → `syncDomFromState()` → re-enables region controls if applicable → hash write → event.
- Load image → replaces `u_tex` → `scheduleRender()` re-renders entire stack against new source (layers persist).

### Error & Failure Propagation

- FBO allocation fails → `render()` falls back to single-layer base render + sets status-bar warning. No stack trace surfaces.
- Malformed hash → `readStateFromHash` returns a default single-base-layer stack; legacy hash migrates cleanly.
- Invalid `setState` patch → rejected silently with `console.warn`, state unchanged; matches today's `setState` permissiveness (`index.html:937-964`).
- Export viewport clamp → status message, existing pattern preserved.
- Draw gesture with no image loaded → "+ Region" button disabled; no error path.

### State Lifecycle Risks

- **Orphaned FBO textures on resize.** Guard: `resizeFBOs()` always frees before reallocating.
- **activeLayerIndex out of bounds after delete.** Guard: clamp to `[0, layers.length-1]` on every mutation.
- **Layer references stale after reorder.** No layer IDs — layers are addressed by index. Agents must re-fetch state after mutations.
- **Partial state during migration.** Hash migration is atomic (parse-and-replace, not incremental). No partial state.
- **Image replace + layers mismatch.** Layers reference no image data directly — only `source='original'` samples `u_tex`. Replacing `u_tex` re-renders correctly. `source='prior'` and `source='crop'` sample the prior composite, which is regenerated each render.

### API Surface Parity

All four agent surfaces migrate together to v2:

1. `window.__kaleido.state` getter + `setState` — new shape.
2. postMessage bridge (`index.html:967-992`) — messages mirror new methods.
3. `kaleido:change` CustomEvent — payload is `{activeLayerIndex, layers}`.
4. `data-kaleido="…"` selectors — new attributes for layer rows and region controls; existing attributes on sliders/buttons preserved.

The `<!-- AGENT API -->` HTML comment at the top of the `<script>` block is the canonical reference (`CLAUDE.md` "When in doubt") — update in the same commit.

### Integration Test Scenarios

1. **Stack round-trip.** Create base + 3 regions; copy URL; open in new tab; verify identical composite (pixel-diff or SSIM).
2. **Image swap with stack.** Add 2 region layers; load new image; verify base + regions re-render against the new source.
3. **Mid-stack edit.** With 3 regions, set active to the middle one, change its mode and feather; verify only that layer (and whatever reads its composite above) changes.
4. **Export at full fidelity.** Add 5 layers; export; verify rendered PNG matches an in-app screenshot (upscaled) within tolerance.
5. **Legacy hash migration.** Load a pre-v2 hash URL (`#m=1&p1=50&…`); verify a single base layer is produced with correct params.
6. **Agent script end-to-end.** Via `window.__kaleido`: `addRegion({shape:'rect', bounds:{x:0.2,y:0.2,w:0.4,h:0.4}, source:'prior'})` → `setState({layers: [...]})` → `exportPNG()`. Verify each step fires `kaleido:change` with correct payload.

## Acceptance Criteria

### Functional Requirements

- [x] Base layer renders pixel-identical to current output for all 5 modes.
- [x] "+ Region" button opens a shape picker; selecting rect or ellipse enables click-drag on canvas.
- [x] During drag, a dashed marquee tracks the pointer; no kaleidoscope preview inside until pointer-up.
- [x] On pointer-up, a new region layer is created with `source='prior'`, `feather=0`, copying base layer's params as defaults, and becomes active.
- [x] Sliders reflect and mutate the active layer's params.
- [x] Selecting a layer in the Layers list makes it active and updates sliders.
- [x] Each layer (except base) can be deleted.
- [x] Layer order is reorderable; visible stacking updates.
- [x] Source picker appears only for region layers; `original` / `prior` / `crop` each produce the expected visual.
- [x] Feather slider softens edges for rect and ellipse regions.
- [x] Bounds are read-only after commit (no resize handles).
- [x] URL hash encodes the full stack and round-trips.
- [x] Legacy hash URLs still load (migrated to single base layer).
- [x] Export at full resolution runs the full stack.
- [x] `window.__kaleido` v2 API exposes `state`, `setState`, `addRegion`, `removeLayer`, `setActiveLayer`, `reorderLayer`, `loadImageFromUrl`, `reset`, `exportPNG`, `MODES`, `apiVersion: 2`, `ready: true`.
- [x] `kaleido:change` fires with new payload.

### Non-Functional Requirements

- [x] Single-layer render performance within 10% of current single-pass render on typical hardware.
- [x] Multi-layer (up to 8 layers) render stays ≥30 fps on 2× DPR displays up to 2560×1600 canvas.
- [x] FBOs clamped to `min(canvas.*, MAX_TEX, 4096)`; allocation failure handled with user-visible fallback.
- [x] No `preserveDrawingBuffer` flag introduced.
- [x] No new runtime dependencies.
- [x] CSP unchanged (or document any update).
- [x] Still a single file: all CSS, GLSL, HTML, JS in `index.html`.

### Quality Gates

- [x] CLAUDE.md updated (state contract, agent surface, GPU safety notes).
- [x] `<!-- AGENT API -->` comment updated.
- [x] README updated with layered-regional feature blurb.
- [x] Manual smoke checklist in the PR covers: add/delete/reorder, each source, feather, export, share-URL round-trip, legacy-URL migration.

## Success Metrics

- User can create a multi-layer composite in under a minute from a fresh page.
- Shared URLs of 3-layer compositions fit in under 500 chars.
- Export of a 4-layer composite at 4K resolution succeeds on a mid-range laptop.

## Dependencies & Prerequisites

- None external.
- Depends on the existing WebGL 1 context at `index.html:440-488`. WebGL 2 is not required but the FBO patterns translate if the context is upgraded later.
- Sits on top of the hardening patterns in `docs/solutions/performance-issues/webgl-single-file-spa-hardening-patterns.md` — do not violate them.

## Risk Analysis & Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| FBO allocation OOM on mobile or large displays | Rendering fails | `CONFIG.maxFboSize` (default 4096) cap; fallback to single-layer base render with warning |
| Shader branch cost on `u_source` / `u_region_kind` | Perf regression | Branches are on uniforms, not per-pixel data — negligible cost on modern drivers; profile in Phase 1 |
| Hash URL length with many layers | Shared URLs truncated by chat apps | Acceptable for v1 per brainstorm; flag in follow-up todo if users hit it |
| Agent API breakage for external consumers | MCP tools / iframe embeds break | Versioned to v2; old v1 callers get an upfront `console.error` pointing to migration notes; document in CLAUDE.md same commit |
| Crop-source sampling outside region | Visual glitches near edges | Shader `clamp()` the crop UV to `[0,1]` before sampling; `CLAMP_TO_EDGE` on FBO textures reinforces |
| Reorder invalidates `source='prior'` chain | Layer depending on prior shows unexpected result | This is user-visible-by-design; no mitigation needed (feature, not bug) |
| Double render on image load (stack re-runs) | Brief flicker | Already masked by `hashWriteSuppressed` pattern at `index.html:828`; reuse |
| Export memory for N layers at 8K × 2 FBOs | Browser tab crash | Reuse the existing `MAX_VP_SIZE` clamp; enforce the same cap on FBOs during export |

## Resource Requirements

- One engineer, sequential phases. Each phase is independently shippable behind the state shape change, but it's simpler to ship all phases together since the state migration is breaking.
- No infrastructure, no new services, no new dependencies.

## Future Considerations

Out of scope for this plan; track as follow-ups:

- **Edit bounds after commit.** Drag-handles on the active layer's region.
- **Freehand / polygon shapes.** Brush or click-to-add-vertex mask.
- **Named layers / visibility toggle.** For long stacks.
- **Per-layer opacity / blend modes** beyond inside-outside masking.
- **Thumbnail previews** in the Layers list (render each layer to a small offscreen FBO).
- **Compressed hash encoding.** Base36 or domain-specific compression if hash lengths become painful.
- **Max-layer cap.** Monitor usage; add a soft cap (e.g. 16) if needed.
- **Rotatable regions.** Currently axis-aligned only.

## Documentation Plan

- **CLAUDE.md** — rewrite *State contract* and *Agent surface* sections for v2. Add a *Rendering pipeline* subsection under *GPU safety* to document the FBO ping-pong invariants.
- **`<!-- AGENT API -->` HTML comment** — v2 schema.
- **README.md** — add the new feature to the scripting/sharing surface section; show an example `addRegion` call.
- **Follow-up todo files** in `todos/` for each item in *Future Considerations*.

## Sources & References

### Origin

- **Brainstorm document:** [docs/brainstorms/2026-04-24-regional-selection-brainstorm.md](../brainstorms/2026-04-24-regional-selection-brainstorm.md). Key decisions carried forward:
  - Layered non-destructive model with full-stack editability (params only; bounds frozen on commit).
  - Rectangle + ellipse shapes, dashed-marquee-only draw feedback.
  - Source choices: `original` / `prior` / `crop`, default `prior`, selectable per layer.
  - Versioned (breaking) agent API: `state = {layers, activeLayerIndex}`; CLAUDE.md update in same commit.

### Internal References

- Shader & render loop: `index.html:327-438` (shaders), `index.html:556-569` (render), `index.html:573-585` (scheduleRender).
- State & sliders: `index.html:499-523` (MODES, SLIDERS), `index.html:526` (state), `index.html:528-547` (sync + applyModeUI).
- Gesture handlers: `index.html:675-742`.
- Export: `index.html:767-811`.
- Hash round-trip: `index.html:828-859`.
- Agent API: `index.html:937-992`.
- CONFIG + GPU limits: `index.html:317-324`, `index.html:486-488`.
- Design contracts: `CLAUDE.md` (all sections).

### External References

- WebGL 1 FBO pattern (canonical): ping-pong framebuffers for multi-pass effects. No external library needed.
- `docs/solutions/performance-issues/webgl-single-file-spa-hardening-patterns.md` — the six patterns this codebase already applies; all six constrain this feature.

### Related Work

- Prior feature on this repo: `docs/plans/2026-04-24-001-feat-shift-drag-rotation-plan.md` — gesture state machine pattern to mirror.
- Existing todos (`todos/003`, `todos/005`, `todos/007`) — resolve `005` shader cleanup before Phase 1 if convenient; `003` and `007` patterns are preserved.
