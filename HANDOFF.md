# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-14
**Branch:** `main`
**Priority:** #55 delivered — blocks-timeline replaces case-timeline. Epic audit of #35 child epics done. CI fix pushed. Next: pick up #33, #34, or remaining app delivery items from #56.

---

## Last Session

Major delivery: unified pluggable timeline (`<blocks-timeline>`) replacing `<case-timeline>`. Strategy pattern with two shipped strategies (event chronology, state progression), three layout modes (vertical, horizontal, compact), render callback resolution, temporal weighting, staggered axis labels. 174 tests. Three example pages. Design-reviewed (3 adversarial rounds, 14 issues resolved). Also audited all five child epics under #35 (cross-repo migration tracking) — updated #36 (OpenClaw), #38 (Clinical), #41 (DevTown), filed missing component issues (#53 queue-board, #54 routing-rationale), created app delivery epic #56. Fixed CI workflow token issue. Updated audit-trail-viewer example page.

## Immediate Next Step

Pick up #33 (subscription editor) or #34 (notification preferences) — both are M/Med, independent, and needed by notification-inbox consumers.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #33 | Subscription editor component | M | Med | notification-inbox incomplete without it |
| #34 | Notification preferences and suppression UI | M | Med | notification-inbox incomplete without it |
| #54 | Routing rationale component | M | Med | DevTown trust visibility |
| #26 | Data-table row and column spanning | M | Med | clinical regulatory grid |
| #27 | PagesTable migration to pages-primitives | M | Med | cleanup |
| #53 | Queue-board component | M | Med | single consumer (DevTown), needs design review |

## References

- Spec: `docs/specs/2026-07-13-blocks-timeline-design.md`
- Plan: `docs/plans/2026-07-14-blocks-timeline.md`
- App delivery epic: #56
- Previous: `git show HEAD~1:HANDOFF.md`
