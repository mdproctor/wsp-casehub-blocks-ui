# Chain Navigator with Inline Editing — Design Spec

**Issue:** #234
**Branch:** `issue-234-chain-navigator-inline-edit`
**Date:** 2026-11-05

## Overview

An interactive chain navigation view for the relationship editor — walk through
relationship chains as an indented tree with inline CRUD. No graph rendering, no
complex layout — clickable agent rows with inline editing controls that show the
impact of edits immediately in context.

Adds a third tab ("Chain") to the existing `agent-relationship-editor`, sharing
the changeset model so edits in Chain and Table tabs are consistent.

## Architecture

Three layers, each independently testable:

### 1. Chain traversal utility (`graph-stencil-org`)

Pure data functions operating on raw `AgentRelationship[]`:

```ts
function findChainRoots(
  relationships: AgentRelationship[],
  agentId: string,
  kind: RelationshipKind
): string[]
```

Walks upward from `agentId` following reverse edges (where `targetAgentId`
matches) to find root agents — agents with no incoming edges of the given kind.
If the focal agent is itself a root, returns `[agentId]`. If the focal agent
appears in multiple disconnected chains, returns multiple roots.

```ts
function buildChainTree(
  relationships: AgentRelationship[],
  rootAgentId: string,
  kind: RelationshipKind
): ChainNode
```

Walks downward from `rootAgentId` following outgoing edges of the given kind.
Builds a tree with cycle detection via visited-set. When a node is revisited,
sets `isCycle: true` on that node and stops expanding.

These functions live in `packages/graph-stencil-org/src/adapter/chain-tree.ts`,
next to the existing `ego-subgraph.ts` and `derived-data.ts`. Exported from the
package index.

### 2. `chain-view` element (`agent-relationship-editor/src/chain-view.ts`)

A LitElement sub-component in the same package as the relationship editor,
following the `arc-view.ts` precedent. Unlike arc-view (read-only renderer),
chain-view is bidirectional: properties in, structured events out.

**Properties in:**
- `relationships: AgentRelationship[]` — full relationship set
- `roster: AgentRosterEntry[]` — agent lookup list
- `agents: Record<string, AgentDescriptor>` — rich agent data (avatar, profession)
- `agentId: string` — focal agent (highlighted in tree)
- `pendingOperations: ChainOperation[]` — pending edits from editor for rendering

**Events out (via `pages-event`, namespaced per protocol):**
- `chain:operation` — payload: `ChainOperation` — emitted on each inline edit
- `chain:kind-changed` — payload: `{ kind: RelationshipKind }` — informational

### 3. Editor orchestrator (`agent-relationship-editor.ts`)

The editor gains:
- `ViewTab = 'table' | 'chain' | 'diagram'` (was `'table' | 'diagram'`)
- `_pendingOperations: ChainOperation[]` — compound operation tracking
- Identity-based removal tracking replacing `_pendingRemovals: Set<number>`
- Flattening logic: `ChainOperation[]` → `RelationshipChangeset` on confirm

The editor listens for `chain:operation` events, appends to
`_pendingOperations`, derives flattened pending additions/removals, and passes
them back to chain-view for rendering. The Confirm/Reset buttons work across all
tabs — confirming dispatches the existing `relationship:changed` event with a
flattened `RelationshipChangeset`.

## Data Model

### ChainNode

Returned by the traversal utility:

```ts
interface ChainNode {
  agentId: string;
  children: ChainNode[];
  isCycle: boolean;
}
```

`isCycle` is `true` when traversal encounters a previously-visited agent. The
node is included (to show where the cycle closes) but `children` is empty.

### RelationshipIdentity

Content-based matching, replacing array index tracking:

```ts
interface RelationshipIdentity {
  sourceAgentId: string;
  targetAgentId: string;
  kind: RelationshipKind;
  scopeDomain?: string;
}
```

Two relationships with the same source, target, and kind but different scope
domains are distinct. The identity is used to match pending removals against the
relationship array regardless of order.

### ChainOperation

Groups compound mutations as atomic units:

