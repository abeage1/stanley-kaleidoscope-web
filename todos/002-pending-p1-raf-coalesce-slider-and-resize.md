---
status: pending
priority: p1
issue_id: 002
tags: [code-review, performance, rendering]
dependencies: []
---

# rAF-coalesce slider input + window resize → render

## Problem Statement

`input` events on sliders can fire faster than vsync (high-polling mice, touch drags). Each event synchronously invokes `render()`, which runs a fullscreen fragment shader over the canvas. On a 2560×1440 stage during a rapid scrub the browser queues redundant draws and may block on the GL pipeline.

`window.resize` (`:571`) has the same problem and is worse: each event reallocates the canvas drawing buffer (expensive) *and* draws a frame. On a 4K display dragging a window edge is the most GPU-heavy path in the app.

## Findings

From **performance-oracle** (#1, #3): *"The single highest-value change. Set a `pending` flag, schedule via `requestAnimationFrame`, read latest slider values inside the rAF callback… Window drags fire dozens of resize events per second, each reallocating the drawing buffer."*

## Proposed Solutions

**Option A — Single `scheduleRender()` helper used by every trigger**
```js
let pending = false;
function scheduleRender() {
  if (pending) return;
  pending = true;
  requestAnimationFrame(() => { pending = false; render(); });
}
```
Replace every `render()` call that reacts to user input (slider `input`, resize, click-to-center, mode change) with `scheduleRender()`. Leave the export path using the synchronous `render()` directly.
- Pros: ~10-line change, covers both issues, obvious correctness
- Cons: None material
- Effort: Small
- Risk: Very low

**Option B — Debounce resize, leave sliders alone**
- Pros: simpler
- Cons: leaves the slider-scrub problem untouched; sliders are the more common case

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Key sites**:
- Slider listeners: `:536-538`
- Resize listener: `:571`
- `setCenter`: `:632-639`
- Mode change: `:521`
- Reset: `:652-657`

**Do NOT coalesce**:
- Export (`:662-683`) — needs synchronous render before `toBlob`.
- Initial `_uploadToGL` (`:588`) — runs once, not rapid-fire.

## Acceptance Criteria

- [ ] `scheduleRender` helper added; rapid slider scrubs emit at most one draw per animation frame.
- [ ] Window resize coalesces the same way — dragging a window edge does not produce visible lag.
- [ ] Export still works and produces pixel-identical output.

## Work Log

_(empty)_

## Resources

- Performance-oracle report
