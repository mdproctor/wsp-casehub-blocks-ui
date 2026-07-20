# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-20
**Branch:** `main`
**Priority:** Case explorer (#87) landed. Examples page broken — needs investigation before visual testing.

---

## Last Session

Closed #80 (NotificationApi.getEventTypes — XS). Recovered 4 blog entries from closed branches. Designed, reviewed, and implemented #87 — composable case explorer (XL). 8-round adversarial design review (33 issues, all resolved). 15 source files, 78 tests, 5 presets, 5 convenience wrappers. Squashed to 1 commit, pushed to both remotes.

## Immediate Next Step

Fix the examples page hang. `initMockState()` in `examples/src/main.ts` blocks in the browser — the case-explorer example page and all other examples are inaccessible. Likely a pages dependency change broke the mock fetch/SSE infrastructure. The `DataSourceController.connect()` method throws `this.controller.connect is not a function` in jsdom tests (visible as stderr) — same root cause may apply in the browser. Check if pages packages changed recently (`git -C ../pages log --oneline -5`). The case-explorer components work in tests (78 pass) — the hang is in the examples mock infrastructure, not the components.

**Stashed changes:** `git -C /Users/mdproctor/claude/casehub/blocks-ui stash list` — two stashes containing unrelated modifications from design review agents (channel-feed.ts `.collapsed`, routing-rationale PHASE_STYLES export) plus two test HTML files (case-explorer-standalone.html, test-minimal.html). Review and either commit or drop.

## What's Left

- Examples page hang — `initMockState()` blocks, all examples inaccessible · S · Med
- Channel-activity gap analysis — connectors has 30 qhorus UI files diverged · M · Med
- Parent deep-dive doc (`docs/repos/casehub-blocks-ui.md`) stale · S · Low
- Engine REST endpoint for routing decision data · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #33 | Subscription editor | M | Med | Blocked by pages #81 |
| #34 | Notification preferences UI | M | Med | Blocked by pages #81 |
| — | Engine EntityStateContributor SPI | L | High | Engine-session work, contracts defined in #87 spec |
| — | DevTown Phase 1 consume | L | Med | Governance views → blocks-ui components |

## References

- Spec: `docs/specs/2026-07-20-composable-case-explorer-design.md`
- Plan: workspace `plans/2026-07-20-composable-case-explorer.md`
- Design review workspace: `~/adr/casehub-blocks-ui/composable-case-explorer-20260720-024642/`
- Previous: `git show HEAD~1:HANDOFF.md`
