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
