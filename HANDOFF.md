# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-08
**Branch:** `main` — #40 approval-gate response payload landed
**Priority:** Core data-table and approval-gate features complete. Remaining open issues are cross-repo migrations, domain-specific components, and notification UI.

---

## Last Session

Closed #40 — approval-gate `gate.decided` event now includes `serverData` (parsed response body) and error messages use server-provided text instead of bare HTTP status codes. Added `GateDecidedPayload` typed interface. Adversarial design review (4 rounds, 6 issues, $11) drove naming (`serverData` over `response`), typed interface, and edge case guards. One garden entry submitted (`.catch()` gotcha on undefined `json` method).

## Immediate Next Step

No trailing work. Pick from open backlog — #33 (subscription editor) and #34 (notification preferences) are the next component features; #26 (row/column spanning) is the next data-table feature.

## Cross-Module

**Blocked by:**
- `casehub-pages` — casehub-pages#145 gates #44 (DataSourceMixin migration). blocks-ui components use DataEndpointMixin now; swap to DataSourceMixin once pages ships the unified mixin.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #37 | AML case-workbench promotion — split-workbench + case-list-pane + case-detail-pane | L | Med | In design — AML feedback received |
| #44 | Migrate DataEndpointMixin → DataSourceMixin | S | Low | Blocked by casehub-pages#145 |
| #33 | Subscription editor component | M | Med | |
| #34 | Notification preferences and suppression UI | M | Med | |
| #26 | Data-table row and column spanning | M | Med | |
| #27 | Migrate PagesTable examples from pages-viz | L | Med | PagesTable deleted; example migration |
| #35 | Cross-repo component migration tracking | L | High | Epic — spans multiple repos |
