# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-08
**Branch:** `main` — #37 generic workbench components landed
**Priority:** Three new layout primitives shipped. AML migrated. Other domain repos can now adopt.

---

## Last Session

Designed, reviewed (6 rounds, $34), and built three generic workbench components: `<split-workbench>`, `<list-pane>`, `<detail-pane>`. Refactored `<work-item-workbench>` to use `<split-workbench>` internally (-400 lines). Migrated AML to consume the generic components — deleted local case-workbench (-535 lines). Filed casehub-pages#145 for DataSource pipeline improvements.

## Immediate Next Step

No trailing work. Pick from open backlog — #44 is blocked by pages, #33/#34 are the next component features.

## Cross-Module

**Blocked by:**
- `casehub-pages` — casehub-pages#145 gates #44 (DataSourceMixin migration). blocks-ui components use DataEndpointMixin now; swap to DataSourceMixin once pages ships the unified mixin.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #44 | Migrate DataEndpointMixin → DataSourceMixin | S | Low | Blocked by casehub-pages#145 |
| #33 | Subscription editor component | M | Med | |
| #34 | Notification preferences and suppression UI | M | Med | |
| #26 | Data-table row and column spanning | M | Med | |
| #27 | Migrate PagesTable examples from pages-viz | L | Med | PagesTable deleted; example migration |
| #35 | Cross-repo component migration tracking | L | High | Epic — OpenClaw, Clinical, DevTown, Claudony remaining |
