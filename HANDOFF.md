# HANDOFF — casehub-blocks-ui

**Branch:** `issue-140-wire-pages-palettes`
**Date:** 2026-08-27

## Last Session

Closed #136 (rich property schemas) — 5 squashed commits landed on main: schema registry, 5 case schemas with discriminated unions, HTN schemas, SWF x-group annotations, custom editor stubs. Then filed #140 and started it: brainstormed, designed (standard spec review with significant enrichment around virtual discriminators and the data-schema mismatch problem), planned (10 tasks in 4 batches), completed Batch 1 (dependencies, editor event contract fix, PropertyPaletteSource adapter in DiagramBaseMixin).

Key design decisions: EditorResolver via protected method on mixin (not abstract), EditPolicy validates before domain adapter mutates YAML, discriminator rendering via closures capturing `this` to access switch functions directly. The spec's §Virtual Discriminators section is critical reading — worker functionType, binding trigger, agent model provider are all virtual selectors that don't match the data structure.

## Immediate Next Step

Start Batch 2: CaseEditPolicy (Task 4), SwfEditPolicy + addSwfTask (Task 5), switchTriggerType + migrate detectTriggerType (Task 6). Run `work continue` on this branch.

## Queue

```
- [ ] #140 <- active
  - [x] Batch 1: Deps+Foundation (3/3 done)
  - [ ] Batch 2: EditPolicy+Palette
    - [ ] Task 4: CaseEditPolicy
    - [ ] Task 5: SwfEditPolicy + addSwfTask
    - [ ] Task 6: switchTriggerType + migrate detectTriggerType
  - [ ] Batch 3: Wire+Cleanup
    - [ ] Task 7: Wire casehub-diagram
    - [ ] Task 8: Wire swf-diagram
    - [ ] Task 9: Remove old code
  - [ ] Batch 4: Showcase
    - [ ] Task 10: Update showcase pages
```

## Notes

- `casehub-diagram.ts` and `swf-diagram.ts` will not compile until Tasks 7-8 update them — `_paletteTypes()` abstract was replaced by `_addElement(type)` in Task 3
- Pre-existing test failure in `case-adapter.test.ts` (external node test) — unrelated
- Spec at `specs/issue-140-wire-pages-palettes/2026-08-27-wire-pages-palettes-design.md`
- Plan at `plans/2026-08-27-wire-pages-palettes.md`
- Deferred issues filed during spec review: #141 (drag-to-canvas), #142 (edge reconnection UX), #143 (context menus)
