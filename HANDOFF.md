# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-14
**Branch:** `main`
**Priority:** CI green on casehubio/blocks-ui. All 16 packages published at 0.2.2. Pages also at 0.2.2. Gap analysis done for #38 clinical promotion — 6 issues filed (#58–#63). Channel-activity divergence with connectors identified but not yet addressed.

---

## Last Session

CI/build stabilisation: bumped all pages and blocks-ui packages to 0.2.2 (Maven alignment), fixed cross-repo package auth (GH_PACKAGES_TOKEN secret + publishConfig.registry), resolved typecheck errors (exactOptionalPropertyTypes, IntersectionObserver mock, node types, stale tsconfig references), made vitest aliases conditional for CI. Closed #27 (PagesTable migration — blocks-ui side done). Full gap analysis on #38 clinical promotion — all 6 components assessed, 3 complementary pairs analysed (all SEPARATE), 6 issues filed. Also identified channel-activity divergence: blocks-ui has the extracted component but connectors still has 30 files of its own copy, never switched over.

## Immediate Next Step

Pick up #54 (routing-rationale) — M/Med, DevTown priority. Run `/work` to start.

## What's Left

- Channel-activity gap analysis — connectors has 30 qhorus UI files diverged from blocks-ui's channel-activity. Needs full diff before migration. · M · Med
- #38 clinical promotion — 6 issues filed (#58–#63), ready when clinical is free for writes · L · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #54 | Routing rationale component | M | Med | DevTown priority |
| #33 | Subscription editor | M | Med | notification-inbox incomplete without it |
| #34 | Notification preferences UI | M | Med | notification-inbox incomplete without it |
| #58 | Commitment-lifecycle strategy | S | Low | blocks-timeline strategy, not new component |
| #59 | CBR precedents panel | S | Low | standalone, from clinical |
| #60 | Trust feedback display | S | Low | complements trust-score-panel |
| #61 | Compliance summary | XS | Low | renamed from regulatory-compliance-summary |
| #62 | GDPR erasure action | M | Low | coordinates with audit-trail-viewer |
| #63 | SLA breach policy | S | Low | complements sla-indicator |
| #53 | Queue-board component | M | Med | needs design review |
| #26 | Data-table row and column spanning | M | Med | clinical regulatory grid |

## References

- Previous: `git show HEAD~1:HANDOFF.md`
