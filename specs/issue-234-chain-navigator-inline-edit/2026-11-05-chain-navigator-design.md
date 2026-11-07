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
  kind: RelationshipKind,
  extendedKind?: string
): string[]
```

Walks upward from `agentId` following reverse edges (where `targetAgentId`
matches) to find root agents — agents with no incoming edges of the given kind.
When `extendedKind` is provided (for `kind: 'EXTENDED'`), only edges matching
both `kind` and `extendedKind` are followed. If the focal agent is itself a
root, returns `[agentId]`. If the focal agent appears in multiple disconnected
chains, returns multiple roots. If the upward walk revisits a node (fully cyclic
chain with no natural root), returns `[agentId]` — the focal agent becomes the
entry point.

```ts
function buildChainTree(
  relationships: AgentRelationship[],
  rootAgentId: string,
  kind: RelationshipKind,
  extendedKind?: string
): ChainNode
```

Walks downward from `rootAgentId` following outgoing edges of the given kind.
When `extendedKind` is provided, only edges matching both `kind` and
`extendedKind` are followed — this ensures "Mentors (Extended)" builds a tree
from mentors edges only, not all EXTENDED edges. Builds a tree with cycle
detection via **ancestor-path tracking** (not a flat visited-set). The traversal maintains the current path from root to the node
being expanded. A node is marked `isCycle: true` only if it appears in its own
ancestor path — i.e., there is a path from the node back to itself. This
correctly distinguishes DAG convergence from cycles:

- **Convergence** (A→B→D, A→C→D): D appears under both B and C but is NOT
  cyclic. D is fully expanded under both parents. The second occurrence is
  marked `isConvergent: true` to signal a shared node.
- **Cycle** (A→B→C→A): A appears in its own ancestor path. Marked
  `isCycle: true`, children not expanded.

This matters for SUPERVISES and DELEGATES_TO, which form DAGs in real org
structures (e.g., matrix management). The existing `walkEscalation` in
`derived-data.ts` uses a flat visited-set, which is correct there because
`escalatesTo` is `Map<string, string>` (single-valued — DAG convergence is
impossible for escalation chains).

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
- `tenancyId: string` — tenancy context for constructing new `AgentRelationship`
  objects in additions (required field on `AgentRelationship`)
- `pendingOperations: ChainOperation[]` — pending edits from editor for rendering

**Events out (via `pages-event`):**
- `chain:operation` — payload: `ChainOperation` — emitted on each inline edit
- `chain:kind-changed` — payload: `{ kind: RelationshipKind, extendedKind?: string }` — informational

These events use `bubbles: true` but **not** `composed: true` — they are
internal communication between chain-view and the editor parent within the
editor's shadow DOM. Because chain-view dispatches on its host element (via
`this.dispatchEvent`), the event enters the editor's shadow DOM directly. The
custom-event-namespace-prefix protocol (PP-20260930-501ce1) scopes to
`composed: true` events and does not apply here. The `chain:` prefix is a
readability convention, not protocol-mandated.

### 3. Editor orchestrator (`agent-relationship-editor.ts`)

The editor gains:
- `ViewTab = 'table' | 'chain' | 'diagram'` (was `'table' | 'diagram'`)
- `_pendingOperations: ChainOperation[]` — unified operation tracking for both
  table and chain views
- Removal of `_pendingRemovals: Set<number>` and `_pendingAdditions:
  AgentRelationship[]` — replaced by `_pendingOperations`

**Unified tracking model:** Both table and chain views produce
`ChainOperation` objects. Table view removals become `ChainOperation { type:
'delete', removals: [identity], additions: [] }`. Table view additions (from
the add-form) become `ChainOperation { type: 'add-leaf', removals: [],
additions: [rel] }`. This replaces the separate `_pendingRemovals: Set<number>`
and `_pendingAdditions: AgentRelationship[]`.

**Migration of table view internals:**
- `_markForRemoval(index)` → looks up `this.relationships[index]`, derives its
  `RelationshipIdentity`, pushes a `delete` ChainOperation
- `_isPendingRemoval(rel)` → checks if any operation's removals match the
  relationship's identity key
- `_isPendingAddition(rel)` → checks if any operation's additions match the
  relationship's identity key
- `_confirm()` → calls `flattenOperations(this._pendingOperations)` to produce
  `RelationshipChangeset`

**Flattening algorithm:** `flattenOperations(ops: ChainOperation[])` →
`RelationshipChangeset`. Processes operations sequentially:

1. For each operation's removals: if the identity matches a prior addition,
   cancel the addition (net zero). Otherwise add to the removal set
   (deduplicated by identity key).
2. For each operation's additions: if the identity matches a prior removal,
   cancel the removal (net zero). Otherwise add to the additions list
   (deduplicated by identity key).
3. Return `{ additions: [...additionMap.values()], removals:
   [...removalMap.values()] }` where removals are resolved back to full
   `AgentRelationship` objects via identity matching against the original
   relationship array.

This correctly handles: duplicate removals (idempotent), add-then-remove
(cancels), remove-then-re-add (cancels).

**Confirmed-relationship suppression:** as a final step, additions whose
identity matches a confirmed relationship (one present in the original
`this.relationships` array and not being removed in the same changeset) are
suppressed. This is a safety net for cases where the operation builder produces
additions that duplicate existing edges (e.g., reconnect when the parent already
has a direct edge to a child — see R2-02).

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
  isConvergent: boolean;
}
```

