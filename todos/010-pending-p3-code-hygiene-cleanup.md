---
status: pending
priority: p3
issue_id: 010
tags: [code-review, simplicity, hygiene]
dependencies: []
---

# Code hygiene: magic numbers, name shadow, dead CSS, redundant comments

## Problem Statement

Small cleanup items that don't block anything but reduce future friction. Bundled into one pass because they touch unrelated parts of the same file and are each too small to be their own todo.

## Findings

From multiple agents (simplicity #6, #7, #10, #11, patterns #7, #11, #12):

1. **Magic numbers scattered**: `PAD = 24` (`:563`), thumbnail sizes `60/120/800/1600` (`:729, 732, 751, 755, 756`). Extract to a small `CONFIG` const near the top.
2. **`status` const shadows `window.status`** (`:484, 688`). Rename to `statusEl` to match the `modeEl` convention and avoid the built-in clash.
3. **Redundant `gap: 0` on `.ctrl-group`** (`:85`) — flexbox default; delete.
4. **Decorative section banner comments** (`// ── UI refs ──`, `// ── Labels ──`, `// ── Status ──`): keep or drop consistently. Current file mixes decorative banners with meaningful section headers.
5. **Comments that restate the code**: e.g., `/* Inner helper — accepts HTMLImageElement or HTMLCanvasElement */` (`:577`), `/* Rotation */` (`:351`), `/* Texture */` (`:463`). Remove unless they explain *why*.
6. **`setStatus(msg)` one-liner** (`:688`) — borderline; 7 callers. Inline if you want fewer symbols, keep if you want a future hook for toasts/logging.
7. **Arbitrary `setTimeout(..., 10000)` for `revokeObjectURL`** (`:680`) — revoking immediately after `a.click()` works in all current browsers. Drop the timer.
8. **`dragenter` + `dragover` + `dragleave` trio** (`:618-620`) — `dragover` must `preventDefault` and fires continuously, so setting `.drag-over` in `dragover` and removing in `dragleave` is sufficient. `dragenter` listener is redundant.

## Proposed Solutions

Single PR that does items 1-5, 7, 8. Keep item 6 (`setStatus`) as-is unless you're adding a logging/toast layer.
- Effort: Small (30-45 min)
- Risk: Very low

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Key sites**:
- CSS: `:85`
- Comments: `:337, 351, 448, 457, 463, 577` and similar
- `status` identifier: `:484, 688, 301, 688`
- Magic numbers: `:563, 729, 732, 751, 755, 756`
- revokeObjectURL timer: `:680`
- Drag listeners: `:618-620`

## Acceptance Criteria

- [ ] No magic numbers in rendering/sample code; all live in a `CONFIG` const.
- [ ] `status` renamed to `statusEl` or similar; no shadowing of `window.status`.
- [ ] Dead CSS (`gap: 0` on `.ctrl-group`) removed.
- [ ] Comment density reduced; remaining comments explain *why*, not *what*.
- [ ] `revokeObjectURL` called immediately after download anchor click.
- [ ] Drag handlers reduced from three listeners to two.

## Work Log

_(empty)_

## Resources

- Simplicity reviewer (#6, #7, #10, #11, #14, #15)
- Pattern-recognition (#7, #11, #12)
