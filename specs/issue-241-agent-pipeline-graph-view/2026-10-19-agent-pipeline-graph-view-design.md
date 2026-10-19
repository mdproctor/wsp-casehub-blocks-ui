# Agent Pipeline — Graph View (fan-out/fan-in)

**Issue:** casehubio/blocks-ui#241
**Parent epic:** casehubio/blocks-ui#239
**Date:** 2026-10-19

## Overview

Add a graph view mode to `coordination-pipeline-view` that renders the pipeline as a tiered topology — nodes as cards, CSS connectors between tiers, a Lanes/Graph toggle in the header, and auto-detection of the preferred view based on pipeline shape.

## Component Changes

### viewMode property

Add `viewMode: 'lanes' | 'graph'` reactive property to `CoordinationPipelineView`. Defaults to the result of `detectPreferredView(phases)`. Externally settable to override auto-detection.

### Header toggle

The existing `.pipeline-header` gains a two-segment pill toggle on the right side (before the meta badges). Active segment has `blue-dim` background with blue text; inactive segment has dim background. Clicking toggles `viewMode`.

```
┌──────────────────────────────────────────────────┐
│ ⫘  KYC VERIFICATION          [Lanes] [Graph]  3 agents · 2 barriers │
└──────────────────────────────────────────────────┘
```

### Render dispatch

`render()` delegates to `renderLanesView()` (existing logic, extracted) or `renderGraphView()` (new) based on `viewMode`.

## Tier Assignment

A pure function `assignTiers(phases: PhaseItem[]): Tier[]` walks the phases array and produces an ordered list of tiers and connectors.

### Types

```typescript
interface GraphNode {
  step: CoordinationStep;
  routing?: MatchRouting;
  colorHint?: 'green' | 'amber' | 'red';
}

interface GraphTier {
  kind: 'tier';
  nodes: GraphNode[];
  labels?: string[];  // edge labels for fan-out (lane names)
}

interface GraphConnector {
  kind: 'connector';
  type: 'flow' | 'fan-out' | 'fan-in' | 'outcomes';
  labels?: string[];
  barrier?: {
    type: 'signal' | 'quorum';
    name: string;
    description?: string;
    required?: number;
    of?: number;
  };
}

type TierElement = GraphTier | GraphConnector;
```

### Algorithm

Walk `PhaseItem[]` sequentially:

1. **`step` (no routing)** — single-node tier. If the next phase is `parallel`, emit a `fan-out` connector with lane names as labels.
2. **`parallel`** — multi-node tier. Each lane's first step becomes a node. If a lane has multiple steps, they stack within the same node card (description list).
3. **`signal`** — emit a `flow` connector with signal barrier metadata.
4. **`quorum`** — emit a `fan-in` connector with quorum barrier metadata.
5. **`step` with `routing` (match)** — single-node tier for the step, then an `outcomes` connector, then a multi-node tier with one node per match case. Color hints derived from match pattern names: `"approve"` → green, `"escalate"` → amber, `"reject"` → red. Unrecognised patterns use default styling.

### Example: KYC Pipeline

```
PhaseItem[]                    → TierElement[]
─────────────                    ──────────────
step: classify-document        → Tier([classify-document])
                               → Connector(fan-out, labels: [identity, financial, address])
parallel: 3 lanes              → Tier([identity-extractor, financial-extractor, address-extractor])
                               → Connector(fan-in, barrier: quorum 2/3)
step: risk-assessor + match    → Tier([risk-assessor])
                               → Connector(outcomes)
match cases: approve/escalate  → Tier([approve, escalate→senior-reviewer, reject])
  /reject                         with colorHints
```

## Graph Rendering

### Tier layout

Each `GraphTier` renders as a CSS flex row:

```css
.graph-tier {
  display: flex;
  justify-content: center;
  gap: 16px;
  padding: 8px 32px;
}
```

Nodes within a tier reuse the existing `.step-card` styles. Outcome nodes (with `colorHint`) get a colored left border matching the pattern:

