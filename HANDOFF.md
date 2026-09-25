# HANDOFF — casehub-blocks-ui

## Last Session

Completed OWS 1.0 stencil coverage (#172) — all 12 task types now have
distinct stencils in graph-stencil-swf. Also closed #161 (IntelliJ LSP
domain schema assembly — all planned tasks were already done).

### OWS 1.0 Stencil Completeness (#172) — DONE

- Upgraded `@openworkflowspec/sdk` from 0.0.1 to 1.0.3-alpha8
- SDK 1.0.3 changed node ID format from index-based (`/do/0/name`) to
  name-based (`/do/name`). Updated yaml-path-walker and all adapter tests.
- Created shared grammar constants `FLOW_SOURCES`/`FLOW_TARGETS` in
  `packages/graph-stencil-swf/src/stencils/grammars.ts`. Refactored all
  11 existing grammars to use them instead of inline arrays.
- Added 4 leaf stencils: Emit (purple, event type subtitle), Listen
  (teal, consumption strategy), Run (green, sub-type indicator), Wait
  (slate, duration). All follow the Call/Set card pattern.
- Added 2 container stencils: Do (sequential sub-tasks, compact header
  like For), Fork (parallel branches, "race" badge when compete=true).
- Wired Do/Fork into swf-diagram: containerTypes set, edge visibility
  (fork hides internal edges), handle visibility for nested containers.
- Added layout tests for Do and Fork containers.
- Updated edit policy: CREATABLE_TYPES now has all 12 types.
- Updated all example YAML from `dsl: 1.0.0-alpha1` to `dsl: '1.0.3'`.
- Added event-driven order processing example exercising all 6 new types.
- Adapter coverage test verifies no types fall through to swf-generic.
- 113 tests passing across 6 test files.

### Key Finding: SDK ID Format Change

The 1.0.3-alpha8 SDK produces name-based flat graph node IDs instead
of index-based IDs. Example: `/do/fetchData` instead of `/do/0/fetchData`.
The yaml-path-walker was updated to match. Any code that hardcodes
index-based IDs will need updating.

## Immediate Next Steps

### Priority 1: SWF Edge Click Picker + Palette Drag (#173)
- Needs pages-repo changes: `_computeLayout()` hook in DiagramBaseMixin,
  standard render template with edge click → picker, palette drag → drop zones
- Then blocks-ui: SWF diagram shrinks to adapter + edit policy + stencils
  (no custom render/fullRender overrides)
- Cross-repo: file pages issue for layout hook + render template

### Priority 2: Stack-Column Layout Optimisation (#170)
- Column reordering, vertical slack, adaptive spacing
- Migrate layout to pages graph-renderer as `computeStackColumnLayout`
- Escape edge routing improvements

### Priority 3: Case Diagram Deterministic Layout (#171)
- Compound PlanItem tree layout
- Nested containers for compound items (reuses stack-column engine)
- Optional stage chevrons
- Depends on #170 (layout infrastructure in pages)

### Priority 4: IntelliJ Diagram Editor
- All layout/stencil prerequisites now complete
- IntelliJ plugin has JCEF panel + split editor + CI pipeline (from #161)
- Next: wire diagram component into IntelliJ editor with full editing

## Cross-Module

Pages changes needed (separate repo):
- `diagram-base-mixin.ts` — layout hook, standard render template
- `graph-renderer/layout/` — stack-column layout strategy

## References

- OWS 1.0 DSL reference: https://github.com/open-workflow-specification/specification/blob/main/dsl-reference.md
- Stencils: `packages/graph-stencil-swf/src/stencils/`
- Grammar constants: `packages/graph-stencil-swf/src/stencils/grammars.ts`
- Layout: `packages/graph-stencil-swf/src/layout/swf-stack-layout.ts`
- Container wiring: `components/swf-diagram/src/swf-diagram.ts:222-247`
- Design spec: `specs/issue-158-lsp-schema-refinements/2026-09-25-ows-1.0-stencil-completeness-design.md`
- Plan: `plans/2026-09-25-ows-stencil-completeness.md`
