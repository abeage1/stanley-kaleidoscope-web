---
status: pending
priority: p2
issue_id: 008
tags: [code-review, security, hygiene]
dependencies: []
---

# Add CSP and Referrer-Policy meta tags

## Problem Statement

GitHub Pages serves no `Content-Security-Policy`, `X-Frame-Options`, or `Referrer-Policy` headers by default. The app has no sensitive surface today, but a CSP meta tag is cheap defence-in-depth — it blocks future regressions (e.g. someone introduces `innerHTML` from untrusted input) and makes clickjacking/embedding explicit policy.

## Findings

From **security-sentinel** (P2-1): *"No HTTP security headers / CSP. Low impact because the app has no sensitive surface, but a CSP meta tag would defence-in-depth future regressions and block clickjacking/embedding."*

## Proposed Solutions

**Option A — Meta tags (recommended)**
In `<head>`:
```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  img-src 'self' blob: data: https://images.unsplash.com;
  style-src 'self' 'unsafe-inline';
  script-src 'self' 'unsafe-inline';
  connect-src 'none';
  frame-ancestors 'none';
  base-uri 'none';
">
<meta name="referrer" content="no-referrer">
```
Note: inline `<style>` and `<script>` require `'unsafe-inline'`. To drop `'unsafe-inline'`, inline code would need to move to external files — conflicts with the zero-dependency single-file principle. Keep `'unsafe-inline'` for now.
- Pros: trivial change, safer posture, no runtime cost
- Cons: must verify nothing breaks (iframe-embedded samples, drag-drop)
- Effort: Small
- Risk: Low — test locally with DevTools Console watching for CSP violations

**Option B — Defer until a feature requires it (e.g. MCP embedding)**
- Pros: no churn
- Cons: easy now, annoying later

## Recommended Action

_(Filled during triage)_

## Technical Details

**Affected file**: `/Users/abeage/agent/kaleidoscope-web-review/index.html`
**Key sites**:
- Insert after `<meta name="viewport">` at `:5`

**If the app is ever embedded in a Claude artifact or iframe** (see 011), `frame-ancestors` will need to allow the parent origin.

## Acceptance Criteria

- [ ] CSP meta tag covers script, style, img, connect, frame-ancestors, base-uri.
- [ ] Unsplash images still load; drag-drop still works; export still works.
- [ ] DevTools Console shows no CSP violations during normal use.

## Work Log

_(empty)_

## Resources

- Security-sentinel P2-1
