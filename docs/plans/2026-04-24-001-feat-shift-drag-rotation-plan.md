---
title: Shift+drag canvas gesture for rotation
type: feat
status: completed
date: 2026-04-24
---

# Shift+drag canvas gesture for rotation

## Overview

Add a mouse gesture that rotates the kaleidoscope: holding **Shift** at
mousedown turns the existing canvas drag into a rotation drag. Horizontal
cursor movement maps to `state.rot`; a full canvas-width sweep = 360°.
Normal (no-modifier) drag still sets `state.cx`/`state.cy` exactly as it
does today. Implemented entirely inside `index.html` with no new
dependencies, no new state fields, and no changes to the public
`window.__kaleido` surface.

## Problem statement / motivation

Rotation currently only has two input paths:

- The `rot` slider in the side panel (`#rot`, `data-kaleido="slider-rot"`).
- Programmatic `window.__kaleido.setState({ rot })`.

Users exploring the canvas spend most of their time hovering over the
preview (where drag already adjusts center). Reaching back to the sidebar
slider to rotate breaks the flow. A canvas gesture keeps the user "in"
the preview. Shift-as-modifier follows convention (Illustrator, Figma,
Photoshop, Blender) without introducing any new UI chrome.

## Proposed solution

At mousedown, lock the gesture mode based on `event.shiftKey`:

- `shiftKey === false` → existing **center** gesture (unchanged).
- `shiftKey === true`  → new **rotate** gesture.

Rotate gesture is **relative** (delta-based), not absolute:

- Anchor: `{ startX: event.clientX, startRot: <current rot in degrees> }`.
- On move: `newRot = startRot + (clientX − startX) / canvas.clientWidth × 360`.
- Wrap into `[0, 360)` before writing to the DOM slider.
- Push through the existing render pipeline: write
  `SLIDERS.rot.el.value`, call `scheduleRender()`. Labels, hash, and
  `kaleido:change` events flow through unchanged.

Visual affordance: while Shift is held over the stage, swap the canvas
cursor from `crosshair` to `ew-resize` so the gesture is discoverable
without reading the hint line. Update the existing hint under the
sliders to mention the gesture. Add one line to the `<!-- AGENT API -->`
comment block so agents reading the source learn the binding.

Touch path is out of scope for v1: touch has no Shift modifier, so touch
drag continues to only set center. A future two-finger twist is left as
a follow-up.

## Technical approach

### File scope

All changes in `index.html`. Per `CLAUDE.md`, no new files, no build
step, no external resources. This feature is pure client-side input
handling — it does not touch GLSL, GPU limits, `texImage2D`, `readPixels`,
or any of the clamped paths listed in CLAUDE.md § "GPU safety".

### State contract

No new fields on `state`. `state.rot` already exists (degrees, 0–360,
0.1° resolution, slider scale `10`). The gesture writes through the
DOM-as-source pattern already used by `setCenterFromClient`
(`index.html:674-681`): it sets `SLIDERS.rot.el.value` and calls
`scheduleRender()`, which runs `syncStateFromDom` → `updateLabels` →
`render` → `writeStateToHash` → dispatch `kaleido:change`
(`index.html:570-581`).

### Gesture state

Replace the current `let dragging = false` (`index.html:672`) with a
small gesture-mode record. Sketch:

```js
// Was:
//   let dragging = false;
let dragMode = null;          // 'center' | 'rotate' | null
let rotAnchor = null;         // { startX, startRot }

function beginDrag(e) {
  if (e.shiftKey) {
    dragMode = 'rotate';
    rotAnchor = {
      startX: e.clientX,
      startRot: +SLIDERS.rot.el.value / SLIDERS.rot.scale,
    };
  } else {
    dragMode = 'center';
    setCenterFromClient(e.clientX, e.clientY);
  }
}

function continueDrag(e) {
  if (dragMode === 'center') {
    setCenterFromClient(e.clientX, e.clientY);
  } else if (dragMode === 'rotate') {
    const dx = e.clientX - rotAnchor.startX;
    const width = canvas.clientWidth || 1;
    setRotation(rotAnchor.startRot + (dx / width) * 360);
  }
}

function setRotation(deg) {
  const wrapped = ((deg % 360) + 360) % 360;   // handle negatives
  SLIDERS.rot.el.value = Math.round(wrapped * SLIDERS.rot.scale);
  scheduleRender();
}

function endDrag() {
  dragMode = null;
  rotAnchor = null;
}

canvas.addEventListener('mousedown', beginDrag);
window.addEventListener('mousemove', e => { if (dragMode) continueDrag(e); });
window.addEventListener('mouseup',   endDrag);
window.addEventListener('blur',      endDrag);  // defensive: drop gesture if window loses focus mid-drag
```

