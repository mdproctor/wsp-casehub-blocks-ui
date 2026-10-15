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
