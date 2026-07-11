# Design Journal — issue-49-typeddataset-native

## §Session-1 — 2026-07-11: Pipeline unification design

**Key insight:** A fetch is just a single-event push. All data sources (HTTP, WS, SSE, inline, simulated) should produce TypedDataSet through the same extraction pipeline. Only the originator differs.

**Root cause:** `DataReceiver.dataSet: unknown` — the interface-level declaration in pages-component. `DataSourceController` already handles TypedDataSet internally. The `unknown` typing was the root of all downstream type loss.

**ColumnDef superseded:** The prior data-table spec's ColumnDef\<R\> model existed because fetchSource bypassed extraction — data arrived as raw JSON, consumers needed extractors. With pipeline unification, the extraction happens in the pipeline, not in getValue. The rendering contract splits into three independently-varying concerns: data schema (TypedDataSet.columns), cell rendering (columnRenderers), presentation config (TableColumnConfig).

**Cross-repo split:** Pages changes (Tasks 1-3) implemented locally, Task 4 (table redesign) deferred to pages session. Filed casehub-pages#152. blocks-ui Tasks 5-8 depend on pages completion.

**Design review:** 3 rounds, 16 issues, 15 verified, 1 accepted, $14.19. Key additions: ExtractionDef type narrowing, fromRows() factory for in-memory domain data, SnapshotEvent.totalRows for pagination, client-sort/filter mechanics with TypedRow.

## §Session-2 — 2026-07-11: Implementation — pipeline integration and first consumer

**Pages session completed Tasks 1-4:** ExtractionDef, fromRows(), SnapshotEvent.totalRows, DataReceiver type fix, SourceFactory extension, and full table redesign (pages-data-table → pages-table, 83 tests). casehub-pages#152 landed.

**blocks-ui Tasks 5-7 implemented this session:**
- fetchSource rewritten to use extractDataSet — raw `as never` cast deleted, real TypedDataSet with working TypedRow accessors produced. navigatePath utility for totalPath extraction.
- DataSourceAdapter and DataSourceMixin typed as `TypedDataSet | undefined`, createSourceFactory passes columns/dataPath/totalPath through.
- list-pane migrated: removed ColumnDef/rows extraction, passes TypedDataSet directly to pages-table. Added columnConfig and columnRenderers properties.

**Discovery:** Empty array `[]` through extractDataSet throws EMPTY_RESULT instead of producing empty TypedDataSet. Garden entry GE-20260711-5170ee filed. Consumer workaround: don't set endpoint until data is expected, or handle EMPTY_RESULT as empty state.

**Remaining:** Task 8 — migrate trust-score-panel, audit-trail-viewer, work-item-inbox, notification-inbox, subscription-list, vitest configs, examples. Mechanical — same pattern as list-pane migration.
