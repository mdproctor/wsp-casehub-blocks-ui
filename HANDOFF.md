# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-13
**Branch:** `main`
**Priority:** Channel-activity promotion landed. Connectors UI extraction and claudony integration are the follow-on. Pick up #33, #34, #26, or #52 next.

---

## Last Session

Promoted connectors' 8 qhorus chat-demo primitives into `@casehubio/blocks-ui-channel-activity` — channel-message, channel-input, channel-feed, channel-nav, channel-member-panel, channel-emoji-picker, channel-reaction-bar, channel-thread. Renamed `qhorus-*` → `channel-*`, event topics `chat:*` → `channel:*`. Added extension points (formatSender, renderContextHeader, renderError, type selector with allowedTypes/deniedTypes filtering, terminalDimming, eventStyling, autoScroll, staleCursorMinutes). Formalised component customisation protocol PP-20260713-8ea1af. Design-reviewed (5 rounds, 18 issues, all resolved). Filed pages#174 (gap surfacing) and pages#175 (cursor persistence).

## Immediate Next Step

Pick up #52 (chat-app module permanent home) or start connectors-side integration — update connectors' chat-demo to consume from blocks-ui instead of local primitives.

## Cross-Module

**We're blocking** (other modules waiting on us):
- `claudony` — can now retire hand-rolled channel-panel and consume `@casehubio/blocks-ui-channel-activity` · M · Med

**Blocked by** (can't proceed until):
- `casehub-pages#174` — gap surfacing on reconnect (gates stale cursor upgrade from timestamp-based to gap-based)
- `casehub-pages#175` — cursor persistence (gates page-reload cursor resumption)

## What's Left

- #47 — audit-trail-viewer row expand. Blocked by casehub-pages#172. · XS · Low
- #52 — chat-app module permanent home decision. · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #33 | Subscription editor component | M | Med | |
| #34 | Notification preferences and suppression UI | M | Med | |
| #26 | Data-table row and column spanning | M | Med | |
| #35 | Cross-repo component migration tracking | L | High | Epic — #39 channel panel portion done |

## References

- Spec: `docs/specs/2026-07-13-channel-activity-promotion-design.md`
- Protocol: `docs/protocols/blocks-ui/component-customisation-pattern.md`
- Blog: `blog/2026-07-13-mdp02-three-repos-one-component-library.md`
- Pages issues: casehub-pages#174, casehub-pages#175
- Chat-app home: #52
- Previous: `git show HEAD~1:HANDOFF.md`
