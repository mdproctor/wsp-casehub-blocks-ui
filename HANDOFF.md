# HANDOFF — casehub-blocks-ui

## Last Session

Implemented #173 (SWF edge-click picker + palette drag-to-canvas). Two
bugs found and fixed during browser verification:

**Bug 1 — picker select did nothing:** The `_baseChooserSelect` arrow
function property capture pattern failed at runtime. Arrow function
class properties don't support `super` delegation. Fix: inlined both
edge-click and pane-click paths directly in `_onChooserSelect`, eliminating
the delegation entirely.

**Bug 2 — inserted node unreachable (nothing visible):** OWS 1.0 uses
name-based flow control (`then:` references), not array position.
`insertSwfTask` only did positional insert, leaving new tasks unreachable
when the edge was a `then:` reference. The SDK's `buildFlatGraph` skipped
unreachable tasks entirely — the node simply didn't appear. Fix: new
`spliceSwfTask` function that rewires `then:` references: changes the
source's `then:` to the new task name, sets the new task's `then:` to
the original target. Handles both direct `then:` and switch-case `then:`.

Also filed pages issues for long-term mixin refactoring:
- casehub-pages#467 — `_computeLayout` hook (eliminate `_fullRender` overrides)
- casehub-pages#468 — standard render template with default event wiring

## Known Issues Needing Attention

### Layout corruption on node reorder (CRITICAL for #170)

**Scenario:** In the Claim Review example, insert two Call nodes before
`humanReview` (via edge-click picker). Then drag the first to swap with
the second. The layout breaks — nodes overlap and edges cross incorrectly.

**Root cause:** `moveSwfTask` + `rewriteThenReferences` only rewires
the SPECIFIC edge source's `then:` reference, not ALL `then:` references
that point to the moved task. After `spliceSwfTask` creates a chain
(routeByRisk → newCall1 → newCall2 → humanReview), moving newCall1
after newCall2 doesn't update routeByRisk's switch case `then:` which
still points to `newCall1`. The SDK produces a graph with broken flow.

**Pre-existing layout overlaps:** The edge routing TDD test for
"Claim Review (DOWN)" already fails with:
```
Overlap: '/do/routeByRisk' overlaps '/do/autoApprove'
```
This is the `computeSwfStackLayout` issue that #170 targets.

**Recommendation for #170:** The `moveNodeToEdge` case in
`_applyGraphEdit` needs the same flow-aware `then:` rewiring that
`spliceSwfTask` provides. Currently it delegates to `moveSwfTask` which
only rewires one reference. A `moveSwfTaskOnEdge` (or extending
`moveSwfTask`) that finds ALL `then:` references to the moved task and
updates them would fix this.

### Palette drag green bar indicators not tested

The palette drag-to-canvas splice indicators (green bars on edges during
drag-over) were not verified. GraphCanvas has the infrastructure
(`getAddPlacement`, drop handler, splice indicator rendering) but the
SWF diagram's custom render might not wire the drag events correctly.
This is a lower-priority visual polish item.

### Pre-existing: degraded property editing

"Property editing unavailable — No YAML path for task node" appears for
the Claim Review example. `yamlPaths` from `buildYamlPaths` doesn't
match `buildFlatGraph` node IDs. Pre-existing — not caused by #173.

## Immediate Next Step

Advance to #170 (stack-column escape edge optimisation and pages
migration). The layout corruption on node reorder is the highest
priority fix. The pre-existing overlap issue also needs addressing.

## Cross-Module

Pages issues filed (separate repo):
- casehub-pages#467 — `_computeLayout` layout hook
- casehub-pages#468 — standard render template

## References

- `spliceSwfTask`: `packages/graph-stencil-swf/src/adapter/swf-yaml-editor.ts`
- Edge-click handler: `components/swf-diagram/src/swf-diagram.ts:90-122`
- `_applyGraphEdit` splitEdge: `components/swf-diagram/src/swf-diagram.ts:220-230`
- Design spec: `specs/issue-158-lsp-schema-refinements/2026-09-25-swf-edge-click-palette-drag-design.md`
- Plan: `plans/2026-09-25-swf-edge-click-palette-drag.md`