```ts
interface ChainOperation {
  type: 'add-leaf' | 'add-root' | 'insert-between'
      | 'remove-disconnect' | 'remove-reconnect' | 'delete';
  removals: RelationshipIdentity[];
  additions: AgentRelationship[];
  description: string;
}
```

Operation examples:

| Operation | removals | additions |
|-----------|----------|-----------|
| Delete A→B | `[A→B]` | `[]` |
| Add leaf: A→C | `[]` | `[A→C]` |
| Insert C between A→B | `[A→B]` | `[A→C, C→B]` |
| Remove B, reconnect (B has children C, D) | `[A→B, B→C, B→D]` | `[A→C, A→D]` |
| Remove B, disconnect | `[A→B, B→C, B→D]` | `[]` |
| New chain root A→B | `[]` | `[A→B]` |

`ChainOperation` is internal to the editor. The external event payload on
confirm remains `RelationshipChangeset` (flattened additions/removals).

## Interaction Model

### Kind selector

Segmented control at the top of the Chain tab. Shows only kinds that have
relationships in the current dataset. BACKS_UP is excluded (pairs, not chains).
Defaults to the first populated kind. Changing kind rebuilds the tree.

### Tree rendering

Each node is a row with indentation per depth level:

```
▸ [avatar] Agent Name [role]          [+ insert] [×]
    ▸ [avatar] Child Agent [role]     [+ insert] [×]
        [avatar] Leaf Agent [role]    [+ add]    [×]
```

- `▸`/`▾` collapse/expand toggle for nodes with children
- Agent rendering: avatar + name + optional role badge (reuses the
  `_renderAgentCell` pattern from the editor)
- `[+ insert]` appears on hover between parent and child — opens inline agent
  picker to insert between the two
- `[+ add]` on leaf nodes — opens inline agent picker to add a new child
- `[×]` on any node — opens removal options inline

A `[+ new chain]` button below the tree starts a new chain with inline source
and target pickers.

### Focal agent highlighting

When the focal agent (`agentId`) is mid-chain, root discovery (D9) finds the
chain root and displays the full tree. The focal agent's row gets an accent
background highlight so the user sees their position in the chain context.

### Removal options

Clicking `[×]` on a non-leaf node shows inline options:

```
[Remove and reconnect] [Remove and disconnect] [Cancel]
```

- **Remove and reconnect:** parent inherits the removed node's children. Creates
  a compound `ChainOperation` with N+1 removals and N additions.
- **Remove and disconnect:** removes all edges involving the node in this chain.
  Creates a compound `ChainOperation` with removals only — the chain splits.

On a leaf node, clicking `[×]` directly creates a `delete` operation (single
removal, no reconnection choice needed).

### Insert picker

Clicking `[+ insert]` or `[+ add]` shows an inline dropdown of available agents
from the roster, filtered to exclude agents already present in the current chain
(prevents obvious cycles). Selecting an agent creates the appropriate
`ChainOperation`.

### Pending state rendering

- Pending removals: row gets strikethrough + red tint (matches table's
  `pending-removal` class)
- Pending additions: row gets green tint (matches `pending-addition` class)
- The tree re-renders structurally to show the "what-if" state: inserted nodes
  appear at their new position, removed nodes remain visible but struck through
- The Confirm/Reset bar (shared with table view) appears when any pending
  operations exist

### Cycle detection

**Display:** When traversal encounters a previously-visited node, it renders as
a visually distinct pill with a cycle indicator icon and a different border/color.
The node name is still visible. Expanding stops — no infinite recursion.

**Edit-time:** When an edit would create a new cycle, the pending state renders
the cycle marker immediately. A warning indicator appears on the pending
operation. The user sees the impact before confirming and can reset if
unintended. No hard block — some circular relationships may be intentional
(e.g. circular escalation).

## ARIA

- Chain view container: `role="tree"`,
  `aria-label="Chain navigator for [kind] relationships"`
- Each agent row: `role="treeitem"`, `aria-level="N"`,
  `aria-expanded="true|false"` for nodes with children
