---
status: pending
priority: p2
issue_id: 006
tags: [code-review, simplicity, refactor]
dependencies: []
---

# Consolidate loadImage / loadImageFromUrl; fix if(!gl) fall-through

## Problem Statement

Two small reliability/clarity issues in the image-loading code:

1. `loadImage` (file) and `loadImageFromUrl` (URL) duplicate the onload/onerror/`_uploadToGL` pattern (`:592-610`). File loading could call `loadImageFromUrl(URL.createObjectURL(file), file.name)` with a revoke hook, eliminating the duplication.
2. `if (!gl) { alert('WebGL not supported…'); }` at `:429` does not early-return. Code below unconditionally uses `gl` and throws `TypeError` anyway. The alert is user-friendly intent but broken execution.

## Findings

From **simplicity reviewer** (#3, #4):
- *"`loadImage` could call `loadImageFromUrl(URL.createObjectURL(file), file.name)` with a revoke hook."*
- *"`if (!gl) { alert(...); }` then proceeds to use `gl` unconditionally → would throw anyway. Either return/throw or delete the alert."*

## Proposed Solutions

**Option A — Single helper + proper guard**
```js
function loadImageFromSrc(src, name, cleanup) {
  setStatus('Loading…');
  const img = new Image();
  img.crossOrigin = 'anonymous';      // harmless for blob URLs
  img.onload  = () => { _uploadToGL(img, name); cleanup?.(); };
  img.onerror = () => { setStatus('Could not load "' + name + '".'); cleanup?.(); };
  img.src = src;
}
function loadImage(file) {
  if (!file || !file.type.startsWith('image/')) { setStatus('Unsupported file type.'); return; }
  const url = URL.createObjectURL(file);
  loadImageFromSrc(url, file.name, () => URL.revokeObjectURL(url));
}
function loadImageFromUrl(url, name) { loadImageFromSrc(url, name); }
```
For the `gl` guard: `if (!gl) { alert('WebGL not supported.'); throw new Error('WebGL unavailable'); }` or wrap everything below in an `init()` function.
- Pros: ~15 LOC removed, clearer failure path
- Cons: none
- Effort: Small
- Risk: Low — but verify `crossOrigin='anonymous'` on blob URLs doesn't regress local-file loading (it shouldn't)

**Option B — Leave as-is**
- Pros: zero work
- Cons: duplication gets worse if future features add more load paths (paste from clipboard, camera capture)

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Key sites**:
- WebGL guard: `:429`
- `_uploadToGL`: `:578-590`
- `loadImage`: `:592-601`
- `loadImageFromUrl`: `:603-610`

## Acceptance Criteria

- [ ] File and URL paths share one helper.
- [ ] If WebGL init fails, the user sees a single clear error and no `TypeError` is thrown at init time.
- [ ] Local-file loading still works (blob URL + `crossOrigin='anonymous'` compatibility verified).

## Work Log

_(empty)_

## Resources

- Simplicity reviewer #3, #4
