# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-06
**Branch:** `main` — #22 landed as `0e3b44b`
**Priority:** Full pages/blocks-ui consolidation — eliminate all duplication and technical debt.

---

## Last Session

Closed #22. Built pages-data-table (ColumnDef\<R\>, 3 display modes, CSS Grid, virtual scroll, selection, sorting, ARIA, keyboard nav). Refactored work-item-inbox to consume it. Design-reviewed (5 rounds), implementation-reviewed (14 fixes), visually verified.

## Immediate Next Step

Start #21 — the token migration is now **unblocked** (pages#112 and pages#116 are both CLOSED). This is the foundation for everything else. Run `/work` to begin.

---

## Consolidation Roadmap — Priority Order

The goal: blocks-ui depends on pages packages for all shared infrastructure. No duplicated tokens, mixins, event helpers, schema-form, or dataset contracts. The old PagesTable is removed. Every blocks-ui component uses `--pages-*` tokens.

### Phase 1: Eliminate blocks-ui-core duplication (#21)

**#21 is now unblocked.** pages#112 (tokens) and pages#116 (primitives push-down) are both closed. All canonical packages exist in pages.

| What to migrate | From (blocks-ui-core) | To (pages) | Scope |
|----------------|----------------------|-----------|-------|
| CSS tokens (colours, themes, spacing, typography) | `tokens/` (3 files) | `@casehubio/pages-ui-tokens` | Delete blocks-ui copy, add dependency |
| CSS var prefix | `var(--blocks-*` (561 occurrences, 15 files) | `var(--pages-*` | Mechanical rename |
| A11y mixins | `mixins/` (4 files — stale copies) | `@casehubio/pages-primitives` | Delete blocks-ui copies. pages-primitives is the superset (directional roving, disconnect cleanup) |
| Schema form | `schema-form/` | `@casehubio/pages-primitives` | Delete blocks-ui copy |
| emitPagesEvent / onPagesEvent | `types/events.ts` | `@casehubio/pages-component` | Move event helpers to pages import. Keep `WorkItemEventTopics` and payload types in blocks-ui-core (domain-specific) |
| DatasetContract | `dataset-contract.ts` | `@casehubio/pages-data` | Delete blocks-ui copy |
| BlocksTheme | `theme.ts` | Not needed — components use CSS vars directly | Delete |

**After #21:** blocks-ui-core contains only domain-specific types (WorkItemResponse, events, identity), SharedTimerController, and blocks-confirm-dialog. Everything else comes from pages.

**Known gotcha:** Spacing scale key format differs (`'0.5'` vs `'0-5'`). Must resolve when switching to pages-ui-tokens.

Scale: L · Complexity: Med (mechanical but wide blast radius — 15 files, 561 var renames)

### Phase 2: Table feature parity + promotion (#27, #29-31)

Before PagesTable can be removed from pages-viz, pages-data-table needs feature parity:

| # | Feature | Scale | Complexity | Blocks PagesTable removal |
|---|---------|-------|------------|--------------------------|
| #29 | Text filter support | S | Med | Required — PagesTable has toolbar filter |
| #30 | Tree/expandable rows | M | High | Required — PagesTable has tree mode |
| #31 | CSV export | S | Low | Required — PagesTable has csv export |
| #26 | Row and column spanning | M | High | Not required for parity — PagesTable doesn't have it |
| #28 | Multi-column sort | S | Med | Not required for parity |

After parity: promote pages-data-table to pages-primitives (pages#129), build the TypedDataSet→ColumnDef bridge in pages-runtime, migrate all ~20 YAML dashboard examples (#27), remove PagesTable from pages-viz (890 lines + 1,975 lines of tests).

### Phase 3: Cleanup dead code

| What | Why | Scale |
|------|-----|-------|
| Delete `components/notification-inbox/` directory | Orphaned — zero source files, no package.json, only stale build artifacts | XS |
| Remove `components/work-item-row/` | Legacy — inbox uses data-table now. Remove example page, vite config alias, tsconfig reference | S |
| Close or fix #19 (queue-board SSE bug) | No queue-board component exists — issue may be stale | XS |
| Close #3 (issue-workflow setup) and #2 (build infra) and #1 (CI workflow) | Setup chores — already done or superseded | XS |

### Phase 4: Foundation work (pages side)

| # | What | Repo | Scale | Complexity |
|---|------|------|-------|------------|
| pages#109 | Export ConfigurablePanel interface | pages | S | Low |
| pages#110 | Bridge dataset pipeline to hostPanel | pages | M | Med |
| pages#111 | Foundation work for blocks-ui hosting (epic) | pages | L | Med |

These enable blocks-ui components to be hosted inside pages dashboards natively.

### Phase 5: Stub implementations

| # | What | Scale | Complexity | Notes |
|---|------|-------|------------|-------|
| #9 | Audit Trail Viewer | M | Med | Can use pages-data-table |
| #10 | Case Timeline (replace stub) | M | Med | |
| #11 | Trust Score Panel (replace stub) | M | Med | |

These are currently 2-line stubs. Lower priority than consolidation.

---

## What's Left (all open issues, prioritised)

| # | Description | Scale | Complexity | Phase | Blocked by |
|---|-------------|-------|------------|-------|-----------|
| **#21** | **Token migration `--blocks-*` → `--pages-*`** | **L** | **Med** | **1** | **UNBLOCKED** |
| #32 | Refactor inbox to use pages-data-table (fully) | S | Low | 1 | — |
| #29 | pages-data-table: text filter | S | Med | 2 | — |
| #30 | pages-data-table: tree/expandable rows | M | High | 2 | — |
| #31 | pages-data-table: CSV export | S | Low | 2 | — |
| #27 | Migrate PagesTable examples, remove from pages-viz | L | High | 2 | #29, #30, #31 |
| #25 | kpi-metric-row: reactive endpoint | S | Low | 3 | — |
| #24 | kpi-metric-row: density property | S | Med | 3 | — |
| #19 | queue-board SSE handler bug | XS | Low | 3 | Verify if stale |
| #18 | work-item-detail: fetch relatedItems | S | Med | 3 | — |
| #26 | pages-data-table: row/column spanning | M | High | 5 | — |
| #28 | pages-data-table: multi-column sort | S | Med | 5 | — |
| #9 | Audit Trail Viewer | M | Med | 5 | — |
| #10 | Case Timeline | M | Med | 5 | — |
| #11 | Trust Score Panel | M | Med | 5 | — |

## Cross-Module

**We're blocking** (other modules waiting on us):
- `pages` — pages#129 (add pages-data-table) waits for blocks-ui validation · L · High
- `pages` — pages#111 (foundation for hosting) is independent but enables blocks-ui embedding

**Blocked by:**
- Nothing — pages#112 and pages#116 are both CLOSED. All migration work is unblocked.