- Kind selector: `role="radiogroup"`,
  `aria-label="Relationship kind"`
- Inline edit controls: `role="group"`,
  `aria-label="Edit actions for [agent name]"`

**Keyboard navigation:**
- `ArrowDown` / `ArrowUp` — navigate between rows
- `ArrowRight` — expand collapsed node
- `ArrowLeft` — collapse expanded node
- `Enter` — toggle edit controls on focused row

## Testing Strategy

### Traversal utility (graph-stencil-org)

Pure unit tests in `chain-tree.test.ts`:
- Linear chain: A→B→C→D
- Branching tree: A→B, A→C, B→D, B→E
- Cycle detection: A→B→C→A
- Root discovery: mid-chain agent finds correct root
- Root discovery: agent in multiple chains returns multiple roots
- Empty relationships for kind: returns empty
- Single node (root with no children)

### chain-view (component tests)

- Renders tree structure with correct indentation and ARIA attributes
- Kind selector shows populated kinds, excludes BACKS_UP
- Focal agent highlighted
- Click insert: shows picker, emits `chain:operation` with correct type
- Click remove on leaf: emits delete operation
- Click remove on non-leaf: shows reconnect/disconnect options
- Pending state: removals struck through, additions green
- Cycle marker renders for cyclic nodes
- Keyboard navigation: arrow keys, enter
- Agent picker filters out agents already in chain

### Editor integration

- Chain tab renders chain-view with correct properties
- `chain:operation` events update `_pendingOperations`
- Cross-tab consistency: pending ops visible in both table and chain
- Confirm flattens operations to `RelationshipChangeset`
- Reset clears all pending operations
- Identity-based removal matching (order-independent)

## File Changes

| File | Change |
|------|--------|
| `packages/graph-stencil-org/src/adapter/chain-tree.ts` | New — `buildChainTree`, `findChainRoots`, `ChainNode` |
| `packages/graph-stencil-org/src/adapter/chain-tree.test.ts` | New — traversal unit tests |
| `packages/graph-stencil-org/src/index.ts` | Export chain-tree functions and types |
| `packages/graph-stencil-org/src/types.ts` | Add `ChainNode` type |
| `components/agent-relationship-editor/src/types.ts` | Add `RelationshipIdentity`, `ChainOperation` |
| `components/agent-relationship-editor/src/chain-view.ts` | New — chain view element |
| `components/agent-relationship-editor/src/chain-view.test.ts` | New — chain view tests |
| `components/agent-relationship-editor/src/agent-relationship-editor.ts` | Add Chain tab, `_pendingOperations`, identity-based tracking |
| `components/agent-relationship-editor/src/agent-relationship-editor.test.ts` | Add integration tests |
| `components/agent-relationship-editor/src/index.ts` | Export chain-view |

## References

- [Issue #234](https://github.com/casehubio/casehub-mosaic/issues/234) — feature request
- [Issue #233](https://github.com/casehubio/casehub-mosaic/issues/233) — relationship editor (landed)
- `components/agent-relationship-editor/src/agent-relationship-editor.ts` — existing editor with table+diagram tabs
- `components/agent-relationship-editor/src/arc-view.ts` — sub-element precedent
- `components/agent-relationship-editor/src/types.ts` — `RelationshipChangeset`, `AgentRosterEntry`
- `packages/graph-stencil-org/src/types.ts` — `AgentRelationship`, `RelationshipKind`
- `packages/graph-stencil-org/src/adapter/ego-subgraph.ts` — single-hop extraction
- `packages/graph-stencil-org/src/adapter/derived-data.ts` — `walkEscalation`, `supervisionSummary`
- `docs/protocols/mosaic/custom-event-namespace-prefix.md` — event naming protocol
- `docs/protocols/mosaic/component-customisation-pattern.md` — typed config + callbacks
- Decision review R1-02, R1-04 — changeset model revision (identity-based + compound ops)
- Decision review R1-06 — kind selection (per-kind navigation)
- Decision review R1-07 — traversal logic location
