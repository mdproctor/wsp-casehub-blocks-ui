# HANDOFF — casehub-blocks-ui

**Branch:** feature/graph-stencils
**Date:** 2026-08-03
**Issue:** #103 (Epic: Visual Diagram Editor — Domain Layer)

## What landed

Phases 0, 2, and 3 of the visual diagram editor. 18 commits, 64 tests.

- **Phase 0:** TypeScript type generation from CaseDefinition.yaml JSON Schema. Generator script with `_codegen*` stripping and `exactOptionalPropertyTypes` post-processing. Schema audit confirmed no staleness.
- **Phase 2:** Read-only viewer — `toGraph()` adapter (YAML → GraphModel with capability-dispatch edge derivation), 5 stencil render functions (lit-html templates bridged to React Flow via `createReactNodeType()`), `<casehub-diagram>` component with ELK auto-layout.
- **Phase 3:** Property editing — `applyPropertyEdit()` CST-preserving YAML editing, `AdapterResult` with `yamlPaths`, validation + field-renderer, trigger oneOf editor, nested group, `<casehub-diagram-properties>` Shadow DOM panel, split layout, selection, undo/redo (YAML snapshots), no re-layout on property edits.

Phase 3 spec underwent light design review (coherence + structure + robustness + cross-cutting). Key revisions: adapter owns all YAML mutations, skip re-layout eliminates async race, array field paths not dot-separated, fresh Document per edit call.

## Immediate next step

Phase 4 is next. Run `/work` to continue on `feature/graph-stencils`. Read the parent spec (§4 Phase 4A/4B) and the Phase 3 spec's §10 for review findings context. Phase 4A (structural editing) and 4B (persistence backends) can run in parallel.

## What's left

- Phase 4A: structural editing — add/remove/replace nodes via palette, `applyEdit()` in domain adapter, YAML round-trip · L · High
- Phase 4B: persistence backends — Git, in-memory, Electron file · S · Low
- Phase 5: SWF drill-down — depends on `@openworkflowspec/sdk` · M · Med
- Phase 6: work registry — marketplace YAML loader · M · Med
- Phase 7: runtime overlay — PushSource-based live state · M · High
- Pre-existing: 3 modified channel-activity files (unstaged, from another session)
- Pre-existing: document-workbench build errors (document-diff.ts)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 4A: structural editing | L | High | Palette, add/remove/replace, applyEdit() |
| — | Phase 4B: persistence backends | S | Low | Parallel with 4A |
| — | Phase 5: SWF drill-down | M | Med | Blocked on @openworkflowspec/sdk |

## References

- Parent design spec: `~/claude/public/casehub/specs/2026-08-01-visual-diagram-editor-design.md`
- Phase 3 spec: `specs/feature-graph-stencils/2026-08-03-phase3-property-editing-design.md`
- Phase 3 plan: `plans/2026-08-03-phase3-property-editing.md`
- Design journal: `design/JOURNAL.md`
- Blog: `blog/2026-08-03-mdp01-from-yaml-to-graph-in-one-session.md`
