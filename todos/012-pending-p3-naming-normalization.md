---
status: pending
priority: p3
issue_id: 012
tags: [code-review, consistency, naming]
dependencies: []
---

# Naming normalization: MODES keys, identifier casing

## Problem Statement

Two small consistency bumps worth making while the file is still ~800 lines and grep-friendly.

## Findings

From **pattern-recognition-specialist** (#2, #4):

1. **`MODES` uses string keys `'0'`-`'4'`** while the shader consumes `u_mode` as an int. Code does `+modeEl.value` at one site and `MODES[modeEl.value]` at another. Pick one — either integer keys throughout or `Number.parseInt` explicitly in `applyMode`.
2. **Identifier casing drifts**: `modeEl` / `fileIn` / `rowP2` / `btnOpen` / `valCX` / `valCY`. `CX`/`CY` are SHOUT-case while the rest are lowerCamel. Pick a prefix convention (`btn*`/`sld*`/`val*`/`lbl*`) and case convention (lowerCamel) so future sliders have a predictable name.

## Proposed Solutions

**Option A — Normalize naming, keep string keys**
String keys match `<select>` values, which are strings — leaves one explicit coercion site at read time, but fewer places to get it wrong. Rename only the identifiers (`valCX → valCx`, `valCY → valCy`, etc.).
- Pros: tiny change
- Cons: still two conceptual types for "mode"

**Option B — Normalize to integer keys throughout**
`MODES[0]`, `MODES[1]`, etc. Parse `<select>.value` once in `applyMode`. All downstream code speaks numbers.
- Pros: one type end-to-end
- Cons: slightly less obvious relationship between `<option value="0">` and `MODES[0]`
- Effort: Small
- Risk: Low

**Option C — Defer**
Fine if 001 (state refactor) subsumes most of this.

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Key sites**:
- `MODES` declaration: `:504-510`
- `applyMode`: `:512`
- `modeEl` / `+modeEl.value` sites: `:521, 527, 548, 653`
- Identifier casing: `:498-499` (`valCX`, `valCY`), `:491-492` (`cx`, `cy`)

## Acceptance Criteria

- [ ] One consistent type for mode throughout the codebase.
- [ ] Slider-related identifiers follow one naming rule; the rule is obvious from reading two examples.
- [ ] No behavioural change.

## Work Log

_(empty)_

## Resources

- Pattern-recognition #2, #4