- `isCycle` is `true` when the agent appears in its own ancestor path. The
  node is included (to show where the cycle closes) but `children` is empty.
- `isConvergent` is `true` when the agent appears elsewhere in the tree (DAG
  convergence). The node is fully expanded — children are populated. A subtle
  visual indicator (e.g., a link icon) signals the shared node.

### RelationshipIdentity

Content-based matching, replacing array index tracking:

```ts
interface RelationshipIdentity {
  sourceAgentId: string;
  targetAgentId: string;
  kind: RelationshipKind;
  extendedKind?: string;
  scope?: RelationshipScope;
}
```

The identity includes all fields that distinguish semantically different
relationships between the same pair of agents:

- `extendedKind` — for `kind: 'EXTENDED'`, this is the sub-type discriminator.
  Two relationships A→B with `extendedKind: 'mentors'` vs `'reviews'` are
  distinct.
- `scope` — the full `RelationshipScope` (capabilityName, domain, custom). Two
  relationships with the same source/target/kind but different scope (e.g., "A
  delegates billing to B" vs "A delegates shipping to B") are distinct.

**Intentionally excluded:**
- `kindVocabulary` — metadata about which vocabulary defines the kind string,
  not a relationship discriminator. Two relationships with the same `kind` but
  different `kindVocabulary` would be a data inconsistency, not two legitimate
  relationships.
- `attestation` — payload on the relationship, not identity. You don't have two
  "A supervises B" relationships distinguished only by attestation grants.
- `tenancyId` — all relationships in the editor share the same tenancy.

A helper `identityKey(id: RelationshipIdentity): string` produces a stable
string key for Map/Set operations. The identity is used to match pending
removals against the relationship array regardless of order.

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

### Metadata propagation rules

Structural operations that remove a relationship and create new ones must
define what happens to the removed relationship's metadata (`scope`,
`attestation`, `extendedKind`, `kindVocabulary`):

**Insert C between A→B** (remove A→B, add A→C and C→B):
- `scope`: both A→C and C→B inherit A→B's scope. The chain operates within a
  scope domain; inserting an intermediate preserves that context.
- `attestation`: NOT inherited. Attestation grants are specific to a
  source-target pair. A's attestation for B does not automatically extend to C.
- `extendedKind`, `kindVocabulary`: inherited (same relationship sub-type).

**Remove B, reconnect** (remove A→B + B→C + B→D, add A→C and A→D):
- `scope`: A→C inherits from **B→C** (not A→B). The scope on B→C describes
  what B was supervising C about. When A takes over B's children, A steps into
  B's role with respect to those children.
- `attestation`: NOT inherited.
- `extendedKind`, `kindVocabulary`: inherited from the child relationships
  (B→C, B→D).

These defaults are applied automatically when constructing the `additions`
array in `ChainOperation`. The user can subsequently edit scope/attestation on
the new relationships via the table view.

## Interaction Model

### Kind selector

Segmented control at the top of the Chain tab. Shows only kinds that have
relationships in the current dataset. BACKS_UP is excluded (pairs, not chains).
Defaults to the first populated kind. Changing kind rebuilds the tree.

