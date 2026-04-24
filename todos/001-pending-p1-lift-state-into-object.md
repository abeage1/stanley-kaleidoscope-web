---
status: pending
priority: p1
issue_id: 001
tags: [code-review, architecture, refactor, foundation]
dependencies: []
---

# Lift state into a plain object; make render() read from state, not DOM

## Problem Statement

Every slider read in `render()` is a direct `+p1.value` / `+cx.value` DOM access. This couples rendering to the DOM and blocks a stack of planned features:

- Undo/redo
- Save/load presets
- URL-shareable state (`#m=2&p1=50&…`)
- Agent-drivable control surface
- Unit testing of mode math

It also explains the repeated `updateLabels(); render();` pair scattered across 5+ sites (mode change, input, click-to-center, reset, sample-load).

## Findings

From **architecture-strategist**: *"The single change with the biggest payoff… introduce `const state = { mode:0, p1:8, p2:1.0, rot:0, zoom:1, cx:50, cy:50 }`. One listener loop: `input → write state → updateLabels(state) → render(state)`. This unlocks undo/redo, presets, URL sharing, and testability."*

From **pattern-recognition-specialist** (#1, #5, #6): slider scale factors (10×, 100×) are duplicated at slider declaration, in `updateLabels()`, and in `render()`. Every format is repeated. Reset hardcodes literals.

From **agent-native-reviewer** (#1): `window.__kaleido.setState(patch)` becomes a 10-line wrapper once state exists.

## Proposed Solutions

**Option A — Config-driven slider table + single state object**
Declare `const SLIDERS = { p1: {el:p1, scale:1, fmt:v=>v}, rot: {el:rot, scale:1/10, fmt:v=>v.toFixed(1)+'°'}, … }`. One render pipeline: `syncStateFromDom() → updateLabels(state) → render(state)`. Reset becomes a `for (k of SLIDERS) el.value = SLIDERS[k].def`.
- Pros: removes ~40 LOC of duplication, unlocks every downstream feature
- Cons: touches every control site — ~1 hour of careful work
- Effort: Medium
- Risk: Low (pure refactor, no behaviour change)

**Option B — State object but keep DOM as source of truth**
Add a `function getState()` that reads DOM once per frame; pass the returned object to `render(state)`. Don't restructure sliders.
- Pros: 5-minute change, no risk
- Cons: doesn't unlock presets/URL/undo without more work — half-step
- Effort: Small

**Option C — Defer entirely; keep direct DOM reads**
- Pros: zero work now
- Cons: Every future feature pays the cost instead — net slower

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Key sites**:
- MODES config: `:504-510`
- `applyMode`: `:512-519`
- `updateLabels`: `:526-534`
- Slider input loop: `:536-538`
- `render`: `:543-556`
- Reset: `:652-657`
- `setCenter`: `:632-639`

## Acceptance Criteria

- [ ] Single `state` object or accessor holds every mode/slider value in human units (degrees, not `rot*10`).
- [ ] `render()` takes no DOM reads — pure function of state.
- [ ] Every `updateLabels(); render();` pair replaced by one entry point (e.g. `applyState(patch)`).
- [ ] Reset uses iteration over the slider config, not inline literals.
- [ ] No user-visible behaviour change; existing interactions still work.

## Work Log

_(empty)_

## Resources

- Architecture agent report (see review artefact)
- Pattern-recognition findings #1, #5, #6
