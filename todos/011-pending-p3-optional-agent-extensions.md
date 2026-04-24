---
status: pending
priority: p3
issue_id: 011
tags: [code-review, agent-native, enhancement]
dependencies: ["004"]
---

# Optional agent-native extensions: CustomEvent + postMessage bridge

## Problem Statement

Follow-ups to 004 that extend the agent-native surface beyond the core API. Only relevant if 004 ships and real agent use-cases emerge.

## Findings

From **agent-native-reviewer** (#4, #5):
- *"`kaleido:change` CustomEvent dispatched from `render()` with `{detail: state}` — lets an agent await a settled state after setState, and enables future reactive embeds."*
- *"`postMessage` bridge when `window.parent !== window`: accept `{type:'setState'|'loadImage'|'export', ...}`, reply with results. Enables embedding in Claude artifacts, proof docs, or parent agent UIs without CORS gymnastics."*

## Proposed Solutions

**Option A — Both**
1. Dispatch `document.dispatchEvent(new CustomEvent('kaleido:change', { detail: state }))` from `render()`.
2. Add a small `postMessage` listener that routes to `__kaleido.setState/loadImageFromUrl/exportPNG` when embedded as an iframe.
- Pros: enables richer embedding and reactive UIs; ~25 LOC total
- Cons: speculative — don't ship until there's a concrete consumer
- Effort: Small-Medium

**Option B — CustomEvent only**
- Pros: pure read-only hook, minimal risk
- Cons: doesn't enable iframe embedding

**Option C — Defer until needed**
- Pros: YAGNI
- Cons: none

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Depends on**: 004 (core agent-native surface)
**Key sites**:
- `render`: `:543-556` (CustomEvent)
- New listener near init (`postMessage`)

## Acceptance Criteria

- [ ] `kaleido:change` event fires with current state after every render; normal usage emits at most one per animation frame.
- [ ] `postMessage` bridge handles `setState`/`loadImage`/`export` with a documented schema.
- [ ] CSP `frame-ancestors` in 008 updated to allow known parent origins (or kept `'none'` if postMessage is iframe-only).

## Work Log

_(empty)_

## Resources

- Agent-native-reviewer #4, #5
- Related: 004, 008
