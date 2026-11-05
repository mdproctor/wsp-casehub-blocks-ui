# Decisions — Issue #234: Chain Navigator

## D1: Integration approach

**Choice:** Third tab ("Chain") in the existing `agent-relationship-editor`
**Alternatives:**
- Standalone component — would need its own changeset plumbing or awkward two-way sync
- Replace table for chain-forming kinds — reduces flexibility, removes table view for those kinds
**Rationale:** The editor already has a tab system (`ViewTab`), the chain view needs the same data and the same changeset model. A tab keeps everything unified.
**Trade-offs:** Editor file grows in complexity (mitigated by D4's sub-component approach)
**Sources:** `components/agent-relationship-editor/src/agent-relationship-editor.ts` — existing tab system at line 9, 376-378
**Exploration:** quick
**Status:** captured

## D2: Layout approach

**Choice:** Indented tree (vertical, file-explorer style)
**Alternatives:**
- Horizontal pill chain — breaks down with branching (SUPERVISES is a tree, not a linear chain)
**Rationale:** SUPERVISES is the primary use case and it branches. Indented tree handles both linear chains and branching trees. Each node gets a full row for inline edit controls.
**Trade-offs:** Less visually compact than horizontal pills for simple linear chains (ESCALATES_TO)
**Sources:** Issue #234 — ASCII diagram shows branching structure
**Exploration:** quick
**Status:** captured

## D3: Editing scope

**Choice:** All four editing operations in the first cut (delete, insert-between, add-to-leaf, new-chain-root)
**Alternatives:**
- Read-only + simple add/remove first, structural ops later — would deliver a view that doesn't differentiate from the diagram tab
**Rationale:** The chain navigator's value proposition IS inline editing. Without insert-between and remove-and-reconnect, it's just a read-only tree duplicating the diagram. The reconnection logic is the hardest part but also the main reason this view exists.
**Trade-offs:** Larger initial implementation scope
**Sources:** Issue #234 — "you see the impact of edits immediately in context"
**Exploration:** quick
**Status:** captured

## D4: Component architecture

**Choice:** Separate `chain-view` element in the same package
**Alternatives:**
- Inline in editor — editor is already 456 lines, chain logic adds 300+; hurts readability
- Standalone component in `components/` — duplicates changeset plumbing, unnecessary for a view scoped to the editor
**Rationale:** Follows the `arc-view` precedent. Editor stays a thin orchestrator (tabs, changeset, confirm/reset). Chain traversal, tree rendering, and inline editing are isolated in their own element.
**Trade-offs:** One more internal file to maintain, events between chain-view and editor need a contract
**Sources:** `components/agent-relationship-editor/src/arc-view.ts` — existing sub-element pattern
**Exploration:** quick
**Status:** captured

## D5: Changeset model (revised after R1-02, R1-04)

**Choice:** Shared changeset with identity-based tracking and compound operation grouping
**Alternatives:**
- Immediate dispatch per edit — inconsistent with table tab's batch-and-confirm semantics
- Keep existing index-based `Set<number>` tracking — breaks when chain-view references relationships by content, not position; can't represent compound operations
**Rationale:** Consistent UX across tabs. The existing index-based `_pendingRemovals: Set<number>` must be replaced with identity-based tracking (match by sourceAgentId + targetAgentId + kind + scope) so chain-view and table-view reference the same relationships regardless of array order. Compound operations (insert-between = 1 removal + 2 additions; remove-and-reconnect = N removals + M additions) are grouped as `ChainOperation` objects so the table view can display them as logical units and undo works atomically.
**Trade-offs:** Migration cost — existing table view and event dispatch must switch from index-based to identity-based removal tracking. Pending state rendering in the tree is complex (strikethrough for removals, green for additions, structural "what-if" re-rendering).
**Depends on:** D1 (third tab integration)
**Sources:** `agent-relationship-editor.ts` lines 220-221, review R1-02, R1-04
**Exploration:** quick → revised after review
**Status:** revised

## D6: Cycle detection UX

**Choice:** Inline cycle marker — show repeated node as a visually distinct pill with cycle icon, stop expanding
**Alternatives:**
- Cycle summary row — breaks visual flow, requires mental mapping back to tree position
**Rationale:** Keeps tree structure intact. You see exactly where the cycle closes and which relationship to delete to break it. The marked node makes the cycle's return point obvious.
**Trade-offs:** Subtle — users must notice the cycle marker among other nodes
**Sources:** Issue #234 — "show the cycle, don't loop"
**Exploration:** quick
**Status:** captured

## D7: Kind selection — per-kind navigation

**Choice:** One kind at a time, with a kind selector (dropdown or segmented control) at the top of the chain tab. BACKS_UP excluded (shown as pairs in table only, not navigable as a chain).
**Alternatives:**
- Multi-kind simultaneous view — creates a multigraph, not a tree; edges of different kinds from the same agent make the tree ambiguous
- Per-kind sub-views (multiple trees visible at once) — information overload, no clear focus
**Rationale:** The tree layout (D2) only works for single-kind navigation. `computeDerivedData` already builds kind-specific structures (escalation chains, supervision targets), confirming the data model is per-kind. The kind selector defaults to the first kind that has relationships for the focal agent.
**Trade-offs:** User must switch kinds to see different chain types. But this matches the table view, which groups by kind in sections.
**Depends on:** D2 (indented tree layout)
**Sources:** Review R1-06, `graph-stencil-org/src/adapter/derived-data.ts` — `walkEscalation`, `supervisionSummary`
**Exploration:** quick (review-prompted)
**Status:** captured

## D8: Traversal logic location

**Choice:** Transitive chain traversal utility in `graph-stencil-org`, operating on raw `AgentRelationship[]` — not in chain-view
**Alternatives:**
- Build traversal inside chain-view — duplicates domain logic from `computeDerivedData`, mixes data operations with UI rendering
- Reuse `computeDerivedData` via GraphModel adapter — adds a conversion step (toOrgGraph → computeDerivedData → extract), and `computeDerivedData` doesn't support all kinds transitively
**Rationale:** Chain traversal is a data operation, not a UI concern. A new `buildChainTree(relationships: AgentRelationship[], rootAgentId: string, kind: RelationshipKind): ChainNode` function in `graph-stencil-org` works directly on the relationship array, includes cycle detection, and is testable independently of the UI.
**Trade-offs:** Adds a new public API to graph-stencil-org. But the function is small, well-bounded, and the package already owns relationship data structures.
**Depends on:** D4 (component architecture — chain-view consumes the utility)
**Sources:** Review R1-07, `extractEgoSubgraph` (single-hop precedent), `computeDerivedData` (partial transitive traversal)
**Exploration:** quick (review-prompted)
**Status:** captured

## D9: Chain root discovery

**Choice:** Walk upward from the focal agent to find chain root(s). If the focal agent is mid-chain, display the full chain from root, with the focal agent highlighted.
**Alternatives:**
- Always start from the focal agent, only show downward — misses chain context above the focal agent
- Show all roots for the selected kind — too many trees when the kind has many disconnected chains
**Rationale:** The issue says "starting from any agent, follow a relationship kind." If the user's focal agent is mid-chain, they need to see both their upstream and downstream context. Walking upward requires reverse lookup (find relationships where targetAgentId = focal agent), which is a simple filter over the relationships array.
**Trade-offs:** For agents that appear in multiple disconnected chains of the same kind, the view shows all chains containing that agent. This could be multiple trees — acceptable since the focal agent is highlighted in each.
**Depends on:** D7 (per-kind navigation), D8 (traversal utility)
**Sources:** Review R1-10, Issue #234 — "starting from any agent"
**Exploration:** quick (review-prompted)
**Status:** captured

## D10: Edit-time cycle prevention

**Choice:** Soft warning — when an edit would create a cycle, show a warning indicator on the pending operation but allow it. The cycle marker (D6) renders immediately in the pending tree state so the user sees the impact before confirming.
**Alternatives:**
- Hard block — prevents the addition entirely; too restrictive since some circular relationships may be intentional (circular escalation)
- Post-hoc only — user doesn't see the cycle until after confirming; too late for prevention
**Rationale:** Cycles in SUPERVISES are always errors, but in other kinds may be intentional. A soft warning in the pending state lets the user see the cycle impact and choose to confirm or reset. The per-kind semantics (D7) could be extended later to make SUPERVISES a hard block.
**Trade-offs:** Users might confirm cycles accidentally. But the inline cycle marker (D6) makes them visually obvious.
**Depends on:** D6 (cycle detection UX), D7 (per-kind navigation)
**Sources:** Review R1-08
**Exploration:** quick (review-prompted)
**Status:** captured
