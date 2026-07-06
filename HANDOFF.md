# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-06
**Branch:** `main` — #22 landed as `0e3b44b`

---

## Last Session

Closed #22. Built pages-data-table: standalone Lit Web Component with ColumnDef<R> data model, three display modes (auto/paginated/scroll), CSS Grid rendering, virtual scroll engine, multi-mode selection, client-side sorting, column visibility, 2D keyboard navigation, ARIA grid, CSS ::part() row styling. Design-reviewed (5 rounds, 19 issues). Implementation-reviewed (14 findings fixed). Visually verified in browser (6 rendering fixes). Refactored work-item-inbox to consume it (~170 lines removed). Filed #26-#31 as deferred items.

## Immediate Next Step

Pick next work from What's Next. #27 (pages migration) is the natural follow-on once the table is stable in production use.

## What's Left

- #27 — Migrate PagesTable examples and remove pages-viz table . L . High . blocked by table stabilisation
- #25 — kpi-metric-row endpoint change after mount should trigger re-fetch . S . Low
- #24 — Add density property to kpi-metric-row . S . Med
- #21 — Token migration --blocks-* to --pages-* . M . Med . blocked by pages#112
- #26 — Row and column spanning support . M . High
- #28 — Multi-column sort . S . Med
- #29 — Text filter support . S . Med
- #30 — Tree/expandable rows . M . High
- #31 — CSV export . S . Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Migrate PagesTable examples and remove pages-viz table | L | High | Promote to pages-primitives first |
| #25 | kpi-metric-row: reactive endpoint property | S | Low | Add willUpdate handler |
| #24 | kpi-metric-row: density property for compact grid | S | Med | Design property API |
| #21 | Token migration to --pages-* | M | Med | Blocked by pages#112 |
| #9 | Audit Trail Viewer | M | Med | Can use new table |
| #10 | Case Timeline (replace stub) | M | Med | |
| #11 | Trust Score Panel (replace stub) | M | Med | |

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*
