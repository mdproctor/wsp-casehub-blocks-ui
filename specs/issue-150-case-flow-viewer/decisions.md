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

## D5: Parallel group rendering — ELK partition constraints

**Choice:** Use ELK's native partitioning via `_layoutOptions()` — assign nodes in the same parallel group a partition index. No synthetic compound nodes, no model mutation. The layout concern stays in the layout layer.
**Alternatives:**
- Compound nodes inserted post-adapter (original proposal) — mutates GraphModel after toGraph(), fragile coupling to internal graph structure
- Extend toGraph() to accept parallel groups — right if parallel groups are a permanent case concept, but over-engineers the adapter for what is fundamentally a layout concern
- Visual grouping only (dashed boxes as post-layout overlay) — simpler but layout may interleave parallel nodes
**Rationale:** Parallel grouping is a layout concern, not a structural one. ELK supports partitioning natively without requiring changes to the graph model. No synthetic nodes, no model mutation, cleanest separation of concerns.
**Trade-offs:** ELK partitioning may have less visual control than compound nodes (no containing border around the group). If visual grouping is needed, a post-layout overlay can supplement the partition layout.
**Sources:** blocks-dag-viewer.ts:56 (computeElkLayout usage), ELK partitioning documentation
**Exploration:** deep-analysis
**Depends on:** D4 (parallelGroups comes from CaseRuntimeState)
**Status:** captured

## D6: Trust score rendering — extend NodeDecoration with pills array

**Choice:** Add an optional `pills` array to `NodeDecoration` in `graph-core` (upstream). Each pill has `text`, `color`, and optional `icon`. `toDecorations()` maps `trustScores` to pills. Any stencil renderer renders pills below the badge.
**Alternatives:**
- Inject trust scores into GraphNode.properties during _adaptYaml() — conflates definition data with runtime visual state, breaks the clean boundary between GraphNode (definition) and NodeDecoration (runtime overlay)
- Add a `label` field to NodeDecoration — too specific, only handles one supplementary element
**Rationale:** NodeDecoration is the runtime visual overlay channel. Trust scores are runtime data, not definition data. A pills array is domain-agnostic and reusable — execution times, SLA deadlines, cost indicators all fit the same pattern. Non-breaking upstream change (additive optional field).
**Trade-offs:** Requires an upstream change to graph-core in the pages repo. If pages can't be updated in this branch, trust scores can be deferred or delivered via the workaround with a follow-up to migrate.
**Sources:** graph-core/src/model.ts (NodeDecoration type), graph-stencil-case/src/runtime/decoration.ts (toDecoration)
**Exploration:** deep-analysis
**Status:** captured
