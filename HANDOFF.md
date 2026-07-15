# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-15
**Branch:** `main`
**Priority:** #38 clinical promotion complete — 5 components + commitmentLifecycleStrategy landed on main. 21 packages total (was 16). Column renderer issue (#67) filed for similarity-panel and compliance-summary. Channel-activity divergence still open.

---

## Last Session

Completed #38 clinical UI promotion epic. Design-reviewed (18 issues, all verified, $18). Built: createTypedFetchSource + EMPTY_DATASET in core, commitmentLifecycleStrategy in blocks-timeline, similarity-panel, compliance-summary, trust-feedback-display, sla-breach-policy, gdpr-erasure-action. 6 showcase pages in examples. Fixed 4 pre-existing vitest watch-mode hangs + sla-indicator alias issue. 387 tests all green. Squashed to single commit and pushed to both fork and blessed.

## Immediate Next Step

Pick up #67 (column renderer issue) — pages-table columnRenderers not applied in similarity-panel and compliance-summary. Investigate pages-table property binding. Run `/work` to start.

## What's Left

- Channel-activity gap analysis — connectors has 30 qhorus UI files diverged from blocks-ui's channel-activity. Needs full diff before migration. · M · Med
- #67 column renderers not applied in similarity-panel/compliance-summary — pages-table integration issue · S · Med
- Parent deep-dive doc (`docs/repos/casehub-blocks-ui.md`) stale — references DataEndpointMixin (renamed), missing new components · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #67 | Column renderers not applied | S | Med | pages-table binding investigation |
| #54 | Routing rationale component | M | Med | DevTown priority |
| #33 | Subscription editor | M | Med | notification-inbox incomplete without it |
| #34 | Notification preferences UI | M | Med | notification-inbox incomplete without it |
| #53 | Queue-board component | M | Med | needs design review |
| #26 | Data-table row and column spanning | M | Med | clinical regulatory grid |

## References

- Previous: `git show HEAD~1:HANDOFF.md`