Touch handlers keep their current shape (center-only). Replace the
`dragging` truthy checks with `dragMode === 'center'` on the touch
branch, or just leave touch on its own boolean — either is fine. The
plan uses `dragMode`:

```js
canvas.addEventListener('touchstart', e => {
  e.preventDefault();
  dragMode = 'center';
  setCenterFromClient(e.touches[0].clientX, e.touches[0].clientY);
}, { passive: false });
canvas.addEventListener('touchmove', e => {
  e.preventDefault();
  if (dragMode === 'center') setCenterFromClient(e.touches[0].clientX, e.touches[0].clientY);
}, { passive: false });
canvas.addEventListener('touchend', endDrag);
```

### Cursor affordance

Add a keydown/keyup pair on `window` that toggles a `rotate-ready`
class on the `<main id="stage">` element. Scoping to Shift-only
keydown avoids firing on every keystroke:

```js
window.addEventListener('keydown', e => {
  if (e.key === 'Shift') stage.classList.add('rotate-ready');
});
window.addEventListener('keyup', e => {
  if (e.key === 'Shift') stage.classList.remove('rotate-ready');
});
window.addEventListener('blur', () => stage.classList.remove('rotate-ready'));
```

CSS (extend the existing `#c` block around `index.html:167-170`):

```css
#c { display: block; cursor: crosshair; }
main.rotate-ready #c { cursor: ew-resize; }
main.rotate-ready.dragging #c { cursor: ew-resize; }  /* optional polish */
```

No `.dragging` class is added in the MVP — cursor stays `ew-resize`
while shift is held and during the drag (because shift remains held).
If the user releases shift mid-drag the cursor flips back to
`crosshair`; the gesture itself stays locked to rotation. That's the
tradeoff of "lock at mousedown" and matches app conventions.

### Wrap-around semantics

`state.rot` is already a circular quantity but the slider bounds are
`[0, 360°]` with no wrap. Current behavior clamps at the ends. Because
rotation is visually circular, we wrap to `[0, 360)` *before writing
the slider* during gesture input only. The slider itself keeps its
existing min/max so manual slider drag behaves as today.

Hash serialization is unchanged: `writeStateToHash` already writes
`state.rot.toFixed(1)`. After wrap, values never exceed 360 via the
gesture, which is a small improvement.

### Agent surface

No change to the `window.__kaleido` shape. No change to
`data-kaleido="…"` attributes. No change to `kaleido:change` payload
shape (it already includes `rot`). The `<!-- AGENT API -->` HTML
comment block at the top of `<script>` gets one additional line under
the existing documentation, e.g.:

```
Canvas input: drag → sets cx/cy; shift+drag → rotates (horizontal
sweep across canvas width = 360°). Touch drag always sets cx/cy.
```

### Hint text

Change `index.html:275` from:

```html
<div class="hint">Click or drag the preview to set center</div>
```

to:

```html
<div class="hint">Drag to set center · Shift+drag to rotate</div>
```

## Alternative approaches considered

- **Tangential "twist" around the current center.** Feels most
  kaleidoscope-native — user drags a point around the center and it
  rotates with them. Rejected for v1 because (a) it amplifies tiny
  motions near the center unless we add a min-radius guard, (b) the
  anchor angle depends on where the user clicked, which is less
  predictable than horizontal-delta, and (c) horizontal-delta is what
  most design tools do. Could be a follow-up without breaking the
  current gesture.
- **Vertical drag for rotation.** Rejected — horizontal "sweep to
  rotate" is the Blender/Figma convention; vertical feels like "tilt."
- **Mouse wheel for rotation.** Genuinely nice but wheel-on-canvas
  fights page scroll behavior on some setups; shift+drag has zero
  scroll-interaction risk. Wheel-zoom/rotate could layer on later.
- **Live gesture switching (detect shift during move, not mousedown).**
  Rejected — requires re-anchoring the rotation mid-drag whenever shift
  flips, and gives the user two gestures that feel mushy. Locking at
  mousedown matches how other tools handle modifier-gated modes.
- **Extending rot slider to ±∞ instead of wrapping.** Rejected — the
  product stores rotation as a visual angle, so `[0, 360)` modulo is
  the right mental model. URL hashes also stay shorter.

