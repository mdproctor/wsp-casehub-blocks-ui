# Chain Navigator Tab — Design Spec

**Issue:** #234 — Interactive chain navigator with inline editing
**Date:** 2026-10-15
**Branch:** issue-234-chain-navigator

## Summary

A third tab ("Chain") in the relationship editor that renders relationship chains as indented trees with inline CRUD. Walks chains step by step with full avatar pills, showing the complete hierarchy for a selected relationship kind. All edits use the existing pending changeset model shared with the table tab.

## Architecture

The chain navigator is inline in `agent-relationship-editor.ts` — same pattern as the table tab. `ViewTab` becomes `'table' | 'chain' | 'diagram'`.

Core logic lives in a pure function in `chain-tree.ts`:

```typescript
interface ChainNode {
  agentId: string;
  children: ChainNode[];
  isCycle: boolean;
  relationshipIndex: number; // index into source relationships array, -1 for roots
  isPending: boolean;        // true if this edge is a pending addition
}

function buildChainForest(
  relationships: AgentRelationship[],
  kind: RelationshipKind
): ChainNode[]
```

Returns a forest (array of root nodes). Operates directly on `AgentRelationship[]` — no graph model needed.

**Root detection:** An agent is a root for kind K if it appears as `sourceAgentId` but never as `targetAgentId` in relationships of kind K. For BACKS_UP, every agent is a root (bidirectional pairs).

**Cycle detection:** Per-path `visited: Set<string>` during tree build. Cycle back-references emit `ChainNode` with `isCycle: true` and empty children.

The tree is rebuilt eagerly on kind selection and whenever `_effectiveRelationships` changes. O(n) where n = relationships of the selected kind.

## Kind Picker

Horizontal pill bar at the top of the chain tab. One pill per `RelationshipKind` with a count badge (number of relationships of that kind in `_effectiveRelationships`). Default selection: first kind that has relationships, or SUPERVISES.

## Tree Rendering

Indented tree with full avatar pills (thumbnail + name + profession badge). Each level indents by a fixed amount (24px). Connector arrows (`→`) between parent and child — the arrow is the clickable target for delete operations.

### Visual States

| State | Rendering |
|-------|-----------|
| Normal edge | Subtle connector arrow |
| Pending addition | Green-highlighted pill and connector |
| Pending removal | Struck-through, reduced opacity |
| Cycle back-reference | `↻` icon, dashed border, not expandable |
| Leaf node | `[+ add]` pill after last child |
| Between nodes | Hover reveals `[+]` insert button on connector |

Pending state styling matches the table tab's existing conventions (green for additions, struck-through for removals).

## Inline Edit Operations

All edits mutate `_pendingAdditions` / `_pendingRemovals` on the host editor — same state the table tab uses. One confirm/reset bar for both tabs.

### Delete (click connector arrow)

Inline action bar appears below the arrow:

- **Remove and reconnect** — removal of clicked edge + additions for parent → each grandchild. Multiple pending operations, all visible in pending state.
- **Remove and leave disconnected** — removal only.
- **Cancel** — dismisses action bar.

### Insert between (hover `[+]` on connector)

Inline agent picker dropdown (filtered to agents not in current chain path). Selecting an agent produces: removal of old edge + addition of parent → inserted + addition of inserted → child.

### Add to leaf (`[+ add]` pill)

Inline agent picker. Adds one pending addition from leaf → selected agent.

### Add new chain root (`[+ new chain]` button below forest)

Two inline agent pickers in sequence: source then target. Adds one pending addition.

### Agent Picker

Filtered dropdown of `roster` entries with avatar thumbnails. Excludes agents already in the current chain path to prevent creating cycles via the UI.

## BACKS_UP Handling

BACKS_UP relationships render as flat pairs (`A ↔ B`). Every agent is a root. The tree builder handles this naturally — depth-1 trees. Same editing affordances apply (delete, add to leaf).

## ARIA

| Element | Attributes |
|---------|------------|
| Tree container | `role="tree"` |
| Tree node | `role="treeitem"`, `aria-expanded`, `aria-level` |
| Connector arrow | `role="button"`, `aria-label="Remove relationship from {source} to {target}"` |
| Insert button | `role="button"`, `aria-label="Insert agent between {source} and {target}"` |
| Add-to-leaf | `role="button"`, `aria-label="Add relationship from {agent}"` |
| Kind picker | `role="tablist"` / `role="tab"`, `aria-selected` |
| Cycle indicator | `aria-label="Cycle detected — {agent} already appears earlier in this chain"` |
| Inline action bar | `role="group"`, `aria-label="Relationship action"` |

## Events

Per `custom-event-namespace-prefix` protocol:
- `chain:kind:selected` — kind pill clicked (detail: `{ kind: RelationshipKind }`)

Edit operations mutate pending state directly on the host editor — no edit events. The existing `relationship:changed` event fires on confirm.

## Testing

### chain-tree.test.ts (pure function)

- Linear chain builds correctly
- Branching tree with multiple children
- Cycle detection marks back-references
- BACKS_UP renders as pairs
- Empty relationships returns empty forest
- Pending additions reflected in tree
- Pending removals excluded from tree
- Root detection (source-only agents)

### agent-relationship-editor.test.ts (component — chain tab)

- Kind picker renders with correct counts
- Tree renders at correct depth with indentation
- Delete shows action bar with reconnect/disconnect/cancel
- Remove-and-reconnect produces correct pending ops (1 removal + N additions)
- Remove-and-disconnect produces correct pending ops (1 removal)
- Insert-between produces correct pending ops (1 removal + 2 additions)
- Add-to-leaf produces correct pending addition
- Add-new-chain produces correct pending addition
- Cycle nodes render with indicator and are not expandable
- Agent picker excludes agents in current path
- ARIA: role, aria-label, aria-expanded, aria-level on every node

## Files Modified

| File | Change |
|------|--------|
| `components/agent-relationship-editor/src/agent-relationship-editor.ts` | Add `'chain'` to `ViewTab`, kind picker, tree rendering, inline edit UI |
| `components/agent-relationship-editor/src/chain-tree.ts` | New — `buildChainForest`, `ChainNode` type |
| `components/agent-relationship-editor/src/chain-tree.test.ts` | New — pure function tests |
| `components/agent-relationship-editor/src/agent-relationship-editor.test.ts` | Chain tab component tests |
| `components/agent-relationship-editor/demo.html` | Chain tab in demo |

## References

- Issue #234 — interactive chain navigator with inline editing
- Issue #233 — relationship editor (landed)
- `components/agent-relationship-editor/src/agent-relationship-editor.ts` — existing editor
- `components/agent-relationship-editor/src/types.ts` — AgentRelationship, RelationshipKind
- `components/agent-relationship-editor/src/events.ts` — RelationshipChangeset
- `packages/graph-stencil-org/src/adapter/ego-subgraph.ts` — single-hop extraction (chain traversal operates at a lower level)
- `docs/protocols/blocks-ui/custom-event-namespace-prefix.md` — event naming convention
- `docs/protocols/blocks-ui/component-customisation-pattern.md` — no slots for content
