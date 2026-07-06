# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-06
**Branch:** `main` — #20 landed as `150a74f`

---

## Last Session

Closed #20 (queue board UX redesign). Brainstormed the orthogonal scope × perspective model, ran adversarial design review (4 rounds, 18 issues), implemented via 10-task subagent-driven development, squashed 20 commits to 1, pushed to fork and blessed repo. Also drove pages#116 (push-down primitives) as a dependency — pages now ships filter-chips, scope-selector, a11y mixins, schema-form, and event helpers.

## Immediate Next Step

Start #22 — build the data table component (`<pages-data-table>`). This is the **top priority** — no other UI work should proceed until tables are in place. Every CaseHub app needs tables; they must all use the same one.

Build it here in blocks-ui first against the real work-item-inbox use case, then promote to pages-primitives when stable.

**Garden push still pending:** entries committed locally but not pushed. Retry `git -C ~/.hortora/garden push origin main`.

## What's Left

- #22 — Build pages-data-table: pagination, virtual scroll, configurable columns, sorting, row selection · L · High · **PRIORITY — blocks all other UI work**
- #16 — Minor findings from epic #4 final review · M · Low · some items resolved by #20, re-triage
- #21 — Token migration `--blocks-*` → `--pages-*` · M · Med · blocked by pages#112

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #22 | Build pages-data-table — pagination, virtual scroll, columns, sorting | L | High | **DO THIS FIRST.** Build in blocks-ui, promote to pages when stable |
| #16 | Clean up minor findings from epic #4 final review | M | Low | Re-triage — some items resolved by #20 |
| #21 | Token migration — `--blocks-*` → `--pages-*` | M | Med | Blocked by pages#112 |
| #8 | SLA / Deadline Indicator component | S | Low | Standalone primitive — needs table first |
| #9 | Audit Trail Viewer | M | Med | Needs table |
| #10 | Case Timeline (replace stub) | M | Med | |
| #11 | Trust Score Panel (replace stub) | M | Med | |
| #12 | KPI Metric Row | S | Low | |
| #13 | Approval Gate | S | Med | |

## Cross-Module

**Pages work completed (this session):**
- casehub-pages#114, #116 — token normalisation, wildcard patterns, event helpers, pages-primitives package

**Pages work deferred:**
- casehub-pages#129 — pages-data-table deferred; building in blocks-ui#22 first, then promoting

**Pages work remaining (from prior sessions):**
- casehub-pages#109 — ConfigurablePanel interface export · XS
- casehub-pages#64 — dockBar() layout primitive · M
- casehub-pages#110 — Dataset pipeline bridge to hostPanel · M
- casehub-pages#112 — blocks-ui token migration support
