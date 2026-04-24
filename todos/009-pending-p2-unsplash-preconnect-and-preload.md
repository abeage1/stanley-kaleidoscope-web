---
status: pending
priority: p2
issue_id: 009
tags: [code-review, performance, ux]
dependencies: []
---

# Preconnect to Unsplash; preload full sample on hover

## Problem Statement

Sample thumbnails load instantly, but clicking one pays 200-800 ms latency before the full-resolution image appears. The user sees only a "Loading…" status line with no progress. First-click experience is noticeably slow.

## Findings

From **performance-oracle** (#4): *"First click pays 200-800 ms network latency with no visual feedback. Consider `<link rel="preconnect" href="https://images.unsplash.com">` in `<head>`, or start the full fetch on `mouseenter` of the thumb."*

## Proposed Solutions

**Option A — Preconnect + hover-prefetch (recommended)**
1. Add `<link rel="preconnect" href="https://images.unsplash.com" crossorigin="anonymous">` to `<head>`.
2. On `mouseenter` / `touchstart` of a sample thumbnail, create a hidden `<img>` with the full URL to warm the browser cache.
- Pros: big perceived-perf win on first interaction; ~8 LOC.
- Cons: prefetches bytes user may not want (minor for ≤ 200 KB images).
- Effort: Small

**Option B — Preconnect only**
- Pros: saves TLS handshake (~50-150 ms), no speculative bandwidth
- Cons: body download still on click

**Option C — Do nothing**
- Pros: zero work
- Cons: first click stays sluggish

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Key sites**:
- `<head>` additions: after `:5`
- Sample grid builder: `:745-770`
- Unsplash URL builders: `:728-733`

## Acceptance Criteria

- [ ] Preconnect link present in `<head>`.
- [ ] Hovering a sample starts downloading the full-resolution image.
- [ ] Clicking an already-hovered sample shows the image in < 100 ms in normal network conditions.

## Work Log

_(empty)_

## Resources

- Performance-oracle #4