**EXTENDED sub-grouping:** EXTENDED relationships use `extendedKind` to specify
the actual relationship type (e.g., `extendedKind: 'mentors'`,
`extendedKind: 'reviews'`). Each distinct `extendedKind` value appears as a
separate selectable entry in the kind selector (e.g., "Mentors (Extended)",
"Reviews (Extended)"). Lumping all EXTENDED relationships under a single
"EXTENDED" option would produce a meaningless tree mixing unrelated chain
semantics. The kind selector groups by `(kind, extendedKind)` rather than
`kind` alone.

### Tree rendering

**Multiple roots:** `findChainRoots` may return multiple roots when the focal
agent appears in multiple disconnected chains of the same kind (e.g., an agent
supervised by two independent root agents). Each root produces a separate tree.
Trees are displayed stacked vertically with a subtle separator between them. The
focal agent is highlighted in all trees where it appears. All trees share the
same `_pendingOperations` state — editing in one tree is reflected in all
trees. The `[+ new chain]` button creates a new root, adding another tree.

Each node is a row with indentation per depth level:

```
▸ [avatar] Root Agent [role]                    [+ add] [×]
    ▸ [avatar] Child Agent [role]   [+ insert] [+ add] [×]
        [avatar] Leaf Agent [role]  [+ insert] [+ add] [×]
```

- `▸`/`▾` collapse/expand toggle for nodes with children
- Agent rendering: avatar + name + optional role badge (reuses the
  `_renderAgentCell` pattern from the editor)
- `[+ insert]` button on each **non-root** node's row — inserts a new agent
  between that node's parent and that node. Opens an inline agent picker on
  click. This is a per-edge operation: clicking `[+ insert]` on child B (whose
  parent is A) inserts between A and B specifically, producing `removals: [A→B],
  additions: [A→C, C→B]`. Root nodes have no parent, so no insert button.
- `[+ add]` button on every node — opens inline agent picker to add a new child
  below this node
- `[×]` on any node — opens removal options inline

A `[+ new chain]` button below the tree starts a new chain with inline source
and target pickers.

### Focal agent highlighting

When the focal agent (`agentId`) is mid-chain, root discovery (D9) finds the
chain root and displays the full tree. The focal agent's row gets an accent
background highlight so the user sees their position in the chain context.

### Removal options

Clicking `[×]` on a non-leaf, non-root node shows inline options:

```
[Remove and reconnect] [Remove and disconnect] [Cancel]
```

- **Remove and reconnect:** parent inherits the removed node's children. Creates
  a compound `ChainOperation` with N+1 removals and N additions. The operation
  builder skips additions where the parent already has a confirmed edge to
  the child (prevents redundant additions — see R2-02).
- **Remove and disconnect:** removes all edges involving the node in this chain.
  Creates a compound `ChainOperation` with removals only — the chain splits.

Clicking `[×]` on a **root node with children** shows only:

```
[Remove and disconnect] [Cancel]
```

Reconnect is hidden because a root has no parent to inherit children. Reconnect
on a root would produce removals without additions — identical to disconnect.

On a leaf node, clicking `[×]` directly creates a `delete` operation (single
removal, no reconnection choice needed).

### Insert picker

Clicking `[+ insert]` or `[+ add]` shows an inline dropdown of available agents
from the roster. The filter excludes only agents that would create a cycle at
the specific insertion point, not all agents present in the chain:

- **Add child** (adding child to node X): exclude ancestors of X (including X
  itself). Agents elsewhere in the tree are valid — DAG convergence is
  legitimate.
- **Insert above** (clicking `[+ insert]` on node B whose parent is A — inserts
  between A and B): exclude ancestors of A (including A) and B itself. Inserting
  B between A and B would create a self-loop; inserting an ancestor of A would
  create a cycle.

This correctly handles DAG structures where an agent appears under multiple
parents. Selecting an agent creates the appropriate `ChainOperation`.

### Pending state rendering

- Pending removals: row gets strikethrough + red tint (matches table's
  `pending-removal` class)
- Pending additions: row gets green tint (matches `pending-addition` class)
- The tree re-renders structurally to show the "what-if" state: inserted nodes
  appear at their new position, removed nodes remain visible but struck through
