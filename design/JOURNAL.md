# Design Journal — feature/graph-stencils

## §Phase0 — Schema Verification + Type Generation (2026-08-03)

**Decision:** Java model classes are generated from CaseDefinition.yaml via jsonschema2pojo — the schema is the single source of truth. No separate verification needed beyond confirming no stages remnants and checking hand-written Worker.java extensions. Worker's `additionalProperties: true` handles plugin-supplied fields (workflow, agent, inputSchema, outputSchema).

**Decision:** TypeScript types generated at developer-time via `json-schema-to-typescript`, checked into git. Generator script strips `_codegen*` engine-only properties and post-processes CaseCompletion index signature for `exactOptionalPropertyTypes` compatibility.

## §Phase2 — Case Stencil Read-Only Viewer (2026-08-03)

**Decision:** Stencil render functions are lit-html templates bridged to React Flow via `createReactNodeType()`. Keeps stencils framework-agnostic (reusable in palette/panels) while rendering inside React Flow's React-based canvas. The bridge is ~20 LOC in graph-stencil-case, not in graph-renderer.

**Decision:** `toReactFlowGraph()` uses local `RFNode`/`RFEdge` types instead of importing React Flow's `Node`/`Edge` directly — avoids React peer dependency for this pure transform function.

**Decision:** Edge derivation from string reference resolution — `binding.capability` matches `worker.capabilities[]` entries. Unresolvable capabilities produce `external:<name>` annotation nodes, not warnings (external workers are normal in CaseHub).

## §Phase3 — Property Editing (2026-08-03)

**Decision:** All YAML mutations go through `CaseAdapter.applyPropertyEdit()` — the diagram component never touches `yaml.Document` directly. Each call creates a fresh Document (no persistent state to sync with undo/redo).

**Decision:** Property edits skip `computeElkLayout()` — property changes don't alter graph topology, so existing positions are reused. This also eliminates the async race condition from rapid edits.

**Decision:** `toGraph()` returns `AdapterResult { model, yamlPaths }` where yamlPaths maps node IDs to YAML document paths. This bridges the gap between graph nodes and their YAML source locations without leaking adapter internals into node properties.

**Decision:** Field paths are arrays `['outcomePolicy', 'onDecline']` throughout — never dot-separated strings. Events use `composed: true, bubbles: true` to cross Shadow DOM.

**Decision (light design review):** Trigger type switching emits a single `property-change` for the full `on` object. Binding target type switching is deferred to Phase 4 (structural editing). CloudEvent form determined by current value type (string vs object), not a toggle.
