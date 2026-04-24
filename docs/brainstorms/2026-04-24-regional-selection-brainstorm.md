---
date: 2026-04-24
topic: Regional selection — layered kaleidoscope over user-drawn regions
branch: feat/regional-selection
status: brainstormed
---

# Regional Selection — Layered Kaleidoscope

## What We're Building

A non-destructive layered kaleidoscope model. The user starts with the existing full-canvas kaleidoscope (the "base" layer). They can then add rectangular or elliptical **region layers** on top. Each region layer renders a kaleidoscope inside its bounds while the backdrop — whatever was composited beneath it — shows through outside its bounds.

Because layers are non-destructive, every layer stays editable forever: selecting a layer in a sidebar list restores its sliders, and any tweak re-renders the whole stack.

## Why This Approach

- The user's mental model is "paint another kaleidoscope on top of what I already made" — a layered, reveal-through-mask operation, not a crop.
- Rectangle + ellipse covers common creative uses (framed windows, vignetted spots) without the UX cost of freehand selection or polygon editing.
- An explicit "+ Region" button plus Commit/Cancel makes the modal boundary obvious; reusing the existing sliders for the active layer avoids doubling the control surface.
- Ping-pong FBOs are a natural fit for the stack semantics (each layer samples the previous composite) and keep rendering pure w.r.t. layer order.
- Keeping the full stack in the URL hash preserves the CLAUDE.md contract that "if the feature is visible in the product, it should be representable in the URL hash."

## Key Decisions

### Semantics
- **Region defines tile bounds.** Inside a region, a new kaleidoscope renders with its own params. Outside, the backdrop (whatever was composited beneath) shows through — **not the original image**.
- **Source image for a region is a per-layer choice**, picked when the region is created: (a) original uploaded image, (b) frozen prior canvas, (c) pixels inside the region only.
- **Edge style is hard by default, with an optional per-layer feather slider** (0 = hard; higher = soft falloff).

### Shapes
- Rectangle and ellipse. Both axis-aligned for v1.
- Drawn by click-drag on the canvas after entering region-draw mode.

### Layer model
- State grows a `layers` array (layer 0 is the base full-canvas kaleidoscope; layers 1..N are regions) plus an `activeLayerIndex`.
- Each layer stores: `kind` (`base` | `region`), `shape` (`rect` | `ellipse`), `bounds` (for regions), `source` (`original` | `prior` | `crop`, default `prior`), `feather`, and the existing `{mode, p1, p2, rot, zoom, cx, cy}`.
- **Non-destructive kaleidoscope params.** Mode, sliders, feather, and source choice stay editable for every layer forever.
- **Bounds are read-only after commit.** To reshape a region, the user deletes it and adds a new one. Keeps the drawing UX simple (no handles).
- Stack depth is arbitrary (practical cap tied to FBO / perf — decided in plan phase).

### UX flow
- Click **"+ Region"** → freezes the current composite as a backdrop preview → user picks shape (rect/ellipse) → click-drag on canvas defines bounds; **only a dashed marquee is shown while dragging** (no live kaleidoscope preview inside the band) → on mouse-up, source defaults to `prior` and the kaleidoscope renders inside → user tunes with existing sliders (now acting on the new layer) → **Commit** adds it to the stack, **Cancel** discards.
- Sidebar grows a **Layers list**: select a layer to make it active (sliders reflect its params), delete, and reorder.
- Clicking an empty area of the canvas (no active layer) continues to behave as today — sets the base layer's center.

### Coordinate conventions
- For `source = original` or `source = prior`, `cx` / `cy` are percentages of the full image/canvas (unchanged from today).
- For `source = crop` (pixels inside the region only), `cx` / `cy` are **region-local** — `50/50` means the center of the region, not of the image. Matches the "fold this patch into itself" mental model.

### Persistence & sharing
- URL hash encodes the full layer stack (shape, bounds, source, params, feather per layer). Hash gets longer with deep stacks; accept that for now.
- Export runs the same stack at full export resolution.

### Agent API (breaking change)
- The flat `state = {mode, p1, p2, rot, zoom, cx, cy}` is replaced by a layered shape: `{layers, activeLayerIndex}`. `setState` takes the new shape.
- New methods expected: `addRegion(shape, bounds, source?)`, `removeLayer(i)`, `setActiveLayer(i)`.
- CLAUDE.md is updated in the same change to document the new contract.
- `kaleido:change` event payload becomes `{layers, activeLayerIndex}`.

## Resolved Questions

1. **Region editability after commit** → **Read-only bounds after commit.** Params (mode/sliders/feather/source) stay editable; to reshape a region, delete and re-add.
2. **Agent API compatibility** → **Versioned, breaking change.** New shape is `{layers, activeLayerIndex}`. CLAUDE.md updates in the same change.
3. **Default source for a new region** → **Frozen prior canvas** (`source = prior`). User can change per-layer.
4. **Visible feedback while drawing** → **Dashed marquee only.** Kaleidoscope appears on mouse-up.
5. **`source = crop` coordinate semantics** → **Region-local.** `cx`/`cy` are percentages of the region, not the image.
6. **`kaleido:change` payload** → Emit the full `{layers, activeLayerIndex}`.

## Open Questions (deferred to plan phase)

1. **Max layer depth.** Practical cap is tied to FBO memory and per-frame render cost; decide during plan phase once the render pipeline is sketched.
2. **Touch support for region drawing.** Button-driven flow should work on mobile (no modifier needed), but the draw gesture needs to be confirmed with touch events during implementation.

## Not Doing (YAGNI)

- Freehand / brush / polygon selection shapes.
- Rotatable (non-axis-aligned) rectangles/ellipses.
- Named layers / visibility toggle.
- Layer opacity/blend modes beyond the inside/outside mask.
- Compressed URL hash encoding — ship the straightforward version first; revisit if hashes get unwieldy.
