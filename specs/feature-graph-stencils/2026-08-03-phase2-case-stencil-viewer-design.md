# Phase 2 — Case Stencil Read-Only Viewer

**Date:** 2026-08-03
**Issue:** #103 (Epic: Visual Diagram Editor — Domain Layer)
**Status:** Approved
**Parent spec:** `specs/2026-08-01-visual-diagram-editor-design.md` (parent workspace)
**Depends on:** Phase 0 (TypeScript types — completed), pages#258 Phase 1A (graph-core), Phase 1B (graph-renderer)

---

## 1. Goal

Render a real CaseDefinition YAML as a directed graph with auto-layout. First visual output of the diagram editor. Viewer mode only — no editing, no palette, no property panel.

**Milestone:** Render `document-processing.yaml` as a visual graph with Workers, Bindings, Milestones, Goals connected by derived edges.

## 2. Data Flow

```
YAML string
  → yaml.parse() → CaseDefinition (generated types from Phase 0)
  → CaseAdapter.toGraph() → GraphModel (graph-core)
  → toReactFlowGraph() → React Flow Node[] + Edge[]
  → computeElkLayout() → positioned Node[]
  → <pages-graph-canvas> renders with registered stencil node types
```

### 2.1 CaseAdapter.toGraph()

Pure function. Input: YAML string. Output: `GraphModel` (graph-core).

**Node creation:**

| Source | Node type | Properties carried |
|--------|-----------|-------------------|
| `spec.workers[]` | `worker` | name, description, capabilities |
| `spec.bindings[]` | `binding` | name, capability, on (trigger), when, subCase, humanTask |
| `spec.milestones[]` | `milestone` | name, condition, entryCriteria, slaDuration, slaStartFrom |
| `spec.goals[]` | `goal` | name, kind, condition |
| `binding.subCase` | `subcase` | namespace, name, version, groupId, totalInGroup, requiredCount |

Node IDs: `<type>:<name>` (e.g., `worker:ocr-worker`, `binding:extract-text`). Bindings without names use index: `binding:_0`.

**Edge derivation:**

| Edge type | Source → Target | Derivation |
|-----------|----------------|------------|
| `capability-dispatch` | Binding → Worker | `binding.capability` string matches `worker.capabilities[]` entry |
| `subcase-spawn` | Binding → SubCase | `binding.subCase` object present |
| `humantask-create` | Binding → (inline annotation) | `binding.humanTask` present — no separate node, shown as binding target badge |

Edge IDs: `<source-id>--<type>--<target-id>`.

Unresolvable capability references (binding references a capability no in-definition worker owns) produce an informational annotation edge to a synthetic `external:<capability-name>` node — not a warning. External workers are normal in CaseHub.

### 2.2 toReactFlowGraph()

Transforms `GraphModel` → React Flow `Node[]` + `Edge[]`.

- `GraphNode` → React Flow `Node`: `{ id, type, position: {x:0, y:0}, data: node.properties }`
- `GraphEdge` → React Flow `Edge`: `{ id, source, target, type: edge.type, animated: false }`
- Position is placeholder — ELK layout overwrites it.

### 2.3 Layout

`computeElkLayout(nodes, edges, { direction: 'DOWN', spacing: 60 })` from graph-renderer. Workers are not containers in Phase 2 (containment layout is Phase 4). Flat layout with capability-dispatch edges connecting bindings to workers.

## 3. Stencil Architecture

### 3.1 Three-part stencil definition

Each stencil has:
1. **Grammar** — `StencilGrammar` (graph-core): connection rules, registered via `registerGrammar()`
2. **Render function** — `(data: Record<string, unknown>) => TemplateResult` using lit-html
3. **Node type registration** — wrapped into a React component via `createReactNodeType()`, registered with graph-renderer's `registerNodeType()`

### 3.2 Lit→React bridge

`createReactNodeType(renderFn)` — utility in graph-stencil-case. Returns a React function component compatible with `NodeTypeDescriptor.component`. Implementation:

1. React component receives `data` prop (from React Flow node)
2. Creates a `<div>` ref
3. On mount and data change, calls `litRender(renderFn(data), div)` using lit-html's `render()`
4. Returns the div

~20 LOC. Lives in `src/bridge/create-react-node-type.tsx`. Requires `react` as a peerDependency (graph-renderer already brings React into the bundle).

### 3.3 Stencil visuals

