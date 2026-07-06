# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-06
**Branch:** `main` — #20 landed as `150a74f`

---

## Last Session

Closed #20 (queue board UX redesign). Brainstormed the orthogonal scope × perspective model, ran adversarial design review (4 rounds, 18 issues, $14.73), implemented via 10-task subagent-driven development, squashed 20 commits to 1, pushed to fork and blessed repo. Also drove pages#116 (push-down primitives) as a dependency — pages now ships filter-chips, scope-selector, a11y mixins, schema-form, and event helpers.

## Immediate Next Step

Pick up #16 (minor cleanup from epic #4 review) or #22 (adopt pages-data-table). Run `yarn examples` → "Queue + Inbox" to see the new UX.

**Garden push still pending:** 4+ entries committed locally but not pushed. Retry `git -C ~/.hortora/garden push origin main`.

## What's Left

- #16 — Minor findings from epic #4 final review (10 items: dead IntersectionObserver code now removed by #20, schema-form type safety, missing aria-controls, etc.) · M · Low
- #22 — Adopt pages-data-table for pagination, virtual scroll, configurable columns, sorting · M · Med
- #21 — blocks-ui token migration from `--blocks-*` to `--pages-*` · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #16 | Clean up minor findings from epic #4 final review | M | Low | Some items resolved by #20 (dead IntersectionObserver removed) — re-triage |
| #22 | Adopt pages-data-table for work item lists | M | Med | Blocked by casehub-pages#129 (pages-data-table component) |
| #21 | Token migration — `--blocks-*` → `--pages-*` | M | Med | Blocked by pages#112 |
| #8 | SLA / Deadline Indicator component | S | Low | Standalone primitive |
| #9 | Audit Trail Viewer | M | Med | |
| #10 | Case Timeline (replace stub) | M | Med | |
| #11 | Trust Score Panel (replace stub) | M | Med | |
| #12 | KPI Metric Row | S | Low | |
| #13 | Approval Gate | S | Med | |

## Cross-Module

**Pages work completed (this session):**
- casehub-pages#114, #116 — token normalisation, wildcard patterns, event helpers, pages-primitives package (filter-chips, scope-selector, a11y mixins, schema-form)

**Pages work filed (this session):**
- casehub-pages#129 — pages-data-table component (pagination, virtual scroll, configurable columns, sorting)

**Pages work remaining (from prior sessions):**
- casehub-pages#109 — ConfigurablePanel interface export · XS
- casehub-pages#64 — dockBar() layout primitive · M
- casehub-pages#110 — Dataset pipeline bridge to hostPanel · M
- casehub-pages#112 — blocks-ui token migration support
