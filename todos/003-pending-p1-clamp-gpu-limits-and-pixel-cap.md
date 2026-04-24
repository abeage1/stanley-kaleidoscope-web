---
status: pending
priority: p1
issue_id: 003
tags: [code-review, performance, reliability, security]
dependencies: []
---

# Clamp export dimensions to GPU limits; reject oversized images

## Problem Statement

Two related bugs in the handling of large images:

1. **Load path**: `_uploadToGL` (`:578-590`) calls `gl.texImage2D` with whatever the user supplies. A 20000×20000 PNG (~1.6 GB decoded) OOM-crashes the tab. On older iOS devices `MAX_TEXTURE_SIZE` is 4096 — a 6000px Unsplash sample exceeds it and texture upload silently fails.
2. **Export path** (`:667-671`): canvas is scaled up to `max(imgW, imgH)` on the larger axis. No check against `gl.MAX_VIEWPORT_DIMS`. On a 4096-limit device exporting a 6000px source produces a silently-clipped or all-black PNG.

Additionally, during export the drawing buffer is doubled temporarily (live canvas at export size, plus PNG encoder buffer), briefly spiking memory 2×.

## Findings

From **performance-oracle** (#2): *"On a 6000×4000 source this is ~96 MB drawing-buffer + another ~96 MB for the PNG encoder, briefly doubled. Risk: OOM on mobile Safari. `gl.getParameter(gl.MAX_TEXTURE_SIZE)` on older iOS is 4096."*

From **security-sentinel** (P2-2): *"No check against MAX_TEXTURE_SIZE or a reasonable pixel cap. Malicious page embedding/linking a file via drag-drop could DoS the tab."*

## Proposed Solutions

**Option A — Clamp on upload + clamp on export**
- On upload: query `const MAX = gl.getParameter(gl.MAX_TEXTURE_SIZE)` at init. If `imgW > MAX || imgH > MAX`, downscale via an offscreen canvas before `texImage2D` and show a status message.
- On export: compute target dims, then clamp to `min(MAX_VIEWPORT_DIMS[0], MAX_VIEWPORT_DIMS[1])`. Warn user if clamped.
- Optionally add a soft pixel cap (e.g., reject > 16384 on any axis with a friendly error) to pre-empt huge-image OOM before GL even sees the image.
- Effort: Small-Medium
- Risk: Low

**Option B — Clamp only on export**
- Pros: fewer code paths touched
- Cons: leaves OOM on upload unsolved

**Option C — Render export to an offscreen FBO, readPixels, encode**
- Pros: live canvas never disturbed; cleaner architecture
- Cons: ~30 lines of FBO boilerplate, still needs the clamp
- Effort: Medium

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Key sites**:
- `_uploadToGL`: `:578-590`
- `loadImage` type check: `:592-601`
- Export path: `:662-683`
- GL init: `:427-470`

**WebGL guards to add**:
- `gl.getParameter(gl.MAX_TEXTURE_SIZE)` — texture upload cap
- `gl.getParameter(gl.MAX_VIEWPORT_DIMS)` — `drawArrays` viewport cap
- Optional soft cap: `imgW * imgH > 64_000_000` → reject with status message

## Acceptance Criteria

- [ ] Oversized images are downscaled before upload, not silently truncated by the driver.
- [ ] Export dimensions clamp to `MAX_VIEWPORT_DIMS` with a visible status-bar warning when clamped.
- [ ] A deliberately huge test image (e.g., 20000×20000) produces a clean error, not a tab crash.
- [ ] Normal-sized images (≤ 4000px) still export at native resolution.

## Work Log

_(empty)_

## Resources

- Performance-oracle report (#2)
- Security-sentinel report (P2-2)