| colorHint | Border | Background |
|-----------|--------|------------|
| green | `#a6e3a1` | `rgba(166, 227, 161, 0.08)` |
| amber | `#f9e2af` | `rgba(249, 226, 175, 0.08)` |
| red | `#f38ba8` | `rgba(243, 139, 168, 0.08)` |

Decision nodes (step with match routing) get `border: 2px solid var(--amber)` for visual emphasis.

### Connectors

Each `GraphConnector` renders as a CSS element between tier rows:

```css
.graph-connector {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 4px 0;
  color: #585b70;
}
```

**Flow connector (1:1):** A centered chevron `▼` or thin dotted vertical line.

**Fan-out connector (1:N):** A horizontal row of edge labels as small badges, with a spreading visual indicator. Labels use lane names from the parallel block.

```
         ▼
  ┌──────┼──────┐
  identity  financial  address
```

Rendered as:

```css
.fan-out-labels {
  display: flex;
  gap: 12px;
  justify-content: center;
}
.edge-label {
  font-size: 10px;
  color: #7f849c;
  padding: 2px 8px;
  background: rgba(255, 255, 255, 0.04);
  border-radius: 3px;
}
```

**Fan-in / quorum connector (N:1):** Converging indicator + quorum barrier if present. Reuses existing `.barrier-bar.quorum` styling but centered within the graph layout:

```css
.graph-quorum {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 6px 16px;
  background: rgba(137, 180, 250, 0.06);
  border: 1px dashed #89b4fa;
  border-radius: 4px;
}
```

**Outcomes connector:** A thin dotted line or chevron, no special styling — the color-coded outcome nodes below are self-explanatory.

### Graph container

```css
.graph-view {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0;
  padding: 24px 32px;
}
```

## Auto-Switch Logic

```typescript
function detectPreferredView(phases: PhaseItem[]): 'lanes' | 'graph' {
  const hasParallel = phases.some(p => p.type === 'parallel');
  const hasQuorum = phases.some(p => p.type === 'quorum');
  return (hasParallel && hasQuorum) ? 'graph' : 'lanes';
}
```

Returns `'graph'` when the pipeline has fan-out/fan-in topology (parallel + quorum). Returns `'lanes'` for purely sequential or simple parallel pipelines without convergence.

## ARIA

- Graph view container: `role="img"` with `aria-label="Pipeline topology: <name>"`.
- Individual node cards: retain existing card semantics (clickable cards keep `role="button"` when drillable).
- Toggle: `role="radiogroup"` with two `role="radio"` buttons, `aria-checked` on the active one.

## Testing

1. **Tier assignment** — unit tests for `assignTiers()`:
   - Linear pipeline (steps only) → one tier per step
   - Fan-out/fan-in (step → parallel → quorum → step) → proper tier grouping with connectors
   - Match routing → decision tier + outcomes tier with color hints
   - Multiple parallel blocks → multiple fan-out/fan-in sequences

2. **Auto-switch detection** — unit tests for `detectPreferredView()`:
   - Parallel + quorum → `'graph'`
   - Parallel without quorum → `'lanes'`
   - No parallel → `'lanes'`

3. **Component rendering** — integration tests:
   - Default view matches auto-detection
   - Toggle switches between views
   - Graph view renders correct number of tiers and connectors
   - Outcome nodes have correct color-coded styling

4. **ARIA** — assertions on `role`, `aria-label`, `aria-checked` for toggle.

5. **Gallery** — update gallery.html to show the graph view alongside the existing lanes view.

## References

- `packages/agent-viewer/src/coordination-pipeline-view.ts` — existing component
- `packages/agent-viewer/src/coordination-types.ts` — CoordinationPipeline, PhaseItem types
- `casehub-pages/docs/mockups/step-review-viewer.html` section 14 — graph view mockup
- `casehub-pages/docs/mockups/step-review-viewer.html` section 13 — swim lane mockup (contrast)
- casehubio/blocks-ui#241 — issue
- casehubio/blocks-ui#239 — parent epic
