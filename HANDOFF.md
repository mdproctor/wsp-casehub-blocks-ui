# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-17
**Branch:** `main`
**Priority:** #67 and #54 landed. Shadow DOM CSS scoping fix + routing-rationale component. 23 packages.

---

## Last Session

Fixed #67 — column renderers in similarity-panel, compliance-summary, and work-item-inbox used CSS classes that couldn't reach pages-table's shadow DOM. Root cause: shadow DOM CSS scoping. Fix: inline styles. Formalised as protocol rule (PP-20260713-8ea1af) and garden entry (GE-20260717-4618a1).

Built #54 — routing-rationale component. Trust-weighted routing decision explanation: score header with threshold/margin visualization, alternatives table with phase badges, policy summary. Data contract maps to engine's TrustCandidateClassifier + TrustRoutingPolicy. Dual-data mode. 20 tests. Full adversarial design review (5 rounds, 25 issues).

## Immediate Next Step

Pick up next issue from What's Next. Run `/work` to start.

## What's Left

- Channel-activity gap analysis — connectors has 30 qhorus UI files diverged from blocks-ui's channel-activity. Needs full diff before migration. · M · Med
- Parent deep-dive doc (`docs/repos/casehub-blocks-ui.md`) stale — references DataEndpointMixin (renamed), missing new components · S · Low
- casehub-pages#196 — table enhancements: interstitial hooks, legend component, rowAccent, nested grouping · M · Med
- Engine REST endpoint for routing decision data — blocks endpoint mode of routing-rationale. Tracked as deferred issue in spec. · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #33 | Subscription editor | M | Med | notification-inbox incomplete without it |
| #34 | Notification preferences UI | M | Med | notification-inbox incomplete without it |
| #26 | Data-table row and column spanning | M | Med | clinical regulatory grid |

## References

- Previous: `git show HEAD~1:HANDOFF.md`
