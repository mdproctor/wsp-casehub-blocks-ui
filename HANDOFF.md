# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-13
**Branch:** `main`
**Priority:** #47 is now blocked on casehub-pages#172. Pick up #50 or another issue.

---

## Last Session

Designed row-detail expansion for `pages-table` — the missing UX pattern that makes audit-trail-viewer's row click actually work. Spec written and committed to pages repo (`docs/specs/2026-07-13-row-detail-expansion-design.md`), issue filed as casehub-pages#172. blocks-ui#47 commented with the dependency. No code changes to blocks-ui this session — this was pure design work.

## Immediate Next Step

Pick up #50 — recover 43 PagesTable tests for TypedDataSet integration.

## What's Left

- #47 — audit-trail-viewer row expand. Blocked by casehub-pages#172. Once pages implements `getRowDetail`, the fix here is passing a callback. · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #50 | Recover 43 PagesTable tests for TypedDataSet integration | M | Med | Filed during design review |
| #33 | Subscription editor component | M | Med | |
| #34 | Notification preferences and suppression UI | M | Med | |
| #26 | Data-table row and column spanning | M | Med | |
| #35 | Cross-repo component migration tracking | L | High | Epic |

## References

- Spec (pages repo): `docs/specs/2026-07-13-row-detail-expansion-design.md`
- Pages issue: casehub-pages#172
- Blog: `blog/2026-07-13-mdp01-the-missing-ux-pattern.md`
- Previous: `git show HEAD~1:HANDOFF.md`
