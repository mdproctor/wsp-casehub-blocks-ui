# Agent Project Drill-Down

**Issue:** casehubio/blocks-ui#242
**Parent epic:** casehubio/blocks-ui#239
**Date:** 2026-10-19

## Overview

Wire drill-down from coordination-pipeline-view to agent-viewer. Clicking a drillable agent card emits a CustomEvent. Agent-viewer gains a Steps tab showing the agent's runtime execution steps. Breadcrumb segments emit navigation events for back-navigation.

## Changes

### 1. coordination-pipeline-view — drill-down event

Add click handler to step cards with `drillDown: true` (both lanes view and graph view). On click, emit a bubbling `coordination:drill-down` CustomEvent.

```typescript
interface DrillDownDetail {
  agentName: string;
  steps: CoordinationStep[];
}
```

- In **lanes view**: the `agentName` is the lane name, `steps` is the lane's step array from `SwimLanes[laneName]`.
- In **graph view**: drillable graph nodes carry the same data. The `agentName` comes from the step's `target` property. For multi-step lanes, the full lane step array is passed (not just the displayed first step).

The click handler is added to `.step-card.drillable` elements. Both `renderStepCard()` (lanes) and `_renderGraphNode()` (graph) need the handler, but the card itself is rendered by `renderStepCard()` which both views share — so the handler goes there.

**Context propagation:** The handler needs to know which lane a card belongs to. For lanes view, the lane name is available in `renderLane()`. For graph view, the `GraphNode` needs to carry the lane name. Add `laneName?: string` and `laneSteps?: CoordinationStep[]` to `GraphNode` so the graph view can emit the correct detail.

### 2. agent-viewer — Steps tab and breadcrumb events

#### Steps tab

Extend `AgentTab` type:
```typescript
type AgentTab = 'eidos' | 'pattern' | 'steps';
```

Extend `AgentViewerData`:
```typescript
interface AgentViewerData {
  agent: AgentDescriptor;
  pattern?: PatternSpec;
  breadcrumb?: string[];
  steps?: CoordinationStep[];
}
```

The Steps tab appears in the tab bar only when `data.steps` is provided (non-empty array). It renders each `CoordinationStep` as a step-card using the same visual style as coordination-pipeline-view.

**Step card rendering:** Import `CoordinationStep` types from coordination-types.ts. Replicate the step-card rendering inline in agent-viewer (the CSS is in coordination-pipeline-view's shadow DOM and can't be shared directly — duplicate the relevant card styles). The step cards are simpler here: no drill-down, no lane context, just the card with action, target, model badge, format badge.

#### Breadcrumb events

Add click handler to breadcrumb segments (all except the last, which is the current view). On click, emit `viewer:breadcrumb-back` CustomEvent:

```typescript
interface BreadcrumbBackDetail {
  index: number;
  segment: string;
}
```

The consumer (e.g., a workbench or page) catches this to navigate back — typically hiding the agent-viewer and showing the pipeline view.

### 3. Gallery update

Add section 6 to gallery.html: "Agent Drill-Down — Project View". Show agent-viewer with:
- `breadcrumb: ['KYC Pipeline', 'identity-extractor']`
- `agent`: the leonardo descriptor (reuse from section 1)
- `pattern`: the debate pattern (reuse from section 3)
- `steps`: the identity-extractor's lane steps from the KYC pipeline YAML

This demonstrates the full drill-down experience with all three tabs.

## Event Flow

```
User clicks drillable card in coordination-pipeline-view
  → coordination:drill-down { agentName, steps }
    → Consumer catches event, creates AgentViewerData, shows agent-viewer

User clicks breadcrumb pipeline name in agent-viewer
  → viewer:breadcrumb-back { index: 0, segment: 'KYC Pipeline' }
    → Consumer hides agent-viewer, shows pipeline view
```

## ARIA

- Drillable step cards: existing `cursor: pointer` and `↗ drill down` hint. Add `role="button"` and keyboard handler (Enter/Space) for accessibility.
- Steps tab: follows existing tab ARIA pattern (`role="tablist"` / `role="tab"` / `role="tabpanel"`).
- Breadcrumb: `role="navigation"` with `aria-label="Breadcrumb"`. Clickable segments get `role="link"`.

## Testing

1. **coordination-pipeline-view drill-down:**
   - Click drillable card → emits `coordination:drill-down` with correct agentName and steps
   - Non-drillable cards don't emit
   - Graph view drillable nodes emit same event
   - Keyboard activation (Enter/Space) on drillable cards

2. **agent-viewer Steps tab:**
   - Steps tab appears when data.steps is provided
   - Steps tab hidden when data.steps is undefined/empty
   - Steps tab renders correct number of step cards
   - Step cards show action, target, model, format badges

3. **agent-viewer breadcrumb:**
   - Click on non-final breadcrumb segment → emits `viewer:breadcrumb-back`
   - Final segment (current view) is not clickable
   - ARIA: navigation role, link roles on clickable segments

4. **Gallery:** Section 6 renders with three tabs visible and breadcrumb.

## References

- `packages/agent-viewer/src/coordination-pipeline-view.ts` — drill-down source
- `packages/agent-viewer/src/coordination-types.ts` — CoordinationStep, SwimLanes
- `packages/agent-viewer/src/agent-viewer.ts` — drill-down target, tab structure
- `packages/agent-viewer/src/coordination-graph-model.ts` — GraphNode (needs laneName/laneSteps)
- `casehub-pages/docs/mockups/step-review-viewer.html` section 15 — mockup
- casehubio/blocks-ui#242 — issue
- casehubio/blocks-ui#239 — parent epic
