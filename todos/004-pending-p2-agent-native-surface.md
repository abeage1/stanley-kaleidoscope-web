---
status: pending
priority: p2
issue_id: 004
tags: [code-review, agent-native, api-surface, architecture]
dependencies: ["001"]
---

# Add agent-native surface: window.__kaleido + URL hash state + data attributes

## Problem Statement

Today every action requires DOM-event simulation on unlabeled elements, with state read implicitly from input `.value`. An agent (Playwright/Puppeteer, MCP tool, future browser-automation) can drive the app but has no stable contract, no way to round-trip a result, and no URL-shareable state.

## Findings

From **agent-native-reviewer** (#1-3):
- *"`window.__kaleido.setState/loadImageFromUrl/exportPNG` unlocks headless drivers, MCP wrappers, eval-based automation — all without rewriting the app."*
- *"URL-hash-encoded state is the single biggest leverage point: shareable, reproducible, bookmarkable, and an agent can drive by navigating to a URL."*
- *"`data-kaleido=` attributes: two-minute change, permanently stable selectors (IDs exist but aren't a documented contract)."*

## Proposed Solutions

**Option A — All three (recommended)**
1. `window.__kaleido = { state, setState(patch), loadImageFromUrl, exportPNG(): Promise<Blob>, ready: true }` exposed as a stable contract (~25 LOC).
2. URL hash state: parse `#m=2&p1=50&rot=45&zoom=1.2&cx=50&cy=50[&img=url]` on load; write on settled state change (~15 LOC).
3. `data-kaleido=` attributes on every control (~2 min).
- Pros: full agent-native parity in ~45 LOC; also benefits users (shareable URLs are a product feature).
- Cons: Requires 001 (state lift) first to be clean.
- Effort: Medium
- Risk: Low

**Option B — Just URL hash state**
- Pros: smallest change, biggest user-visible payoff
- Cons: Agents still need JS injection for one-shot ops like `exportPNG()`

**Option C — Defer until agent use-case materializes**
- Pros: YAGNI
- Cons: Adds churn to every future feature — each one must be retrofitted

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`

**Surfaces to expose**:
```js
window.__kaleido = {
  get state() { /* return normalized state */ },
  setState(patch),         // applies patch, updates labels, renders
  loadImageFromUrl(url, name),
  loadImageFromDataUrl(dataUrl, name),
  exportPNG(): Promise<Blob>,
  MODES,
  ready: true
};
```

**URL hash schema** (v1):
- `m` = mode (0-4)
- `p1`, `p2`, `rot`, `zoom`, `cx`, `cy` = sliders in human units
- `img` = optional URL to preload

**Data attributes**:
- `data-kaleido="mode"`
- `data-kaleido="slider-p1"` through `"slider-cy"`
- `data-kaleido="btn-open"`, `"btn-save"`, `"btn-reset"`
- `data-kaleido="sample" data-sample-id="…"`

**Future** (P3 follow-ups — see 011):
- `kaleido:change` CustomEvent for reactivity
- `postMessage` bridge when embedded in iframe

## Acceptance Criteria

- [ ] An agent can reproduce any state by setting `location.hash` and reloading.
- [ ] An agent can drive the app via `window.__kaleido.setState({mode:2, rot:45})` and observe the result.
- [ ] `window.__kaleido.exportPNG()` resolves to a Blob without touching the DOM.
- [ ] Every interactive control carries a stable `data-kaleido` attribute.
- [ ] Surface is documented in a short `<!-- AGENT API -->` HTML comment at the top of the script.

## Work Log

_(empty)_

## Resources

- Agent-native-reviewer report
