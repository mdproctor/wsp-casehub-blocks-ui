# HANDOFF — casehub-blocks-ui

**Branch:** issue-95-align-table-pipeline (closed)
**Date:** 2026-07-25
**Issues:** casehubio/blocks-ui#95

## What landed

Aligned all 10 table components with the pages pipeline (`DataSource → SourceConnector → DataSourceController`). Components now use `DataSourceMixin` or `DataSourceAdapter` for data lifecycle instead of manual fetch/fromRows/loading state.

**Per-component:**
- compliance-summary, similarity-panel: +client-sort
- routing-rationale: fixed endpoint-path renderer bug (renderers only built for property path), +client-sort
- trust-score-panel: removed duplicate fromRows in render, use pipeline dataSet, +client-sort
- trust-workbench: declarative list-pane binding replacing imperative _syncListPane
- audit-trail-viewer: pipeline delivers real dataset (was pushing empty), +client-sort
- preferences-editor: adopted DataSourceAdapter (was manual _loading/_error/_dataSet)
- work-item-inbox: extracted fromRows from render path into @state() (SSE lifecycle too complex for full adapter adoption)
- list-pane, grouped-data-view: verified aligned, no changes

**Stats:** 14 files, 192 insertions, 99 deletions, 316 tests passing

## What's left

- Replace portal `@casehubio/pages-form` links with published version refs before release · XS · Low
- Update `docs/repos/casehub-blocks-ui.md` in parent repo with new component descriptions · S · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #94 | Rename all components to use `blocks-` prefix consistently | M | Med | Breaking change |
| #93 | Migrate native HTML elements to `@casehubio/pages-ui-components` | S | Low | |
| #88 | pages-modal duplicate CustomElementRegistry crash in esbuild bundle | S | Med | Bug |
| #85 | Column resizing interaction with single-grid model | M | High | |
| #84 | Variable row heights with cell spanning | L | High | |
