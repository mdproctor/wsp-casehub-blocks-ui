# Decisions — Issue #234 Chain Navigator

## D1: Component structure

**Choice:** Inline in relationship editor — add `'chain'` to `ViewTab`, render the chain tree directly inside `agent-relationship-editor.ts` alongside existing table/diagram rendering. Chain traversal logic in a pure utility function.
**Alternatives:**
- Extracted chain sub-component — clean separation but more event wiring, inconsistent with how table tab is done (inline)
- Standalone top-level component — over-engineered, no other consumer exists
**Rationale:** Consistent with existing table tab pattern (inline rendering). Shares `_pendingAdditions`/`_pendingRemovals` state naturally without event proxying. Simpler.
**Trade-offs:** Editor file grows larger. Chain-specific rendering mixed with table rendering in one file.
**Sources:** components/agent-relationship-editor/src/agent-relationship-editor.ts (table tab is inline), diagram tab composes external component but chain is simpler
**Exploration:** quick
**Status:** captured

## D2: Chain tree data structure

**Choice:** Eager tree build on kind selection — traverse full relationship array once, build `ChainNode { agentId, children: ChainNode[], isCycle: boolean }` tree. Find roots (sources that are never targets for that kind). Cycle detection via visited set, marking cycle-back nodes.
**Alternatives:**
- Lazy traversal on expand — show only roots, expand on click. Harder to show full structure at a glance, roots still require full scan anyway.
**Rationale:** Relationship arrays are small (dozens). Tree maps directly to indented render. Showing full structure immediately is the point.
**Trade-offs:** Rebuilds on every relationship change, but O(n) where n = relationships of that kind.
**Sources:** ego-subgraph.ts (single-hop precedent), AgentRelationship[] flat array
**Exploration:** quick
**Status:** captured

## D3: Inline delete with reconnect

**Choice:** Multi-step pending changeset — "remove and reconnect" produces one removal + N additions in the existing pending model. Tree re-renders from `_effectiveRelationships` showing structural result immediately.
**Alternatives:**
- Compound operation type — `CompoundEdit { type: 'reconnect', removal, additions }` in changeset. Better UX label but changes the public `RelationshipChangeset` API for marginal benefit.
**Rationale:** Pending additions/removals already tell the full story. Tree view shows the structural result. No API change needed.
**Trade-offs:** User must understand that green additions are the reconnection — no single "reconnect" label in the confirm bar.
**Sources:** RelationshipChangeset type in events.ts, _pendingAdditions/_pendingRemovals in agent-relationship-editor.ts
**Exploration:** quick
**Status:** captured

## D4: BACKS_UP handling in chain tab

**Choice:** Show as pairs — BACKS_UP renders as flat `A ↔ B` pairs (depth-1 trees). Every agent is a root. Keeps one view for all relationship kinds.
**Alternatives:**
- Exclude BACKS_UP from chain tab — only directional kinds shown. Adds conditional logic and user must know which tab to check.
**Rationale:** Pairs are trivial to render in the tree. No exclusion logic. Consistent UX — all kinds in one place.
**Trade-offs:** BACKS_UP doesn't really "chain" — minor conceptual mismatch, but harmless.
**Sources:** Issue #234 relationship kinds table
**Exploration:** quick
**Status:** captured
