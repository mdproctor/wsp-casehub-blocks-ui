# HANDOFF — casehub-blocks-ui

**Branch:** main (no active branch)
**Date:** 2026-08-04
**Issues:** #103 Phase 7 closed

## What landed

Phase 7 runtime overlay (#103) — 3 squashed commits, 21 files changed, +587/-170 lines. Pushed to both fork and upstream.

- Stencil API migration: all 5 case stencils now use pages StencilDescriptor API (new render signature, registerStencil, deleted old createReactNodeType bridge and local toReactFlowGraph)
- RuntimeAdapter: toDecorations() pure function with active-worst-first PlanItem aggregation, terminal severity tiebreaker, tooltip breakdown, milestone mapping, unknown status fallback
- Badge mappings: all 9 TaskStatus states + 3 MilestoneLifecycleStatus states
- casehub-diagram: property-based runtimeState, design/runtime mode toggle, decoration flow via toReactFlowGraph, staleness indicator, OBSOLETE opacity

## Open items

- #104: update consumer guide with Phase 7 runtime overlay API
- Phase 5 (SWF drill-down) blocked on @openworkflowspec/sdk supporting bare do: task lists
- Phase 6 (Work Registry) not in epic scope

## Key decisions

- Property-based runtime data (not PushSource — component is transport-agnostic)
- MilestoneLifecycleStatus has 3 states not 5 (engine is authoritative)
- Decorations applied in both _fullRender and _updateWithoutLayout paths

## Dependencies

- pages#277 (NodeDecoration types) and decoration rendering pipeline must be synced to .casehub-packages/ before blocks-ui compiles against new APIs
