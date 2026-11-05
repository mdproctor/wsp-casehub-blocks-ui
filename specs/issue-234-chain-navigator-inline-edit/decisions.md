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

## D5: Changeset model

**Choice:** Shared changeset model with staged confirm
**Alternatives:**
- Immediate dispatch per edit — inconsistent with table tab's batch-and-confirm semantics
**Rationale:** Consistent UX across tabs. Edits in the chain tab appear as pending, switching to table shows the same pending state. Users can make several chain edits, review in table, and confirm as a batch.
**Trade-offs:** Pending state in a tree is visually more complex (dashed lines, strikethrough, what-if rendering of reconnections)
**Depends on:** D1 (third tab integration)
**Sources:** `agent-relationship-editor.ts` lines 220-221 — `_pendingAdditions`, `_pendingRemovals`
**Exploration:** quick
**Status:** captured

## D6: Cycle detection UX

**Choice:** Inline cycle marker — show repeated node as a visually distinct pill with cycle icon, stop expanding
**Alternatives:**
- Cycle summary row — breaks visual flow, requires mental mapping back to tree position
**Rationale:** Keeps tree structure intact. You see exactly where the cycle closes and which relationship to delete to break it. The marked node makes the cycle's return point obvious.
**Trade-offs:** Subtle — users must notice the cycle marker among other nodes
**Sources:** Issue #234 — "show the cycle, don't loop"
**Exploration:** quick
**Status:** captured