## System-wide impact

- **Interaction graph.** Shift+drag writes `SLIDERS.rot.el.value` →
  `scheduleRender()` (`index.html:570`) → rAF tick → `syncStateFromDom`
  (`524`) → `updateLabels` (`545`) → `render` (`552`) →
  `writeStateToHash` (`777`) → `document.dispatchEvent('kaleido:change')`
  (`579`). This is the exact same chain used by every slider today, so
  consumers of `kaleido:change` see no new event shape.
- **Error propagation.** No new async paths and no new error surfaces.
  Gesture handlers swallow nothing — they just compute numbers and
  write to a slider.
- **State lifecycle risks.** The anchor object (`rotAnchor`) only
  exists between mousedown and mouseup. `blur` listener clears it
  defensively so an orphaned anchor can't carry over to the next
  drag. `dragMode` is a single variable; no persisted state.
- **API surface parity.** `window.__kaleido.setState({ rot })`, the
  `#rot` slider, the URL hash `rot=` parameter, and the new gesture all
  converge on the same path (DOM slider → `scheduleRender`). No
  divergent code path to keep in sync.
- **Integration test scenarios.** See "Acceptance criteria" below —
  they're written as manual-test steps because the project has no test
  framework (per `CLAUDE.md`).

## Acceptance criteria

### Functional

- [x] Shift+mousedown on canvas begins rotation mode. Mousemove
      translates `clientX` delta into `state.rot`: full `canvas.clientWidth`
      sweep = 360°.
- [x] Non-shift mousedown on canvas sets center, exactly as today. No
      regression in any of the five kaleidoscope modes.
- [x] Rotation wraps so the gesture never hits a dead stop; values
      always land in `[0, 360)` after wrap.
- [x] Rotation slider's numeric readout (`data-kaleido="val-rot"`)
      updates in real time during shift+drag.
- [x] `kaleido:change` CustomEvent fires during shift+drag with the
      updated `rot` field in `detail`.
- [x] URL hash (`#…&rot=…`) reflects the new rotation after each drag
      tick, same as sliders today.
- [x] Releasing Shift *during* a rotation drag keeps the gesture in
      rotation mode until mouseup (gesture locks at mousedown).
- [x] Pressing Shift *during* a center drag does **not** switch the
      gesture to rotation (same rationale).
- [x] Cursor is `ew-resize` while Shift is held over the stage, and
      reverts to `crosshair` on Shift release or window blur.
- [x] Window blur mid-drag ends the gesture cleanly (`dragMode` and
      `rotAnchor` reset).
- [x] Touch drag behavior is unchanged: always sets center, never
      rotates.
- [x] Hint text under the sliders mentions shift+drag.
- [x] `<!-- AGENT API -->` comment lists the gesture.

### Non-functional

- [x] No new dependencies, no build step, no external imports
      (`CLAUDE.md` § "Design constraints").
- [x] No changes to `window.__kaleido` shape, `data-kaleido` attributes,
      or `kaleido:change` detail shape.
- [x] No calls to `gl.texImage2D` or mutations of `canvas.width`/
      `canvas.height` introduced (`CLAUDE.md` § "GPU safety").
