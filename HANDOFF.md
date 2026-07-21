# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-21
**Branch:** `main`
**Priority:** CI red (yarn.lock), stale epics to close, stashed channel-activity changes to review.

---

## Last Session

Closed #86 (channel-activity topic showcase — XS). Investigated case-explorer blocking — the infinite loop fix was already applied in a prior session; verified page loads and row selection work correctly in dev mode. Created work slot 12 for #33/#34 (subscription editor + notification preferences). Investigated CI failure — yarn.lock out of date; lockfile sync now automated in git-commit skill (Step 1d).

## Immediate Next Step

Fix CI: run `yarn install` on main and push the updated `yarn.lock`. The git-commit skill's lockfile sync (Step 1d) will prevent recurrence, but the current main needs a one-time fix for commits that landed before the skill step existed.

**Stashed changes:** `git -C /Users/mdproctor/claude/casehub/blocks-ui stash list` — four stashes including channel-activity design review changes (channel-feed threaded view removal, topic-bar simplification), unrelated design review agent modifications, and subscription editor fragments. Review and either commit or drop.

## What's Left

- CI red — yarn.lock out of date, one-time fix needed · XS · Low
- Stashed changes — 4 stashes to review/triage · XS · Low
- Channel-activity gap analysis — connectors has 30 qhorus UI files diverged · M · Med
- Parent deep-dive doc (`docs/repos/casehub-blocks-ui.md`) stale · S · Low
- vitest pages-component source/dist mismatch — 1 test failure in case-explorer.test.ts · S · Med
- #81 (schema-form epic) — all children CLOSED, epic should be closed · XS · Low
- #41 (devtown migration) — Phase 2 complete from blocks-ui side, epic stale · XS · Low
- Slot 8 (trust-workbench) — #89 CLOSED, slot ready to remove · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #33 | Subscription editor | M | Med | Slot 12 active, unblocked |
| #34 | Notification preferences UI | M | Med | Slot 12 active, unblocked |
| #88 | pages-modal duplicate CE crash | S | Med | chat-app blocker |
| — | Engine EntityStateContributor SPI | L | High | Engine-session work |

## References

- Previous: `git show HEAD~1:HANDOFF.md`
