# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-10
**Branch:** `main` — no open work
**Priority:** #46 (pages-data-table rendering fix) is next — user requested it.

---

## Last Session

Implemented #45 — trust-score-panel trend sparkline. Built three new blocks-ui-core primitives: shared `renderSparkline`, `TrendPoint` + `extractTrendPoints` (safe TypedDataSet extraction), and `TrendSourceMixin` (reusable time-series trend pattern). Migrated kpi-metric-row to shared sparkline (fixed gradient ID collision). Examples page shows live simulated + static demos with play/pause. Design review: 4 rounds, 15 issues, all resolved ($14.19). Filed #46 for pre-existing pages-data-table rendering issue in trust-score-panel capability breakdown and audit-trail-viewer.

## Immediate Next Step

Pick up #46 — `pages-data-table` shows "No data" in trust-score-panel Per-Capability Breakdown and Audit Trail Viewer examples despite mock data loading correctly. The gauge renders (data arrives), but the table is empty. Likely a data-table rendering issue with `.data`/`.columns` property binding.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #46 | pages-data-table shows "No data" in trust-score-panel and audit-trail-viewer examples | S | Med | Pre-existing; mock data loads but table empty |
| #33 | Subscription editor component | M | Med | |
| #34 | Notification preferences and suppression UI | M | Med | |
| #26 | Data-table row and column spanning | M | Med | |
| #27 | Migrate PagesTable examples from pages-viz | L | Med | PagesTable deleted; example migration |
| #35 | Cross-repo component migration tracking | L | High | Epic — OpenClaw, Clinical, DevTown, Claudony remaining |
