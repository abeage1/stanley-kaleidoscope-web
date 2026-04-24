---
status: pending
priority: p2
issue_id: 005
tags: [code-review, webgl, shader, simplicity, performance]
dependencies: []
---

# Shader cleanup: DRY tile setup, remove redundant clamp, verify radial passthrough, precision fallback

## Problem Statement

The fragment shader is correct but has small inefficiencies and compatibility concerns worth addressing before the shader grows (filters, more modes).

## Findings

Combined from multiple agents:

1. **Duplicated `ts = u_p1/100 * mdim` across tile modes 1-4** (`:378, 385, 392, 407`) — hoist above the branch. (patterns #3)
2. **`clamp(uv, 0.0, 1.0)` at `:419` is redundant** with `CLAMP_TO_EDGE` (`:466-467`) — belt-and-suspenders. (simplicity #9)
3. **Radial `u_p1 < 2.0` passthrough branch** (`:361-374`) likely equivalent to the general formula when `segments=1`. Verify and remove if so. (simplicity #8)
4. **`highp` precision on fragment shader** (`:313`) — fails to compile on older Mali / PowerVR. Guard with `#ifdef GL_FRAGMENT_PRECISION_HIGH`. Shader math tolerates `mediump`. (performance #6)
5. **`SQRT3` has 11 digits vs `PI`'s 14** (`:326-328`) — cosmetic but matters for tile seams at high zoom. Use `1.7320508075688772`. (patterns #8)

## Proposed Solutions

**Option A — All five as a single shader-cleanup pass**
- Pros: shader is touched once; easy to re-test visually
- Cons: none material
- Effort: Small
- Risk: Low-Medium (item 3 requires visual verification that `segments=1` output is bit-identical)

**Option B — Skip item 3 (radial passthrough)**
If the passthrough branch proves non-equivalent on close inspection, keep it — it was added deliberately per the commit log (`8ed3c37 Support Segments=1 as original-image passthrough in Radial mode`). Verify math first.

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Affected regions**:
- Shader source: `:312-422`
- Constants: `:326-328`
- Precision qualifier: `:313`
- `clamp`: `:419`
- Radial block: `:361-374`
- Tile `ts` setup: `:378, 385, 392, 407`

**Verification needed for radial passthrough**: compute `r*cos(tn)` and `r*sin(tn)` with `segments=1` (`seg=TWO_PI`) — should equal `d.x, d.y`. If bit-identical visually under extreme zoom, remove the branch.

## Acceptance Criteria

- [ ] `ts = u_p1/100 * mdim` computed once outside the tile-mode chain.
- [ ] Top-level `clamp(uv, 0, 1)` removed; output still identical (CLAMP_TO_EDGE handles it).
- [ ] Either the radial `u_p1 < 2.0` branch is removed OR a comment is added explaining why it's not equivalent.
- [ ] Shader compiles and runs on devices without `GL_FRAGMENT_PRECISION_HIGH` (test path exists even if hardware not available).
- [ ] `SQRT3` uses 16-digit precision.

## Work Log

_(empty)_

## Resources

- Simplicity reviewer (#8, #9)
- Pattern-recognition (#3, #8)
- Performance-oracle (#6)
- Prior commit: `8ed3c37` (radial passthrough rationale)
