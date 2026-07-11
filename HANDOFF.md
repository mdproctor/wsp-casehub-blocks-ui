# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-11
**Branch:** `issue-49-typeddataset-native` — #49 in progress
**Priority:** Blocked on pages session (casehub-pages#152) for Task 4 (table redesign).

---

## Last Session

Designed and began implementing #49 — TypedDataSet native. Core insight: fetch is just a single-event push — all data sources (HTTP, WS, SSE, inline, simulated) should flow through the same extraction pipeline. Design review: 3 rounds, 16 issues, all resolved ($14.19). Pages foundation changes (Tasks 1-3) implemented locally but uncommitted in pages repo (peer repo). Filed casehub-pages#152 for pages session.

## Immediate Next Step

Start a **pages session** — paste the prepared instructions from the blocks-ui session (see blocks-ui issue #49 comments or this session's conversation). The pages session commits Tasks 1-3 and implements Task 4 (table redesign: pages-data-table → pages-table). After pages lands, return here for Tasks 5-8.

## Cross-Module

**Blocked by:**
- `pages` — casehub-pages#152: table redesign + type foundations must land before blocks-ui Tasks 5-8 can proceed · L · High

## What's Left

- Tasks 5-8 of the implementation plan — fetchSource pipeline integration, adapter/mixin types, list-pane migration, remaining consumer migration · L · Med (mechanical once pages lands)
- Pages repo has 9 uncommitted files from Tasks 1-3 — the pages session must commit these first

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #49 | TypedDataSet integration — Tasks 5-8 (blocks-ui side) | L | Med | Blocked on casehub-pages#152 |
| #50 | Recover 43 PagesTable tests for TypedDataSet integration | M | Med | Filed during design review |
| #46 | pages-data-table shows "No data" in trust-score-panel/audit-trail-viewer | S | Med | Paused on stack; may be fixed by #49 migration |
| #33 | Subscription editor component | M | Med | |
| #34 | Notification preferences and suppression UI | M | Med | |
| #26 | Data-table row and column spanning | M | Med | |
| #35 | Cross-repo component migration tracking | L | High | Epic |

## References

- Spec: `docs/specs/2026-07-11-typeddataset-native-design.md`
- Plan: workspace `plans/2026-07-11-typeddataset-native.md`
- Blog: workspace `blog/2026-07-11-mdp01-fetch-is-just-push.md`
- Design review: `~/adr/blocks-ui/typeddataset-native-*/tracker.md`
- Pages issue: casehub-pages#152
