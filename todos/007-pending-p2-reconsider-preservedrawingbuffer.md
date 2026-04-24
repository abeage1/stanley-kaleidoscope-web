---
status: pending
priority: p2
issue_id: 007
tags: [code-review, performance, webgl]
dependencies: []
---

# Drop preserveDrawingBuffer: true by rendering inside toBlob window

## Problem Statement

`preserveDrawingBuffer: true` (`:428`) is paying a per-frame cost on every GPU today just to make the one-off `toBlob` export work. On Apple GPUs and Adreno this has a measurable FPS impact during slider scrubs.

## Findings

From **performance-oracle** (#5): *"It's only needed at toBlob time. Modern browsers capture the current buffer if you render immediately before toBlob in the same task. Alternatively, render to an FBO on export. Removing preserveDrawingBuffer gives a measurable FPS boost on Apple GPUs and Adreno."*

## Proposed Solutions

**Option A — Synchronous render + toBlob in same task**
```js
btnSave.addEventListener('click', () => {
  // resize...
  render();                    // same task
  canvas.toBlob(blob => { ... });  // captures the just-rendered frame
});
```
Works on modern Chrome/Safari/Firefox because the drawing buffer is kept alive until the end of the current task if `toBlob` is queued synchronously after draw.
- Pros: drops `preserveDrawingBuffer`, free FPS on hot path
- Cons: slightly more brittle — must keep `render()` and `toBlob()` in same synchronous task; document this with a short comment
- Effort: Small
- Risk: Low-Medium — validate on Safari/iOS specifically

**Option B — Render to an offscreen FBO for export, readPixels, encode to PNG manually**
- Pros: cleanest architecture, no `preserveDrawingBuffer` dependency
- Cons: ~40 LOC of boilerplate, custom PNG encoder or another canvas element
- Effort: Medium

**Option C — Leave `preserveDrawingBuffer: true`**
- Pros: zero risk
- Cons: pays perf tax forever

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Key sites**:
- GL context creation: `:428`
- Export: `:662-683`

Combine with the GPU-limit clamp from 003 — both touch the export path.

## Acceptance Criteria

- [ ] `preserveDrawingBuffer` either removed or justified with an inline comment.
- [ ] Export on Safari/iOS still produces a non-empty PNG.
- [ ] Slider scrub FPS either measurably improves or the change is reverted with notes.

## Work Log

_(empty)_

## Resources

- Performance-oracle #5
- Related: 003 (export-path hardening)