- [x] `preserveDrawingBuffer` is not touched (`CLAUDE.md` § "Export path
      quirk").
- [x] CSP meta tag is unchanged (no new `connect-src`/`script-src`
      needs).
- [x] No new globals beyond the gesture-scoped `dragMode` / `rotAnchor`
      and the helper functions (`beginDrag`, `continueDrag`, `endDrag`,
      `setRotationDeg`), which match the existing top-level-function
      pattern used throughout the file.

### Manual test plan

All tests run against `index.html` opened directly in a browser (no
server required per CLAUDE.md).

1. Load the `Colorful pattern` sample.
2. Drag on canvas without Shift: center follows cursor. Rotation
   slider does **not** move.
3. Hold Shift, hover over canvas: cursor becomes `ew-resize`. Release
   Shift: cursor returns to `crosshair`.
4. Shift+drag right across the full canvas width: rotation slider
   sweeps ~360° and wraps cleanly through 0/360.
5. Shift+drag left past 0°: rotation wraps to ~359° and continues
   decreasing.
6. During a rotation drag, release Shift while still holding the
   mouse: gesture stays on rotation. Release mouse: gesture ends.
7. During a center drag, press Shift mid-drag: center continues to
   follow cursor (no mode switch).
8. Open the page in a second tab, copy the URL after shift+dragging,
   paste into the first tab's reload: the shared rotation loads.
9. Observe `document.addEventListener('kaleido:change', e =>
   console.log(e.detail.rot))` fires on each rAF tick during drag.
10. Repeat steps 4–6 in each of the five `mode` values to confirm no
    mode-specific regression.
11. Touch the canvas on a touch device (or DevTools touch emulation):
    single-finger drag sets center, never rotates.
12. `window.__kaleido.setState({ rot: 123 })` still works and the
    slider moves to match.

## Success metrics

This is a UX polish feature, not an instrumented one. Success is
qualitative:

- Rotation adjustment feels reachable without moving to the sidebar.
- No user-visible regressions in drag-to-center behavior or other
  sliders.
- Agents and iframe consumers observe no behavioral change beyond
  the new `kaleido:change` events that result from gesture input —
  which are shaped identically to slider-driven events.

## Dependencies & risks

- **No external dependencies.** Change is one file.
- **Risk: window-blur-mid-drag orphaning.** Pre-existing risk for
  center drag too (see `index.html:685` — no blur handler today). This
  plan adds a `blur` handler that clears `dragMode`, incidentally
  fixing the pre-existing edge case for both gestures. Flag, low-risk.
- **Risk: Shift+drag for marquee/text-select** is not relevant here —
  the canvas has no text selection; mousedown on canvas is not a
  browser-native shift gesture.
- **Risk: Shift+click+release (no drag).** Starts rotation mode with
  zero delta; mouseup ends it. Net effect: no-op, identical to
  current non-shift click (which sets center to itself). For shift
  case, center is *not* set, which is the intended behavior.
- **Risk: Non-US keyboards where Shift is remapped.** `e.shiftKey` is
  standardized across layouts via `KeyboardEvent.shiftKey`; low risk.

## Open questions

- **Sensitivity curve.** Plan defaults to "full canvas width = 360°."
  Alternatives: 720° (half sweep = full turn, faster) or a fixed
  pixels-per-degree value independent of canvas size. Recommend
  starting with canvas-width-= -360°; tune based on feel.
- **Should the gesture update the URL hash mid-drag, or only at
  mouseup?** Plan uses mid-drag via `scheduleRender`, which matches
  slider behavior. If mid-drag hash thrash is noisy in browser
  history, consider debouncing `history.replaceState` for *all*
  sliders — would be a separate change, not part of this feature.
- **Should touch get a two-finger twist now or later?** Plan says
  later. If the user wants touch parity in v1, that's a larger
  scope and should be explicit.

## Implementation order

1. Add `dragMode`/`rotAnchor` state and `beginDrag`/`continueDrag`/
   `endDrag`/`setRotation` helpers (`index.html` drag section around
   line 671–689).
2. Rewire mouse handlers to the new helpers; rewire touch handlers to
   use `dragMode === 'center'`.
3. Add `keydown`/`keyup`/`blur` listeners that toggle
   `main.rotate-ready`.
4. Add `main.rotate-ready #c { cursor: ew-resize; }` to the style
   block near the existing `#c` rule.
5. Update hint text at `index.html:275`.
6. Add one line to the `<!-- AGENT API -->` comment block.
7. Manual test plan above; spot-check each of the five modes.
8. Commit.

## Sources & references

### Internal references

- `index.html:522` — `state` object; `state.rot` already exists.
- `index.html:506-513` — `SLIDERS` table, confirms `rot.scale = 10`.
- `index.html:570-581` — `scheduleRender` pipeline (authoritative path
  for all gesture writes).
- `index.html:672-689` — current drag implementation (the block being
  refactored).
- `index.html:674-681` — `setCenterFromClient`, the model being
  mirrored for rotation.
- `index.html:292-309` — `<!-- AGENT API -->` comment to extend.
- `index.html:777-788` — `writeStateToHash`, already serializes `rot`.
- `CLAUDE.md` — constraints that shaped this plan (single file, no
  build, state contract, agent surface, GPU safety, export path,
  CSP).
- `docs/solutions/performance-issues/webgl-single-file-spa-hardening-patterns.md`
  — referenced in CLAUDE.md; no patterns in it are violated by this
  change (new code goes through `scheduleRender` and does not touch
  GPU paths).

### External references

- MDN `KeyboardEvent.shiftKey` — layout-independent modifier flag.
  <https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/shiftKey>
- MDN `Element.getBoundingClientRect` — underpins cursor→canvas math
  (already used in `setCenterFromClient`).
  <https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect>
