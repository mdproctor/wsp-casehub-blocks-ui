*Updated: #26 closed — removed from backlog.*

# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-19
**Branch:** `issue-26-data-table-row-col-spanning`
**Priority:** Cell spanning design complete — handed to pages as epic casehub-pages#210. blocks-ui needs new work.

---

## Last Session

Designed data-table cell spanning (row + column). Replaced the per-row CSS Grid model with a single body grid in the spec. Design-reviewed (13 findings, all addressed). Filed 7 child issues (#211–#217) against casehub-pages. Closed blocks-ui#26, #76, #77, #78, #79. Garden entry GE-20260719-4db710 (CSS Grid single-container virtual scroll technique).

## Immediate Next Step

Close this branch via `/work end` — design deliverables are complete. Then find new blocks-ui work from the What's Next table or check the issue tracker for unblocked items.

## What's Left

- Channel-activity gap analysis — connectors has 30 qhorus UI files diverged from blocks-ui's channel-activity · M · Med
- Parent deep-dive doc (`docs/repos/casehub-blocks-ui.md`) stale · S · Low
- Engine REST endpoint for routing decision data · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #33 | Subscription editor | M | Med | Blocked by pages #81 (needs #207, #208 — now landed, pages needs to close) |
| #34 | Notification preferences UI | M | Med | Blocked by pages #81 |
| #80 | NotificationApi.getEventTypes() | XS | Low | API work, not schema-form |

## References

- Previous: `git show HEAD~1:HANDOFF.md`
