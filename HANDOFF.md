# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-10
**Branch:** `main` — no open work
**Priority:** #45 (trust-score-panel trend sparkline) is next.

---

## Last Session

Migrated all DataEndpointMixin consumers to DataSourceMixin + DataSourceAdapter wrapping pages' DataSourceController (#44). Two-layer Lit binding: DataSourceAdapter (ReactiveController) as the reusable unit, DataSourceMixin as convenience sugar. Added fetchSource for raw JSON domain components, renderPropertyTree for human-readable payload rendering. Design review: 16 issues raised, all resolved. Four components migrated (list-pane, trust-score-panel, case-timeline, audit-trail-viewer), DataEndpointMixin deleted. Examples app fixed — mock-fetch centralised, vitest configs updated with dist/ aliases. Filed #45 for trust-score-panel trend sparkline.

## Immediate Next Step

Pick up #45 — add trend sparkline to trust-score-panel using `simulated()` data source from pages-data. The DataSource pipeline (`simulated`, `inlineSource`, `ScenarioController`) is designed for exactly this. Use a second DataSourceAdapter on the panel for trend data.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #45 | Trust-score-panel trend sparkline with simulated data source | S | Med | Just filed — uses simulated() from pages-data |
| #33 | Subscription editor component | M | Med | |
| #34 | Notification preferences and suppression UI | M | Med | |
| #26 | Data-table row and column spanning | M | Med | |
| #27 | Migrate PagesTable examples from pages-viz | L | Med | PagesTable deleted; example migration |
| #35 | Cross-repo component migration tracking | L | High | Epic — OpenClaw, Clinical, DevTown, Claudony remaining |