- **Pending operations panel:** below the tree, a list shows each operation
  with its description and an individual dismiss button `[×]`. Dismissing an
  operation removes it from `_pendingOperations` and reverses all its effects
  atomically (important for compound operations like insert-between, which
  affect multiple rows). This replaces the need for per-row toggle on compound
  operations.
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
  `aria-expanded="true|false"` for nodes with children,
  `aria-setsize="M"` and `aria-posinset="K"` to communicate set position
- Kind selector: `role="radiogroup"`,
  `aria-label="Relationship kind"`
- Inline edit controls: `role="group"`,
  `aria-label="Edit actions for [agent name]"`

**Roving tabindex:** Only the currently focused treeitem has `tabindex="0"`;
all other treeitems have `tabindex="-1"`. This ensures Tab moves focus out of
the tree (to the next focusable element), not through every row. The chain-view
manages focused-node state and updates tabindex on arrow key navigation.

**Keyboard navigation:**
- `ArrowDown` / `ArrowUp` — navigate between visible rows (roving tabindex)
- `ArrowRight` — expand collapsed node; if already expanded, move to first child
- `ArrowLeft` — collapse expanded node; if already collapsed, move to parent
- `Home` — focus the first visible node in the tree
- `End` — focus the last visible node in the tree
- `Enter` — toggle edit controls on focused row

**Focus management for inline controls:** When removal options or insert
picker appear, focus moves to the first actionable control within the inline
group. `Escape` dismisses the inline controls and returns focus to the
originating row. While inline controls are visible, `Tab` cycles within the
control group (not the tree).

## Testing Strategy

### Traversal utility (graph-stencil-org)

Pure unit tests in `chain-tree.test.ts`:
- Linear chain: A→B→C→D
- Branching tree: A→B, A→C, B→D, B→E
- **DAG convergence**: A→B→D, A→C→D — D fully expanded under both parents,
  `isConvergent: true` on second occurrence, `isCycle: false`
- Cycle detection: A→B→C→A — path-based, not flat visited-set
- **Convergence vs cycle**: A→B→D, A→C→D, D→A — D is convergent under C but
  A is cyclic under D
- Root discovery: mid-chain agent finds correct root
- Root discovery: agent in multiple chains returns multiple roots
- **EXTENDED sub-grouping**: mixed `extendedKind` values ('mentors', 'reviews')
  — `buildChainTree` with `extendedKind: 'mentors'` follows only mentors edges,
  ignores reviews edges
- Empty relationships for kind: returns empty
- Single node (root with no children)

### chain-view (component tests)

- Renders tree structure with correct indentation and ARIA attributes
- Kind selector shows populated kinds, excludes BACKS_UP
- Focal agent highlighted
- Click `[+ insert]` on non-root node: shows picker, emits `chain:operation`
  with insert-between type for parent→this edge
- Click `[+ add]` on any node: shows picker, emits add-leaf operation
- Click remove on leaf: emits delete operation
- Click remove on non-leaf non-root: shows reconnect/disconnect options
- Click remove on root with children: shows disconnect only (no reconnect)
- Pending state: removals struck through, additions green
- Cycle marker renders for cyclic nodes
- Keyboard navigation: arrow keys, enter
- Agent picker for add-leaf excludes ancestors of insertion point; non-ancestor
  tree agents remain available (DAG convergence is valid)
- Agent picker for insert-between excludes ancestors, self, and target

### Editor integration

- Chain tab renders chain-view with correct properties (including `tenancyId`)
- `chain:operation` events update `_pendingOperations`
- Cross-tab consistency: pending ops visible in both table and chain
- Table view removals produce `ChainOperation { type: 'delete' }`
- Confirm flattens operations to `RelationshipChangeset`
- Flattening: duplicate removals deduplicated, add-then-remove cancels,
  remove-then-re-add cancels
- Reset clears all pending operations
- Identity-based removal matching (order-independent)
- Individual operation dismissal removes single operation and reverses effects
- Metadata propagation: insert-between inherits scope, not attestation
- Reconnect suppresses additions that duplicate confirmed relationships
- Multiple root rendering: stacked trees, focal agent highlighted in all

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
- Future: consolidation of `walkEscalation` with `buildChainTree`/`findChainRoots` (see issue #TBD)
