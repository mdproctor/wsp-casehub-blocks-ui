# HANDOFF — casehub-blocks-ui

## Last Session

Drained the full plan queue (23/23 issues). Two batches of work:

**Batch 1 — Cleanup queue from LSP schema work (7 issues):**
Removed dead `computeSwfStackLayout` (730L), stripped debug scaffolding
from IIFE bundle, fixed `DiagramSyncListener` timer leak (Timer → 
ScheduledExecutorService + Disposer registration), extracted
`prepareSwfTaskInsert` helper, and fixed `moveSwfTask` to rewire ALL
`then:` references on move (was only rewiring one edge source). Two
issues (#193, #194) closed as wrong-repo — fixes belong in casehub-pages.

**Batch 2 — Codebase audit (8 issues):**
Fixed 5 broken index.ts exports (wrong class names), removed stale
`ElkAlgorithm` re-export from graph-stencil-org, filled Props interface
gaps (GroupedDataViewProps, WorkItemDetailProps), fixed
`_handleCanvasEvent` visibility mismatch in 3 diagram components,
removed redundant `_setupPush` calls, added 10 tests (radial-layout +
htn-diagram), and migrated `case-dependency-graph` from d3 to pages
graph infrastructure (502L deleted, 8 d3 dependencies removed).

**Graph migration audit result:** All four stencil packages (SWF, Case,
HTN, Org) now contain only domain config — schemas, icons, callbacks,
stencil definitions, edit policies. No generic infrastructure remains in
blocks-ui. Every graph component uses the pages rendering and layout
stack.

## Known Issues Needing Attention

### Palette drag green bar indicators not tested

The palette drag-to-canvas splice indicators (green bars on edges during
drag-over) were not verified. GraphCanvas has the infrastructure
(`getAddPlacement`, drop handler, splice indicator rendering) but the
SWF diagram's custom render might not wire the drag events correctly.
Lower-priority visual polish.

### Pre-existing: degraded property editing

"Property editing unavailable — No YAML path for task node" appears for
the Claim Review example. `yamlPaths` from `buildYamlPaths` doesn't
match `buildFlatGraph` node IDs. Pre-existing.

### Pre-existing typecheck errors (8 component groups)

`channel-activity`, `kpi-metric-row` (after _setupPush removal — needs
pages PushMixin to expose `willUpdate` reconnect path), `grouped-data-view`,
`blocks-dag-viewer`, `case-flow-viewer`, `blocks-decomposition-tree`,
`blocks-plan-item-tree`, `blocks-plan-model-dashboard`, `org-diagram`.
Most are `exactOptionalPropertyTypes` strictness issues or mixin type
mismatches with pages.

## What's Next

No active issues in the blocks-ui queue. Potential work:

- casehub-pages#473 — migrate `radial-layout.ts` from graph-stencil-org to
  graph-renderer (115L, standalone algorithm, no org-specific types)
- casehub-pages#467 — `_computeLayout` hook (eliminate `_fullRender` overrides)
- casehub-pages#468 — standard render template with default event wiring
- Address pre-existing typecheck errors (mostly pages-side mixin fixes)

## Cross-Module

Pages issues filed:
- casehub-pages#467 — `_computeLayout` layout hook
- casehub-pages#468 — standard render template
- casehub-pages#473 — migrate radial-layout to graph-renderer

## References

- `moveSwfTask` fix: `packages/graph-stencil-swf/src/adapter/swf-yaml-editor.ts`
- d3→pages migration: `components/case-dependency-graph/src/blocks-case-dependency-graph.ts`
- Contributor guide updated: `docs/guides/contributor-guide.md:295`
- Diary: `blog/2026-09-27-mdp01-the-last-d3-component.md`
