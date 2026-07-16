# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-16
**Branch:** `main`
**Priority:** #64 and #65 landed (channel-activity extension points for claudony). PR #74 open on blessed repo. 22 packages.

---

## Last Session

Delivered #64 (channel-nav compact dropdown mode, showCreate/showDelete toggles, messageCounts) and #65 (renderContent callback on channel-message). Both claudony-requested. Also fixed formatSender passthrough gap on channel-feed and channel-thread. Discovered shadow DOM `<select>` positioning gotcha — native popups displace in nested shadow roots with scrolled containers; replaced with custom dropdown. Garden entry GE-20260716-424a17.

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
