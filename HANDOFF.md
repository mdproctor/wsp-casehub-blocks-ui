# HANDOFF — casehub-blocks-ui

*Updated: #46 closed — removed from backlog.*

**Date:** 2026-07-11
**Branch:** `issue-49-typeddataset-native` — #49 in progress (Task 8 remaining)
**Priority:** Task 8 — migrate 5 remaining consumers to TypedDataSet + pages-table.

---

## Last Session

Two sessions on #49. Session 1: designed the unified pipeline (fetch is just push), ran adversarial design review (3 rounds, 16 issues, $14.19), implemented pages foundation changes (Tasks 1-3 locally). Session 2: pages session landed Tasks 1-4 (casehub-pages#152). This session implemented blocks-ui Tasks 5-7: fetchSource pipeline integration, DataSourceAdapter/Mixin type propagation, list-pane migration. All blocks-ui-core tests pass (116). list-pane tests pass (18).

## Immediate Next Step

Continue on branch `issue-49-typeddataset-native`. Run `/work` — it will detect the existing `.meta` and resume. Then implement **Task 8: migrate remaining consumers**. The plan is at `plans/2026-07-11-typeddataset-native.md` (workspace).

## Task 8 — What Needs Doing

Five consumers + vitest configs + examples. Each follows the same pattern as list-pane:

**For each consumer:**
1. Replace `import type { ColumnDef } from '@casehubio/pages-data-table'` → remove (ColumnDef deleted)
2. Replace `import '@casehubio/pages-data-table'` → `import '@casehubio/pages-table'`
3. Import `TableColumnConfig`, `ColumnRenderer` from `@casehubio/pages-table`
4. Import `TypedRow`, `ColumnId`, `CellValue`, `columnId`, `ColumnType` from `@casehubio/pages-data/dist/dataset/types.js`
5. Replace `ColumnDef<DomainType>[]` columns with `columnConfig: TableColumnConfig[]` + `columnRenderers: Map<ColumnId, ColumnRenderer>`
6. Replace `<pages-data-table .rows= .columns=>` → `<pages-table .dataSet= .columnConfig= .columnRenderers=>`
7. For components with DataSourceMixin: remove manual `willUpdate` extraction from `unknown` — `this.dataSet` is now `TypedDataSet | undefined`
8. For components building in-memory data (trust-score-panel): use `fromRows()` from `@casehubio/pages-data/dist/dataset/conversion.js`
9. Update tests: mock fetches need `headers: new Headers(...)`, assertions change from `table.rows` to `table.dataSet.rows`

**Consumer-specific notes:**

| Consumer | Special handling |
|----------|-----------------|
| `trust-score-panel` | Uses `fromRows()` to build TypedDataSet from capability scores. Has existing bugs: wrong ColumnDef field names (`key`/`header` instead of `id`/`label`). Fix as part of migration. Uses `@row-click` event (may need updating to `@row-activate`). |
| `audit-trail-viewer` | Uses `DataSourceAdapter` (not mixin). Passes `.data=` (wrong prop name — was always broken). Has domain filtering via `_filteredEntries()` that operates on `LedgerEntry[]` — needs to filter `TypedRow[]` instead. Custom renderers for timestamp, actor, digest columns. |
| `work-item-inbox` | Most complex. Direct HTTP fetch (not DataSourceMixin). `ColumnDef<WorkItemRootResponse>` with `getValue` extractors and custom renderers (status pills, relative time). Needs `fromRows()` or column declarations on its data source. Has tree-table support via `getChildren`. |
| `notification-inbox` | Similar to work-item-inbox. Custom renderers for notification badges, timestamps. |
| `subscription-list` | Simpler consumer. Custom column rendering. |

**vitest.config.ts updates needed:**

Every component's vitest.config.ts that has `@casehubio/pages-data-table` alias needs changing to `@casehubio/pages-table` pointing to `../../../pages/packages/pages-table/src`. Components: trust-score-panel, audit-trail-viewer, work-item-inbox, work-item-workbench, notification-inbox, split-workbench, detail-pane, approval-gate, case-timeline, kpi-metric-row.

Also: `examples/vite.config.ts`, `examples/vitest.config.ts`, `examples/src/pages/data-table-page.ts`.

**Known issue:** Empty array `[]` through extractDataSet throws EMPTY_RESULT instead of producing empty TypedDataSet (garden GE-20260711-5170ee). Consumers showing empty state need to handle this — either don't set endpoint until data expected, or treat EMPTY_RESULT error as empty state.

## What's Done (don't redo)

- ✅ Task 1: ExtractionDef type (pages-data) — committed in pages
- ✅ Task 2: fromRows() + SnapshotEvent.totalRows (pages-data) — committed in pages
- ✅ Task 3: DataReceiver.dataSet: TypedDataSet, SourceFactory extension (pages-component) — committed in pages
- ✅ Task 4: pages-data-table → pages-table redesign (pages) — committed in pages, 83 tests
- ✅ Task 5: fetchSource pipeline integration (blocks-ui-core) — committed on this branch
- ✅ Task 6: DataSourceAdapter/Mixin type propagation (blocks-ui-core) — committed on this branch
- ✅ Task 7: list-pane migration (blocks-ui) — committed on this branch
- ✅ CLAUDE.md updated (pages-data-table → pages-table references)

## After Task 8

Once all consumers are migrated and tests pass:
- Run `yarn typecheck` across blocks-ui to catch any remaining type errors
- Invoke `work-end` — it handles code review, squash, push, and branch closure
- work-end will also check for deferred concerns to file as issues

## What's Next (after #49)

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #50 | Recover 43 PagesTable tests for TypedDataSet integration | M | Med | Filed during design review |
| #33 | Subscription editor component | M | Med | |
| #34 | Notification preferences and suppression UI | M | Med | |
| #26 | Data-table row and column spanning | M | Med | |
| #35 | Cross-repo component migration tracking | L | High | Epic |

## References

- Spec: `docs/specs/2026-07-11-typeddataset-native-design.md`
- Plan: workspace `plans/2026-07-11-typeddataset-native.md`
- Blog: workspace `blog/2026-07-11-mdp01-fetch-is-just-push.md`, `blog/2026-07-11-mdp02-the-pipeline-closes.md`
- Journal: workspace `design/JOURNAL.md` (§Session-1, §Session-2)
- Design review: `~/adr/blocks-ui/typeddataset-native-*/tracker.md`
- Pages issue: casehub-pages#152 (landed)
- Garden: GE-20260711-190f40 (ide_edit_member gotcha), GE-20260711-5170ee (empty array gotcha)
