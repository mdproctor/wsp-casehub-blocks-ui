# HANDOFF — casehub-blocks-ui

## Last Session

Extended the IIFE diagram bundle from a read-only viewer into a fully
interactive editor. 22 commits on blocks-ui, 2 on pages. 17 Playwright
tests covering all features.

### IIFE Bundle Foundation
- `ignoreAnnotations: true` preserves all web component registrations
- External stencil rendering (stale dist rebuild)
- SWF stencils + thumbnail renderer registered
- Selection outline matches content (`height: auto` on wrapper)
- `nodesselection-rect` hidden in JCEF shell

### Diagram Workbench + Drill-down
- Case format uses `blocks-diagram-workbench` for drill-down navigation
- Drilled-down SWF diagrams are editable (removed `readonly`)
- Drill-down button clickable (buttons at z-index 3 above source-full handle)

### Node Picker (DiagramBaseMixin — shared)
- Canvas click and connect-end-on-empty show `PagesNodeChooser`
- Grammar-derived filtering (outbound allowedTo + inbound allowedFrom)
- Dynamic filtering (hides types when source at max connections, excludes auto-generated edges)
- 800ms mouse-leave auto-dismiss
- `composedPath()` fix for shadow DOM click-outside

### Connect-End Auto-Wiring (casehub-diagram)
- binding→worker: shared capability
- worker→binding: shared capability (reverse)
- binding→milestone: condition expression
- binding→goal / milestone→goal: expression.all
- New bindings get capability set via `parseDocument`

### Pages Fixes (on pages `main` + `issue-433-structural-editing`)
- `getNodeTypes()` memoized (stable React Flow handles)
- Default handle positions for new nodes
- `isEligible` allows root-children for hold-to-drag
- Wrapper z-index removed so buttons punch through source-full
- `PagesNodeChooser` mouse-leave timeout

## Open Issues

Three issues filed for next session:

| # | Title | Priority |
|---|-------|----------|
| #163 | SWF palette adds connected instead of standalone | Start here |
| #164 | Canvas pans during hold-to-drag | After #165 |
| #165 | Showcase gallery SWF diagram not rendering | Blocks #163/#164 |

## Immediate Next Step

Start with **#165** — get the showcase gallery SWF diagram rendering.
This is needed to investigate and verify #163 (standalone task add) and
#164 (hold-to-drag panning), both of which the user reports were working
in the gallery previously.

## Cross-Module

- pages `main`: graph-renderer fixes (handles, nodeTypes, z-index, coordinator eligibility)
- pages `issue-433-structural-editing`: DiagramBaseMixin picker + chooser composedPath fix
- pages-diagram-palette: mouse-leave timeout on PagesNodeChooser

## References

- Spec: `specs/issue-158-lsp-schema-refinements/2026-09-17-domain-schema-assembly-design.md`
- Decisions: `specs/issue-158-lsp-schema-refinements/decisions.md`
- Playwright tests: `examples/tests/diagram-iife-bundle.spec.ts` (17 tests)
