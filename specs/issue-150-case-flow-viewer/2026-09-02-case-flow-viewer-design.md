# blocks-case-flow-viewer — Design Spec

**Issue:** casehubio/blocks-ui#150
**Date:** 2026-09-02
**Branch:** issue-150-case-flow-viewer

## Summary

Read-only case flow DAG viewer — the viewer counterpart to `casehub-diagram`. Composes the same rendering pipeline pieces (`toGraph`, `toDecorations`, `computeElkLayout`, `pages-graph-canvas`, case stencils) that `casehub-diagram` uses, without the editor machinery. Any CaseHub application that needs to display a case definition with runtime state can use this component instead of wiring `casehub-diagram` in readonly mode with all its editor weight (palette, property panels, YAML editing, undo/redo).

## Architecture

### Rendering pipeline (composed, not duplicated)

The viewer uses `DiagramBaseMixin` from `@casehubio/pages-diagram-core` with `readonly=true`. This provides:

- `src` / `yaml` properties — fetch YAML from URL or pass directly
- `_fullRender()` — async ELK layout pipeline
- `_nodes` / `_edges` — positioned graph data for `pages-graph-canvas`
- `_error` / `_renderError()` — error and degraded mode handling
- `_exportDiagram()` — SVG/PNG export
- `_mode` — design/runtime toggle (viewer defaults to `runtime`)

The viewer overrides three abstract methods:

| Method | Implementation |
|--------|---------------|
| `_adaptYaml(yaml)` | Calls `toGraph(yaml)` from `graph-stencil-case` |
| `_applyPropertyEdit()` | No-op (throws — read-only) |
| `_emptyTemplate()` | Returns `null` (no empty template — viewer requires data) |

### Decoration pipeline

The viewer overrides `_decorations()` to call `toDecorations(runtimeState)` from `graph-stencil-case`'s runtime module when `runtimeState` is set.

Trust scores, adaptive decisions, and parallel groups flow through extended `CaseRuntimeState` fields (see Data Contract below). The `toDecorations()` function is extended to handle these new fields, keeping the decoration logic in the stencil package where it belongs.

## Component API

```typescript
@customElement('blocks-case-flow-viewer')
export class BlocksCaseFlowViewer extends DiagramBaseMixin(LitElement) {
  // Inherited from DiagramBaseMixin:
  // @property() yaml: string        — pass CaseDefinition YAML directly
  // @property() src: string         — fetch YAML from this URL
  // @property({ type: Boolean }) readonly: boolean  — always true

  /** Runtime state for decoration overlay */
  @property({ attribute: false }) runtimeState: CaseRuntimeState | null = null;

  /** Selection topic for node click events */
  @property({ attribute: 'selection-topic' }) selectionTopic: string = '';
}
```

Usage:

```html
<!-- Fetch from endpoint -->
<blocks-case-flow-viewer
  src="/api/cases/${caseId}/definition"
  .runtimeState=${this._runtimeState}
  selection-topic="case-flow">
</blocks-case-flow-viewer>

<!-- Inline YAML -->
<blocks-case-flow-viewer
  .yaml=${yamlString}
  .runtimeState=${this._runtimeState}>
</blocks-case-flow-viewer>
```

## Data Contract

### Input: CaseDefinition YAML

Same YAML format that `casehub-diagram` consumes — the `toGraph()` adapter parses it into a `GraphModel` with worker, binding, milestone, goal, and subcase nodes.

### Input: CaseRuntimeState (extended)

Extend the existing `CaseRuntimeState` type in `graph-stencil-case/src/runtime/types.ts` with optional fields:

```typescript
export interface TrustScoreSnapshot {
  readonly bindingName: string;
  readonly workerId: string;
  readonly score: number;      // 0-100
}

export interface AdaptiveDecisionSnapshot {
  readonly trigger: string;
  readonly condition: string;
  readonly fired: boolean;
  readonly timestamp: string;
  readonly affectedBindings?: readonly string[];
}

export interface CaseRuntimeState {
  readonly planItems: readonly PlanItemSnapshot[];
  readonly milestones: readonly MilestoneSnapshot[];
  readonly timestamp: string;
  readonly caseStatus?: string;

  // New optional fields for flow viewer
  readonly trustScores?: readonly TrustScoreSnapshot[];
  readonly adaptiveDecisions?: readonly AdaptiveDecisionSnapshot[];
  readonly parallelGroups?: readonly (readonly string[])[];  // arrays of bindingNames
}
```

These fields are optional — existing consumers of `CaseRuntimeState` are unaffected.

### Output: Selection events

On node click, emits a `pages-event` on the configured `selectionTopic` with payload:

```typescript
{ nodeId: string, nodeType: string, properties: Record<string, unknown> }
```

The `nodeId` follows graph-stencil-case conventions: `binding:<name>`, `worker:<name>`, `milestone:<name>`, etc.

## Rendering Features

### Runtime decorations (base)

Status badges on each node via `toDecorations()` using the existing `lookupStatus` → `BADGE_COLORS` pipeline:

- Green badge — COMPLETED
- Blue pulse — RUNNING
- Amber — DELEGATED / SUSPENDED
- Red — FAULTED
- Grey — PENDING / CANCELLED / OBSOLETE / REJECTED

Binding nodes aggregate their plan items using the existing `aggregateBinding()` logic (active-worst-first priority, terminal severity sorting).

### Trust score pills

When `runtimeState.trustScores` is provided, `toDecorations()` adds a trust score label to each worker node's decoration. The label renders as a pill below the node badge showing the numeric score (0-100) with colour thresholds:

