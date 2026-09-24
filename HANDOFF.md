# HANDOFF — casehub-blocks-ui

## Last Session

Rewrote SWF diagram layout from flat grid to recursive column tree.
Added for-loop support (stencil, grammar, container rendering, internal
connectors). Fixed container styling, edge visibility, handle visibility.
Filed four follow-up issues for next session.

### Recursive Column Layout (DONE)
- Replaced flat row/col/colSpan grid with four-phase recursive algorithm:
  build column tree → size bottom-up → position top-down → bottom-align
- Handles nested splits correctly (inner splits widen parent columns)
- 93 tests passing: linear, switch, unequal branches, nested splits,
  3-way bottom-fill, try/catch containers
- File: `packages/graph-stencil-swf/src/layout/swf-stack-layout.ts`

### For-Loop Support (DONE)
- Added `for` to `SWF_KNOWN_TYPES`
- Created `swf-for` stencil with grammar, render, icon
- Added to `CREATABLE_TYPES` in edit policy (shows in picker)
- Updated all grammar `allowedFrom`/`allowedTo` to include `swf-for`
- Container styling: orange border + grey background (matches try/catch)
- Internal connectors: edges shown inside `for` containers (sequential body)
- Internal edges hidden inside `try`/`try-catch` containers (structural, not sequential)
- Handle visibility: only inner structural nodes hide handles (top-level try-catch keeps handles for convergence edges)

### Example Workflows Added
- Nested Risk Assessment: switch inside a switch branch
- Batch Processing: for-loop with 3-step body
- Nested For Loops (ETL): outer loop over regions, inner loop over records, inside a 3-way switch split
- Claim Review updated: for-loop in left branch to test column resizing

### Key Architectural Decision
**blocks-ui must stay thin.** Stencil config, schemas, adapters, edit policies, icons — domain logic only. All generic infrastructure (layout strategies, render pipelines, event wiring, picker, drag-drop) belongs in pages. The SWF diagram currently overrides `render()` and `_fullRender()` from DiagramBaseMixin — this needs to be eliminated by adding proper hooks in pages.

### Issues Filed This Session
| # | Title | Summary |
|---|-------|---------|
| #170 | Stack-column layout: escape edge optimisation and pages migration | Column reordering, vertical slack, adaptive spacing; layout hook + base render in pages |
| #171 | Case diagram: deterministic layout for Plans and PlanItems | Compound PlanItem tree, optional stage chevrons; depends on #170 |
| #172 | SWF 1.0: complete task type stencils and picker filtering | Missing: Do, Emit, Fork, Listen, Run, Wait; upgrade SDK to 1.0.3 |
| #173 | SWF diagram: edge click picker and palette drag-to-canvas | Needs pages mixin changes (layout hook, standard render wiring) |

### Design Context Recovered (from engine specs)
- CMMN Stage was fully retired (blocks#60 Phase 3C.3), replaced by `PlanItemDefinition.Compound`
- Milestones no longer structurally contained in stages — expression-evaluated waypoints
- Cases mix 4 execution models (choreography, orchestration, workflow, agentic) simultaneously
- Layout should follow Compound PlanItem tree, not milestone columns
- Milestones can appear as chevron overlays for progress but don't drive column structure

## Immediate Next Steps

### Priority 1: SWF 1.0 Completeness (#172)
- Upgrade SDK to `@openworkflowspec/sdk@1.0.3-alpha8`
- Add stencils for Do, Emit, Fork, Listen, Run, Wait
- Update all grammars, verify picker filtering for complete set
- Update example YAML to `dsl: '1.0.3'`

### Priority 2: Pages Infrastructure (#170, #173)
- Pages: add `_computeLayout(model, options)` hook to DiagramBaseMixin
- Pages: move stack-column layout to graph-renderer as `computeStackColumnLayout`
- Pages: standard render template with edge click → picker, palette drag → drop zones
- Then: SWF diagram shrinks to adapter + edit policy + stencils (no custom render/fullRender)

### Priority 3: Case Layouts (#171)
- Deterministic layout for Compound PlanItem tree
- Nested containers for compound items (reuses stack-column engine)
- Optional stage chevrons when stages pattern is present
- Topological layers for flat DAGs

### Priority 4: IntelliJ Diagram Editor
- The end goal of all this layout/stencil work
- Needs deterministic layout, complete stencils, proper picker/drag interaction
- IntelliJ plugin already has JCEF panel + split editor + CI pipeline (#161 batches done)
- Next: wire the diagram component into the IntelliJ editor with full editing capabilities

## Cross-Module

Pages changes needed (separate repo):
- `diagram-base-mixin.ts` — layout hook, standard render template
- `graph-renderer/layout/` — stack-column layout strategy

## References

- Stack layout: `packages/graph-stencil-swf/src/layout/swf-stack-layout.ts`
- Stack layout tests: `packages/graph-stencil-swf/src/layout/swf-stack-layout.test.ts`
- For stencil: `packages/graph-stencil-swf/src/stencils/for.ts`
- Edit policy: `packages/graph-stencil-swf/src/editing/swf-edit-policy.ts`
- Example page: `examples/src/pages/swf-diagram-page.ts`
- Engine design specs (Stage retirement): `engine/docs/specs/2026-07-15-unified-execution-model-design.md`
- Engine design specs (Milestone alignment): `engine/docs/specs/2026-07-31-milestone-goal-alignment-design.md`