| Stencil | Shape | Key visuals |
|---------|-------|-------------|
| **Binding** | Rounded rectangle (200×80) | Trigger type icon (contextChange/cloudEvent/schedule/scopeActivated), target type badge (capability/subCase/humanTask), when-condition preview truncated to 40 chars |
| **Worker** | Rectangle (220×60) | Capability list (comma-separated), description truncated |
| **Milestone** | Diamond (160×80) | Condition preview, SLA duration badge if present |
| **Goal** | Hexagon (160×70) | Kind badge (success=green, failure=red, custom=blue), condition preview |
| **SubCase** | Double-border rectangle (200×70) | namespace/name/version, M-of-N group badge if groupId present |

All stencils use:
- Inline styles only (no `<style>` blocks — CSS contract from design spec)
- `--pages-*` CSS custom properties for colors/typography (available via `applyTheme()` on the canvas container)
- Node dimensions are initial estimates — ELK adjusts spacing

### 3.4 Registration function

`registerCaseStencils()` — called once by casehub-diagram on init. Registers all 5 grammars with graph-core and all 5 node types with graph-renderer.

## 4. casehub-diagram Component

### 4.1 Phase 2 scope

Viewer mode only. Single Lit element `<casehub-diagram>`.

**Properties:**
- `yaml: string` — raw YAML string to render
- `src: string` — URL to fetch YAML from (alternative to `yaml` property)

**Lifecycle:**
1. On `yaml` or `src` change: parse → `toGraph()` → `toReactFlowGraph()` → `computeElkLayout()` → set nodes/edges on canvas
2. Registers case stencils on first connect (`registerCaseStencils()`)

**Internal structure:**
- Contains a `<pages-graph-canvas>` element
- Forwards `graph:node-click` and `graph:selection-change` events from the canvas

**No palette, property panel, or toolbar in Phase 2.**

### 4.2 Package structure

New package `components/casehub-diagram/` following blocks-ui's component layout convention:
- `src/casehub-diagram.ts` — Lit component
- `examples/index.html` — demo page with `document-processing.yaml`
- Standard `package.json`, `tsconfig.json`, `tsconfig.build.json`

Dependencies: `@casehubio/graph-stencil-case`, `@casehubio/graph-core`, `@casehubio/graph-renderer`, `lit`, `yaml`

## 5. File Structure

```
packages/graph-stencil-case/
  src/
    adapter/
      case-adapter.ts           ← toGraph() implementation
      case-adapter.test.ts      ← unit tests
      react-flow-transform.ts   ← GraphModel → React Flow types
      react-flow-transform.test.ts
    bridge/
      create-react-node-type.tsx ← Lit template → React component wrapper
    stencils/
      index.ts                  ← exports all stencil render fns + grammars
      register.ts               ← registerCaseStencils()
      binding.ts                ← Binding render function + grammar
      worker.ts                 ← Worker render function + grammar
      milestone.ts              ← Milestone render function + grammar
      goal.ts                   ← Goal render function + grammar
      subcase.ts                ← SubCase render function + grammar
    types/
      (unchanged from Phase 0)

components/casehub-diagram/
  src/
    casehub-diagram.ts
    casehub-diagram.test.ts
  examples/
    index.html
  package.json
  tsconfig.json
  tsconfig.build.json
```

## 6. Testing Strategy

1. **CaseAdapter.toGraph()** — parse `document-processing.yaml`, assert:
   - 5 workers, 6 bindings, 3 milestones, 1 goal created as nodes
   - 6 capability-dispatch edges (one per binding with `capability`)
   - Node properties carry the correct YAML values
   - External capability references produce annotation edges
2. **toReactFlowGraph()** — verify GraphModel → React Flow transformation preserves types and data
3. **Stencil render functions** — each returns valid TemplateResult given sample data, contains expected visual elements (trigger icon, capability badge, etc.)
4. **Example page** — visual integration test: `document-processing.yaml` renders as a connected graph

## 7. Dependencies

**graph-stencil-case additions:**
- `react` (peerDependency — for the bridge component)
- `react-dom` (peerDependency)
- `@casehubio/graph-renderer` (dependency — for `registerNodeType`, `computeElkLayout`)
- `@xyflow/react` (peerDependency — React Flow Node/Edge types)

**casehub-diagram (new package):**
- `@casehubio/graph-stencil-case`, `@casehubio/graph-core`, `@casehubio/graph-renderer`
- `lit`, `yaml`
