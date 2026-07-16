# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-16
**Branch:** `main`
**Priority:** #53 grouped-data-view landed. 22 packages total (was 21). Pages composability (#188) shipped — pages-grouped-view now composes pages-table. Enhancement issue #196 filed for interstitial hooks, legend, rowAccent, nested grouping.

---

## Last Session

Completed #53 (renamed from queue-board to grouped-data-view). Key insight: queue-board was a configuration of a generic grouped table pattern, not a dedicated component. Filed and got casehub-pages#188 (composability) shipped first. Built grouped-data-view as thin DataSourceMixin wrapper over composable pages-grouped-view. Design-reviewed (4 rounds, $14.86). Also committed #68 (button color), #69 (thread reactions) PRs from another session, and synced #66 to fork.

## Immediate Next Step

Pick up #67 (column renderers not applied in similarity-panel/compliance-summary) — pages-table integration issue. Run `/work` to start.

## What's Left

- Channel-activity gap analysis — connectors has 30 qhorus UI files diverged from blocks-ui's channel-activity. Needs full diff before migration. · M · Med
- #67 column renderers not applied in similarity-panel/compliance-summary — pages-table integration issue · S · Med
- Parent deep-dive doc (`docs/repos/casehub-blocks-ui.md`) stale — references DataEndpointMixin (renamed), missing new components · S · Low
- casehub-pages#196 — table enhancements: interstitial hooks, legend component, rowAccent, nested grouping · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #67 | Column renderers not applied | S | Med | pages-table binding investigation |
| #54 | Routing rationale component | M | Med | DevTown priority |
| #33 | Subscription editor | M | Med | notification-inbox incomplete without it |
| #34 | Notification preferences UI | M | Med | notification-inbox incomplete without it |
| #26 | Data-table row and column spanning | M | Med | clinical regulatory grid |

## References

- Previous: `git show HEAD~1:HANDOFF.md`
