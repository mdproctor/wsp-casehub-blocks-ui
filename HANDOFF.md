# HANDOFF — casehub-blocks-ui

**Branch:** `issue-140-wire-pages-palettes`
**Date:** 2026-08-27

## Last Session

Completed Batch 2 (Tasks 4-6) of the #140 plan — EditPolicy implementations and YAML mutation functions. Three commits landed:

1. **CaseEditPolicy** (`graph-stencil-case/src/editing/`): grammar-aware `canConnect`, 4 creatable types (binding/worker/milestone/goal — not subcase), subcase non-deletable, disconnect strategy for all types (bipartite graph topology prevents auto-join reconnections — worker→worker and binding→binding are always grammar-invalid).

2. **SwfEditPolicy + addSwfTask** (`graph-stencil-swf/src/editing/` + `adapter/swf-yaml-editor.ts`): 5 creatable task types (call/set/switch/raise/try), boundary+root non-deletable, auto-join via `grammarAllows` helper (type-only check without cardinality — edges through deleted node are removed before reconnection, so cardinality constraints don't apply). `addSwfTask` does CST-preserving insertion of named task entries into `do:` block with type-specific defaults and unique step name generation.

3. **switchTriggerType + detectTriggerType** (`graph-stencil-case/src/adapter/yaml-editor.ts` + `worker-function/detect.ts`): CST-preserving trigger type switching in binding `on:` block, following the same pattern as `switchFunctionType`/`switchMcpTransport`. `detectTriggerType` and `TriggerType` migrated from diagram-core `trigger-editor.ts` to graph-stencil-case alongside other detection functions.

Key design finding: the `grammarAllows` helper in SwfEditPolicy checks type compatibility without cardinality, because `getDeleteStrategy` runs against the pre-deletion model where edges through the doomed node still count. CaseEditPolicy doesn't need this — its bipartite topology means auto-join always produces invalid type pairs regardless of cardinality.

## Immediate Next Step

Start Batch 3: Wire casehub-diagram (Task 7), Wire swf-diagram (Task 8), Remove old code (Task 9). Task 7 is the largest — wiring `_editPolicy()`, `_editorResolver()`, discriminator rendering closures, `_addElement()`, and removing old event handlers + palette + properties components. Run `work continue` on this branch.

## Queue

```
- [ ] #140 <- active
  - [x] Batch 1: Deps+Foundation (3/3 done)
  - [x] Batch 2: EditPolicy+Palette (3/3 done)
  - [ ] Batch 3: Wire+Cleanup
    - [ ] Task 7: Wire casehub-diagram
    - [ ] Task 8: Wire swf-diagram
    - [ ] Task 9: Remove old code
  - [ ] Batch 4: Showcase
    - [ ] Task 10: Update showcase pages
```

## Notes

- `casehub-diagram.ts` and `swf-diagram.ts` still won't compile until Tasks 7-8 update them — `_paletteTypes()` abstract was replaced by `_addElement(type)` in Task 3
- Pre-existing test failure in `case-adapter.test.ts` (external node test) — unrelated
- Spec at `specs/issue-140-wire-pages-palettes/2026-08-27-wire-pages-palettes-design.md`
- Plan at `plans/2026-08-27-wire-pages-palettes.md`
- Deferred issues: #141 (drag-to-canvas), #142 (edge reconnection UX), #143 (context menus)
- Both edit policies export from their respective package indexes and re-export `EditPolicy`/`StencilTypeInfo`/`DeleteStrategy` types
- `TRIGGER_TYPES` constant and `TriggerType` type added to `worker-function/types.ts`
