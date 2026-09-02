## D1: Architectural approach — compose existing rendering pipeline, not build parallel

**Choice:** The viewer composes the existing rendering pipeline pieces (toGraph, toDecorations, computeElkLayout, pages-graph-canvas, case stencils) — the same pieces casehub-diagram uses. No code extraction, no duplication. This is the read-only counterpart to the editor.
**Alternatives:**
- Build a new adapter from a CaseFlowResponse type — introduces a parallel data contract and duplicates the rendering wiring
- Wrap casehub-diagram with editor features hidden — carries unnecessary weight (undo/redo, palette, property panels, YAML editing)
**Rationale:** Diagram editors are designed so their rendering sub-parts can be composed and used read-only. The viewer is not an "extraction" — it's an independent composition of the same shared units.
**Trade-offs:** The issue's proposed CaseFlowResponse API is replaced by the existing CaseDefinition YAML + CaseRuntimeState contract. Consumers must provide data in the existing shape.
**Sources:** casehub-diagram.ts:100-218, graph-stencil-case/src/adapter/case-adapter.ts, graph-stencil-case/src/runtime/runtime-adapter.ts
**Exploration:** quick
**Status:** captured

## D2: Data contract — YAML string + src endpoint

**Choice:** Same data contract as casehub-diagram: `src` attribute (fetch YAML from URL) and `yaml` property (pass string directly), plus `runtimeState` property for runtime decorations.
**Alternatives:**
- Props only (like blocks-dag-viewer) — consumer handles fetch lifecycle, but blocks-dag-viewer has no editor counterpart so it's not the right reference
- DataSourceMixin — heavier than needed for a viewer that takes a single YAML source + runtime state
**Rationale:** The viewer is casehub-diagram's read-only counterpart. Same data contract means consumers can switch between editor and viewer by changing a tag name.
**Trade-offs:** None significant — this is the established pattern.
**Sources:** diagram-core DiagramBaseMixin (src fetch pipeline)
**Exploration:** quick
**Status:** captured

## D3: Rendering pipeline — DiagramBaseMixin in readonly mode

**Choice:** Extend DiagramBaseMixin with `readonly=true`. Override `_adaptYaml()` to call `toGraph()` and `_decorations()` to call `toDecorations()` — same as casehub-diagram. Gets src fetch, ELK layout, canvas rendering, error/degraded modes, SVG/PNG export for free.
**Alternatives:**
- Direct composition (import toGraph, computeElkLayout, toReactFlowGraph, pages-graph-canvas individually) — more explicit but duplicates the rendering pipeline wiring that DiagramBaseMixin already provides
**Rationale:** DiagramBaseMixin is designed for this — the readonly flag disables editing while preserving the full rendering pipeline. Smallest implementation surface.
**Trade-offs:** Inherits some editor concepts (dirty tracking, undo/redo) that are inert in readonly mode but present in the class.
**Sources:** packages/diagram-core (DiagramBaseMixin), casehub-diagram.ts:112-218
**Exploration:** quick
**Status:** captured

## D4: Runtime data — extend CaseRuntimeState with optional flow fields

**Choice:** Add optional `trustScores`, `adaptiveDecisions`, and `parallelGroups` fields to CaseRuntimeState. `toDecorations()` handles all runtime overlays. Single data source keeps the viewer API simple.
**Alternatives:**
- Separate properties on the viewer (trustScores, adaptiveDecisions, parallelGroups alongside runtimeState) — fragments runtime data across multiple inputs
- New CaseFlowResponse wrapper type — creates two tiers of runtime data, adds API surface
**Rationale:** CaseRuntimeState is the single source of runtime overlay data for case diagrams. Extending it keeps the viewer's property surface minimal and the decoration pipeline unified.
**Trade-offs:** CaseRuntimeState grows — consumers that don't need trust/adaptive/parallel data pass the same type with those fields absent (optional fields).
**Sources:** graph-stencil-case/src/runtime/types.ts (CaseRuntimeState), graph-stencil-case/src/runtime/runtime-adapter.ts (toDecorations)
**Exploration:** quick
**Status:** captured

## D5: Parallel group rendering — ELK sub-graphs

**Choice:** Use ELK's compound node / partition feature to lay out parallel groups as side-by-side sub-graphs. Structurally correct layout — parallel branches are visually distinct columns.
**Alternatives:**
- Visual grouping only (dashed boxes as post-layout overlay) — simpler but layout may interleave parallel nodes, defeating the purpose
- Defer parallel groups — simplifies v1 but was explicitly scoped in
**Rationale:** Parallel execution is a first-class concept in case flows. The layout must reflect it structurally, not just decoratively. ELK supports this natively via compound nodes.
**Trade-offs:** More complex ELK configuration. May need investigation into ELK partition API if not already used in the codebase.
**Sources:** blocks-dag-viewer.ts:56 (computeElkLayout usage), issue #150 spec (parallelGroups field)
**Exploration:** quick
**Depends on:** D4 (parallelGroups comes from CaseRuntimeState)
**Status:** captured
