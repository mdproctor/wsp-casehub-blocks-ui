# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-17
**Branch:** `main`
**Priority:** #82 closed (emoji picker viewport fix). Schema-form epic #81 filed with 5 children. #33/#34 branch created but no implementation yet.

---

## Last Session

Fixed emoji picker viewport overflow (#82) — dead CSS `.flip` class was defined but never wired to JS. Added `_computePickerPosition()` with `getBoundingClientRect`-based flip/align logic. Garden entry GE-20260717-6610cc submitted.

Designed #33 (subscription editor) + #34 (notification preferences) together. Identified schema-form gaps that gate both issues. Filed epic #81 with five children: #76 array editing, #77 nested object editing, #78 field metadata, #79 validation, #80 getEventTypes API. Branch `issue-33-subscription-prefs` exists but has no implementation — waiting on schema-form enhancements.

## Immediate Next Step

Start schema-form epic #81 — build order: #78 + #80 first (quick wins), then #77 → #76, then #79.

## What's Left

- Channel-activity gap analysis — connectors has 30 qhorus UI files diverged from blocks-ui's channel-activity. Needs full diff before migration. · M · Med
- Parent deep-dive doc (`docs/repos/casehub-blocks-ui.md`) stale — references DataEndpointMixin (renamed), missing new components · S · Low
- casehub-pages#196 — table enhancements: interstitial hooks, legend component, rowAccent, nested grouping · M · Med
- Engine REST endpoint for routing decision data — blocks endpoint mode of routing-rationale · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #81 | Schema-form enhancements epic (5 children) | L | Med | Gates #33 and #34 |
| #33 | Subscription editor | M | Med | Blocked by #81 |
| #34 | Notification preferences UI | M | Med | Blocked by #81 |
| #26 | Data-table row and column spanning | M | Med | Clinical regulatory grid |

## References

- Previous: `git show HEAD~1:HANDOFF.md`