| Score range | Colour | Meaning |
|-------------|--------|---------|
| 80-100 | Green (#22c55e) | High trust |
| 50-79 | Amber (#eab308) | Moderate trust |
| 0-49 | Red (#ef4444) | Low trust |

`NodeDecoration` is extended with an optional `pills` array in `graph-core` (upstream). Each pill has `text`, `color`, and optional `icon`. This is domain-agnostic — trust scores, execution times, SLA deadlines, cost indicators all use the same mechanism. The stencil renderer in `graph-renderer` renders pills below the node badge.

`toDecorations()` maps `runtimeState.trustScores` into pills on the corresponding worker node decorations. The colour thresholds (green/amber/red) are applied during decoration construction, not rendering — the pill carries its resolved colour.

### Adaptive decision highlighting

When `runtimeState.adaptiveDecisions` is provided, `toDecorations()` marks affected binding/worker nodes:

- **Fired decisions** — the affected nodes get a lightning bolt (⚡) secondary badge and a tooltip showing the trigger and condition
- **Unfired decisions** — no visual change (they didn't affect the flow)

The `affectedBindings` field on each decision maps it to graph node IDs via the `binding:<name>` convention.

### Parallel group rendering

When `runtimeState.parallelGroups` is provided, the viewer passes ELK compound node constraints to `computeElkLayout()` via the `_layoutOptions()` override.

Parallel groups use ELK's native partitioning — no synthetic compound nodes, no model mutation. The viewer overrides `_layoutOptions()` to assign partition indices to nodes based on `runtimeState.parallelGroups`.

Implementation approach:
1. Override `_layoutOptions()` to return ELK partition constraints derived from `runtimeState.parallelGroups`
2. Each parallel group's member nodes (identified by binding name) receive the same partition index
3. ELK lays out partitioned nodes side-by-side without requiring changes to the `GraphModel`
4. If visual grouping indicators are needed (dashed borders around groups), a post-layout overlay draws them based on the partition boundaries — this is a rendering concern, not a model concern

### Toolbar

A toolbar component (`blocks-case-flow-toolbar`) showing:

- Node count and completion stats (completed / running / failed counts)
- Staleness timer (seconds since last `runtimeState.timestamp`)
- Case status badge (from `runtimeState.caseStatus`)
- Export buttons (SVG / PNG via `_exportDiagram()`)

Follows the same pattern as `blocks-dag-toolbar`.

## ARIA

| Element | Role | Attributes |
|---------|------|------------|
| Host | `region` | `aria-label="Case flow viewer"` |
| Toolbar | `status` | `aria-label="Case flow status"`, `aria-live="polite"` |
| Graph canvas | `img` | `aria-label="Case flow diagram"` |

## Component Structure

```
components/case-flow-viewer/
  package.json
  tsconfig.json
  src/
    blocks-case-flow-viewer.ts     — main component (extends DiagramBaseMixin)
    blocks-case-flow-toolbar.ts    — toolbar with stats + export
    blocks-case-flow-viewer.test.ts
    blocks-case-flow-toolbar.test.ts
```

Changes in existing packages:

```
packages/graph-stencil-case/src/runtime/
  types.ts        — add TrustScoreSnapshot, AdaptiveDecisionSnapshot, extend CaseRuntimeState
  runtime-adapter.ts  — extend toDecorations() for trust scores + adaptive decisions
  decoration.ts   — add trust score pill colour logic
```

Trust score rendering uses the `NodeDecoration.pills` array — an upstream extension to `graph-core`. See the trust score pills section for details.

## Relation to existing components

| Component | Stencil package | Purpose |
|-----------|----------------|---------|
| `casehub-diagram` | `graph-stencil-case` | Full case definition editor (palette, YAML, properties) |
| **`blocks-case-flow-viewer`** | **`graph-stencil-case`** | **Read-only case flow viewer with runtime state** |
| `blocks-dag-viewer` | `graph-stencil-htn` | HTN plan execution DAG viewer |
| `swf-diagram` | `graph-stencil-swf` | SWF workflow diagram (extends DiagramBaseMixin) |

The viewer and the editor share the same rendering pipeline and stencil registrations. The viewer is not extracted from the editor — it is an independent composition of the same shared units.

## Testing Strategy

### Unit tests

- `toGraph()` produces correct `GraphModel` from sample CaseDefinition YAML (existing adapter tests cover this)
- Extended `toDecorations()` produces correct decorations for trust scores, adaptive decisions
- Parallel group compound node insertion works correctly
- Toolbar renders stats and staleness accurately

### Component tests

- Viewer renders graph canvas when YAML is set
- Viewer renders graph canvas when `src` is set (mock fetch)
- Runtime decorations appear when `runtimeState` is set
- Trust score pills appear with correct colours at threshold boundaries
- Adaptive decision badges appear on affected nodes for fired decisions
- Parallel groups render as side-by-side compound nodes
- Node click emits `pages-event` on `selectionTopic`
- Empty state shows "No case definition loaded"
- Error state shows error message with retry
- ARIA attributes present on host, toolbar, and canvas

## References

- `components/casehub-diagram/src/casehub-diagram.ts:100-218` — editor using same pipeline
- `components/blocks-dag-viewer/src/blocks-dag-viewer.ts` — analogous HTN viewer pattern
- `packages/graph-stencil-case/src/adapter/case-adapter.ts` — `toGraph()` adapter
- `packages/graph-stencil-case/src/runtime/runtime-adapter.ts` — `toDecorations()` function
- `packages/graph-stencil-case/src/runtime/types.ts` — `CaseRuntimeState` type
- `packages/graph-stencil-case/src/stencils/register.ts` — stencil registrations
- `.casehub-packages/packages/pages-diagram-core/src/diagram-base-mixin.ts` — DiagramBaseMixin
- PP-20260806-320d50 — stencil package isolation protocol
- PP-20260713-8ea1af — component customisation pattern protocol
- casehubio/blocks-ui#150 — issue spec
