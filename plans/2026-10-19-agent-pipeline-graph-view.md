# Agent Pipeline — Graph View Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #241 — Agent Pipeline — Graph View (fan-out/fan-in)
**Issue group:** #241 (part of epic #239)

**Goal:** Add a graph view mode to coordination-pipeline-view that renders pipelines as tiered topology cards with CSS connectors and a Lanes/Graph toggle.

**Architecture:** Extend the existing `CoordinationPipelineView` Lit component with a `viewMode` property. A pure function `assignTiers()` transforms `PhaseItem[]` into a `TierElement[]` model (tiers of nodes + connectors between them). The graph view renders tiers as CSS flex rows with styled connectors — no SVG, no external layout libraries. Auto-detection picks the best default view based on pipeline shape.

**Tech Stack:** TypeScript, Lit 3, Vitest, CSS flexbox

## Global Constraints

- All components must have ARIA attributes (role, aria-label, state attrs)
- Reuse existing color constants from `COLOR_MAP` in coordination-pipeline-view.ts
- Reuse existing `.step-card` and `.barrier-bar` CSS classes where possible
- Follow existing test patterns from coordination-pipeline-view.test.ts
- Follow existing gallery patterns from gallery.html (Rendered/YAML tab panel)

---

## Batch 1: Tier Model + Auto-Detection

### Task 1: Tier assignment model and function

**Files:**
- Create: `packages/agent-viewer/src/coordination-graph-model.ts`
- Test: `packages/agent-viewer/src/coordination-graph-model.test.ts`

**Interfaces:**
- Consumes: `PhaseItem`, `CoordinationStep`, `MatchRouting`, `SwimLanes` from `coordination-types.ts`
- Produces: `GraphNode`, `GraphTier`, `GraphConnector`, `TierElement`, `assignTiers(phases: PhaseItem[]): TierElement[]`, `detectPreferredView(phases: PhaseItem[]): 'lanes' | 'graph'`

- [ ] **Step 1: Write failing tests for assignTiers — linear pipeline**

```typescript
import { describe, it, expect } from 'vitest';
import type { PhaseItem } from './coordination-types.js';
import { assignTiers, detectPreferredView } from './coordination-graph-model.js';
import type { GraphTier, GraphConnector } from './coordination-graph-model.js';

describe('assignTiers', () => {
  it('assigns single step to one tier', () => {
    const phases: PhaseItem[] = [
      { type: 'step', action: 'agent-delegate', target: 'classifier' },
    ];
    const result = assignTiers(phases);
    expect(result).toHaveLength(1);
    expect(result[0].kind).toBe('tier');
    const tier = result[0] as GraphTier;
    expect(tier.nodes).toHaveLength(1);
    expect(tier.nodes[0].step.target).toBe('classifier');
  });

  it('assigns two consecutive steps to separate tiers with flow connector', () => {
    const phases: PhaseItem[] = [
      { type: 'step', action: 'agent-delegate', target: 'step-a' },
      { type: 'step', action: 'agent-delegate', target: 'step-b' },
    ];
    const result = assignTiers(phases);
    expect(result).toHaveLength(3);
    expect(result[0].kind).toBe('tier');
    expect(result[1].kind).toBe('connector');
    expect((result[1] as GraphConnector).type).toBe('flow');
    expect(result[2].kind).toBe('tier');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/agent-viewer/src/coordination-graph-model.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Write failing tests for assignTiers — fan-out/fan-in pipeline**

```typescript
  it('assigns parallel block as multi-node tier with fan-out connector', () => {
    const phases: PhaseItem[] = [
      { type: 'step', action: 'agent-delegate', target: 'classifier' },
      { type: 'signal', name: 'classified', description: 'fan-out' },
      {
        type: 'parallel',
        lanes: {
          'identity': [{ action: 'agent-delegate', target: 'id-extract' }],
          'financial': [{ action: 'agent-delegate', target: 'fin-extract' }],
          'address': [{ action: 'agent-delegate', target: 'addr-extract' }],
        },
      },
      { type: 'quorum', name: 'done', required: 2, of: 3 },
      { type: 'step', action: 'agent-delegate', target: 'risk-assessor' },
    ];
    const result = assignTiers(phases);

    // Tier: classifier
    expect(result[0].kind).toBe('tier');
    expect((result[0] as GraphTier).nodes).toHaveLength(1);

    // Connector: fan-out with lane names as labels
    expect(result[1].kind).toBe('connector');
    const fanOut = result[1] as GraphConnector;
    expect(fanOut.type).toBe('fan-out');
    expect(fanOut.labels).toEqual(['identity', 'financial', 'address']);
    expect(fanOut.barrier).toBeDefined();
    expect(fanOut.barrier!.type).toBe('signal');

    // Tier: 3 parallel extractors
    expect(result[2].kind).toBe('tier');
    expect((result[2] as GraphTier).nodes).toHaveLength(3);

    // Connector: fan-in with quorum
    expect(result[3].kind).toBe('connector');
    const fanIn = result[3] as GraphConnector;
    expect(fanIn.type).toBe('fan-in');
    expect(fanIn.barrier).toBeDefined();
    expect(fanIn.barrier!.type).toBe('quorum');
    expect(fanIn.barrier!.required).toBe(2);
    expect(fanIn.barrier!.of).toBe(3);

    // Tier: risk-assessor
    expect(result[4].kind).toBe('tier');
    expect((result[4] as GraphTier).nodes).toHaveLength(1);
  });
```

- [ ] **Step 4: Write failing tests for assignTiers — match routing outcomes**

```typescript
  it('assigns match routing as decision tier + outcomes tier', () => {
    const phases: PhaseItem[] = [
      {
        type: 'step',
        action: 'agent-delegate',
        target: 'risk-assessor',
        routing: {
          expression: '${result.decision}',
          cases: [
            { pattern: 'approve', step: { action: 'rest', target: '/approve', iconColor: 'green' } },
            { pattern: 'escalate', step: { action: 'agent-delegate', target: 'reviewer', iconColor: 'amber' } },
            { pattern: 'reject', step: { action: 'rest', target: '/reject', iconColor: 'red' } },
          ],
        },
      },
    ];
    const result = assignTiers(phases);

    // Tier: decision node
    expect(result[0].kind).toBe('tier');
    expect((result[0] as GraphTier).nodes[0].step.target).toBe('risk-assessor');
    expect((result[0] as GraphTier).nodes[0].routing).toBeDefined();

    // Connector: outcomes
    expect(result[1].kind).toBe('connector');
    expect((result[1] as GraphConnector).type).toBe('outcomes');

    // Tier: 3 outcome nodes with color hints
    expect(result[2].kind).toBe('tier');
    const outcomes = result[2] as GraphTier;
    expect(outcomes.nodes).toHaveLength(3);
    expect(outcomes.nodes[0].colorHint).toBe('green');
    expect(outcomes.nodes[1].colorHint).toBe('amber');
    expect(outcomes.nodes[2].colorHint).toBe('red');
  });
```

- [ ] **Step 5: Write failing tests for assignTiers — multi-step lanes**

```typescript
  it('uses first step from multi-step lane and adds stepCount', () => {
    const phases: PhaseItem[] = [
      {
        type: 'parallel',
        lanes: {
          'lane-a': [
            { action: 'agent-delegate', target: 'primary' },
            { action: 'rest', target: '/secondary' },
            { action: 'graphql', target: 'third' },
          ],
        },
      },
    ];
    const result = assignTiers(phases);
    const tier = result[0] as GraphTier;
    expect(tier.nodes[0].step.target).toBe('primary');
    expect(tier.nodes[0].extraStepCount).toBe(2);
  });
```

- [ ] **Step 6: Write failing tests for detectPreferredView**

```typescript
describe('detectPreferredView', () => {
  it('returns graph when parallel + quorum present', () => {
    const phases: PhaseItem[] = [
      { type: 'step', action: 'agent-delegate', target: 'a' },
      { type: 'parallel', lanes: { x: [{ action: 'rest', target: 'y' }] } },
      { type: 'quorum', name: 'q', required: 1, of: 1 },
    ];
    expect(detectPreferredView(phases)).toBe('graph');
  });

  it('returns lanes when no parallel', () => {
    const phases: PhaseItem[] = [
      { type: 'step', action: 'agent-delegate', target: 'a' },
      { type: 'step', action: 'rest', target: 'b' },
    ];
    expect(detectPreferredView(phases)).toBe('lanes');
  });

  it('returns lanes when parallel without quorum', () => {
    const phases: PhaseItem[] = [
      { type: 'parallel', lanes: { x: [{ action: 'rest', target: 'y' }] } },
    ];
    expect(detectPreferredView(phases)).toBe('lanes');
  });
});
```

- [ ] **Step 7: Run all tests to verify they fail**

Run: `yarn vitest run packages/agent-viewer/src/coordination-graph-model.test.ts`
Expected: FAIL — module not found

- [ ] **Step 8: Implement types and functions**

Create `coordination-graph-model.ts`:

```typescript
import type { PhaseItem, CoordinationStep, MatchRouting, SwimLanes } from './coordination-types.js';

export interface GraphNode {
  step: CoordinationStep;
  routing?: MatchRouting;
  colorHint?: 'green' | 'amber' | 'red';
  extraStepCount?: number;
}

export interface GraphTier {
  kind: 'tier';
  nodes: GraphNode[];
  labels?: string[];
}

export interface GraphConnector {
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

export type TierElement = GraphTier | GraphConnector;

const COLOR_HINT_MAP: Record<string, 'green' | 'amber' | 'red'> = {
  approve: 'green',
  escalate: 'amber',
  reject: 'red',
};

export function assignTiers(phases: PhaseItem[]): TierElement[] {
  const elements: TierElement[] = [];

  for (let i = 0; i < phases.length; i++) {
    const phase = phases[i];

    // Insert flow connector between consecutive tiers (not after fan-out/fan-in connectors)
    if (elements.length > 0 && elements[elements.length - 1].kind === 'tier' && phase.type !== 'quorum') {
      // Check if next phase is parallel — emit fan-out instead of flow
      if (phase.type === 'parallel') {
        // fan-out handled below
      } else {
        elements.push({ kind: 'connector', type: 'flow' });
      }
    }

    switch (phase.type) {
      case 'step': {
        const node: GraphNode = { step: phase };
        elements.push({ kind: 'tier', nodes: [node] });
        if (phase.routing) {
          node.routing = phase.routing;
          elements.push({ kind: 'connector', type: 'outcomes' });
          const outcomeNodes: GraphNode[] = phase.routing.cases.map(c => ({
            step: c.step,
            colorHint: COLOR_HINT_MAP[c.pattern] ?? (c.step.iconColor as 'green' | 'amber' | 'red' | undefined),
          }));
          elements.push({ kind: 'tier', nodes: outcomeNodes });
        }
        break;
      }
      case 'signal': {
        // Signal before parallel → will be attached to fan-out connector
        const nextPhase = phases[i + 1];
        if (nextPhase?.type === 'parallel') {
          // Handled when parallel is processed — store barrier info
          // The fan-out connector gets this signal's info
        } else {
          // Standalone signal between steps
          elements.push({
            kind: 'connector',
            type: 'flow',
            barrier: { type: 'signal', name: phase.name, description: phase.description },
          });
        }
        break;
      }
      case 'parallel': {
        const laneNames = Object.keys(phase.lanes);
        // Check if previous phase was a signal — attach it to the fan-out
        const prevPhase = phases[i - 1];
        const signalBarrier = prevPhase?.type === 'signal'
          ? { type: 'signal' as const, name: prevPhase.name, description: prevPhase.description }
          : undefined;

        elements.push({
          kind: 'connector',
          type: 'fan-out',
          labels: laneNames,
          barrier: signalBarrier,
        });

        const nodes: GraphNode[] = laneNames.map(name => {
          const steps = phase.lanes[name];
          const node: GraphNode = { step: steps[0] };
          if (steps.length > 1) {
            node.extraStepCount = steps.length - 1;
          }
          return node;
        });
        elements.push({ kind: 'tier', nodes });
        break;
      }
      case 'quorum': {
        elements.push({
          kind: 'connector',
          type: 'fan-in',
          barrier: { type: 'quorum', name: phase.name, required: phase.required, of: phase.of },
        });
        break;
      }
    }
  }

  return elements;
}

export function detectPreferredView(phases: PhaseItem[]): 'lanes' | 'graph' {
  const hasParallel = phases.some(p => p.type === 'parallel');
  const hasQuorum = phases.some(p => p.type === 'quorum');
  return hasParallel && hasQuorum ? 'graph' : 'lanes';
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `yarn vitest run packages/agent-viewer/src/coordination-graph-model.test.ts`
Expected: ALL PASS

- [ ] **Step 10: Export from index.ts**

Add to `packages/agent-viewer/src/index.ts`:

```typescript
export { assignTiers, detectPreferredView } from './coordination-graph-model.js';
export type { GraphNode, GraphTier, GraphConnector, TierElement } from './coordination-graph-model.js';
```

- [ ] **Step 11: Commit**

```bash
git add packages/agent-viewer/src/coordination-graph-model.ts packages/agent-viewer/src/coordination-graph-model.test.ts packages/agent-viewer/src/index.ts
git commit -m "feat(#241): add tier assignment model and auto-detect for graph view"
```

---

## Batch 2: Graph View Rendering + Toggle

### Task 2: Add viewMode property and header toggle

**Files:**
- Modify: `packages/agent-viewer/src/coordination-pipeline-view.ts`
- Test: `packages/agent-viewer/src/coordination-pipeline-view.test.ts`

**Interfaces:**
- Consumes: `assignTiers`, `detectPreferredView`, `TierElement`, `GraphTier`, `GraphConnector` from `coordination-graph-model.ts`
- Produces: `viewMode` property on `CoordinationPipelineView`

- [ ] **Step 1: Write failing tests for viewMode and toggle**

Add to `coordination-pipeline-view.test.ts`:

```typescript
import { assignTiers, detectPreferredView } from './coordination-graph-model.js';

describe('viewMode and toggle', () => {
  let el: CoordinationPipelineView;

  beforeEach(async () => {
    el = document.createElement('coordination-pipeline-view') as CoordinationPipelineView;
    el.pipeline = kycPipeline();
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => el.remove());

  it('auto-detects graph mode for KYC pipeline (parallel + quorum)', () => {
    expect(el.viewMode).toBe('graph');
  });

  it('renders toggle in header', () => {
    const toggle = el.shadowRoot!.querySelector('.view-toggle');
    expect(toggle).not.toBeNull();
    const buttons = toggle!.querySelectorAll('[role="radio"]');
    expect(buttons).toHaveLength(2);
  });

  it('toggle has correct ARIA attributes', () => {
    const group = el.shadowRoot!.querySelector('[role="radiogroup"]');
    expect(group).not.toBeNull();
    const radios = el.shadowRoot!.querySelectorAll('[role="radio"]');
    const active = Array.from(radios).find(r => r.getAttribute('aria-checked') === 'true');
    expect(active?.textContent?.trim()).toBe('Graph');
  });

  it('switches to lanes view on toggle click', async () => {
    const radios = el.shadowRoot!.querySelectorAll('[role="radio"]');
    const lanesBtn = Array.from(radios).find(r => r.textContent?.trim() === 'Lanes');
    (lanesBtn as HTMLElement).click();
    await el.updateComplete;
    expect(el.viewMode).toBe('lanes');
    expect(el.shadowRoot!.querySelector('.swim-lanes')).not.toBeNull();
  });

  it('renders graph view container when in graph mode', () => {
    expect(el.shadowRoot!.querySelector('.graph-view')).not.toBeNull();
  });

  it('does not render swim-lanes in graph mode', () => {
    expect(el.shadowRoot!.querySelector('.swim-lanes')).toBeNull();
  });

  it('defaults to lanes for pipeline without parallel + quorum', async () => {
    el.pipeline = {
      name: 'simple',
      phases: [
        { type: 'step', action: 'agent-delegate', target: 'a' },
        { type: 'step', action: 'rest', target: 'b' },
      ],
    };
    await el.updateComplete;
    expect(el.viewMode).toBe('lanes');
    expect(el.shadowRoot!.querySelector('.graph-view')).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/agent-viewer/src/coordination-pipeline-view.test.ts`
Expected: FAIL — viewMode not defined, .view-toggle not found

- [ ] **Step 3: Add viewMode property, auto-detection, and toggle to the component**

Modify `coordination-pipeline-view.ts`:

1. Add import: `import { assignTiers, detectPreferredView } from './coordination-graph-model.js';` and `import type { TierElement, GraphTier, GraphConnector } from './coordination-graph-model.js';`
2. Add `@property()` for `viewMode: 'lanes' | 'graph'`
3. Override `willUpdate` to auto-detect on pipeline change:
```typescript
override willUpdate(changed: PropertyValues) {
  if (changed.has('pipeline') && this.pipeline) {
    this.viewMode = detectPreferredView(this.pipeline.phases);
  }
}
```
4. Add toggle rendering in `renderHeader()`:
```typescript
html`
  <div class="view-toggle" role="radiogroup" aria-label="View mode">
    <span role="radio" aria-checked="${this.viewMode === 'lanes'}"
          class="toggle-btn ${this.viewMode === 'lanes' ? 'active' : ''}"
          @click="${() => { this.viewMode = 'lanes'; }}"
          tabindex="${this.viewMode === 'lanes' ? '0' : '-1'}"
          @keydown="${this.handleToggleKey}">Lanes</span>
    <span role="radio" aria-checked="${this.viewMode === 'graph'}"
          class="toggle-btn ${this.viewMode === 'graph' ? 'active' : ''}"
          @click="${() => { this.viewMode = 'graph'; }}"
          tabindex="${this.viewMode === 'graph' ? '0' : '-1'}"
          @keydown="${this.handleToggleKey}">Graph</span>
  </div>
`
```
5. Add `handleToggleKey` for arrow key navigation between radio buttons.
6. Change `render()` to dispatch on `viewMode`:
```typescript
override render() {
  if (!this.pipeline) return nothing;
  const p = this.pipeline;
  return html`
    ${this.renderHeader(p)}
    ${this.viewMode === 'graph' ? this.renderGraphView(p) : this.renderLanesView(p)}
  `;
}
```
7. Extract existing phase rendering into `renderLanesView(p)`.
8. Add stub `renderGraphView(p)` returning `html`<div class="graph-view"></div>`` (filled in next step).

- [ ] **Step 4: Add CSS for toggle**

```css
.view-toggle {
  display: flex;
  gap: 0;
  margin-left: auto;
  margin-right: 8px;
}
.toggle-btn {
  padding: 3px 10px;
  font-size: 11px;
  font-weight: 500;
  border-radius: 4px;
  cursor: pointer;
  color: #585b70;
  background: rgba(255, 255, 255, 0.04);
  user-select: none;
}
.toggle-btn.active {
  color: #89b4fa;
  background: rgba(137, 180, 250, 0.12);
  font-weight: 600;
}
.toggle-btn:hover:not(.active) {
  color: #7f849c;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn vitest run packages/agent-viewer/src/coordination-pipeline-view.test.ts`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add packages/agent-viewer/src/coordination-pipeline-view.ts packages/agent-viewer/src/coordination-pipeline-view.test.ts
git commit -m "feat(#241): add viewMode property and Lanes/Graph toggle"
```

### Task 3: Render graph tiers and connectors

**Files:**
- Modify: `packages/agent-viewer/src/coordination-pipeline-view.ts`
- Test: `packages/agent-viewer/src/coordination-pipeline-view.test.ts`

**Interfaces:**
- Consumes: `assignTiers`, `TierElement`, `GraphTier`, `GraphConnector` from `coordination-graph-model.ts`
- Produces: Full graph view rendering in the component

- [ ] **Step 1: Write failing tests for graph tier rendering**

Add to `coordination-pipeline-view.test.ts`:

```typescript
describe('graph view rendering', () => {
  let el: CoordinationPipelineView;

  beforeEach(async () => {
    el = document.createElement('coordination-pipeline-view') as CoordinationPipelineView;
    el.pipeline = kycPipeline();
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => el.remove());

  it('renders graph tiers as flex rows', () => {
    const tiers = el.shadowRoot!.querySelectorAll('.graph-tier');
    expect(tiers.length).toBeGreaterThanOrEqual(3);
  });

  it('renders classifier as single-node tier', () => {
    const tiers = el.shadowRoot!.querySelectorAll('.graph-tier');
    const firstTier = tiers[0];
    const cards = firstTier.querySelectorAll('.step-card');
    expect(cards).toHaveLength(1);
    expect(cards[0].querySelector('.step-target')?.textContent).toBe('doc-classifier');
  });

  it('renders 3 parallel extractors in one tier', () => {
    const tiers = el.shadowRoot!.querySelectorAll('.graph-tier');
    const parallelTier = Array.from(tiers).find(t => t.querySelectorAll('.step-card').length === 3);
    expect(parallelTier).not.toBeNull();
  });

  it('renders fan-out connector with lane name labels', () => {
    const fanOut = el.shadowRoot!.querySelector('.graph-connector.fan-out');
    expect(fanOut).not.toBeNull();
    const labels = fanOut!.querySelectorAll('.edge-label');
    expect(labels).toHaveLength(3);
    expect(labels[0].textContent?.trim()).toBe('identity-extractor');
    expect(labels[1].textContent?.trim()).toBe('financial-extractor');
    expect(labels[2].textContent?.trim()).toBe('address-verifier');
  });

  it('renders fan-in connector with quorum barrier', () => {
    const fanIn = el.shadowRoot!.querySelector('.graph-connector.fan-in');
    expect(fanIn).not.toBeNull();
    const quorum = fanIn!.querySelector('.graph-quorum');
    expect(quorum).not.toBeNull();
    expect(quorum!.textContent).toContain('QUORUM');
    expect(quorum!.textContent).toContain('2 of 3');
  });

  it('renders outcome nodes with color-coded borders', () => {
    const tiers = el.shadowRoot!.querySelectorAll('.graph-tier');
    const lastTier = tiers[tiers.length - 1];
    const outcomes = lastTier.querySelectorAll('.outcome-node');
    expect(outcomes).toHaveLength(3);
    expect(outcomes[0].classList.contains('green')).toBe(true);
    expect(outcomes[1].classList.contains('amber')).toBe(true);
    expect(outcomes[2].classList.contains('red')).toBe(true);
  });

  it('renders decision node with amber emphasis border', () => {
    const decision = el.shadowRoot!.querySelector('.decision-node');
    expect(decision).not.toBeNull();
  });

  it('renders extra step count badge for multi-step lanes', () => {
    const badges = el.shadowRoot!.querySelectorAll('.extra-steps');
    expect(badges.length).toBeGreaterThanOrEqual(3);
    expect(badges[0].textContent?.trim()).toBe('+1 more');
  });

  it('graph view has correct ARIA', () => {
    const graphView = el.shadowRoot!.querySelector('.graph-view');
    expect(graphView?.getAttribute('role')).toBe('img');
    expect(graphView?.getAttribute('aria-label')).toContain('kyc-verification');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/agent-viewer/src/coordination-pipeline-view.test.ts`
Expected: FAIL — .graph-tier not found, .graph-connector not found

- [ ] **Step 3: Implement renderGraphView**

In `coordination-pipeline-view.ts`, implement `renderGraphView`:

```typescript
private renderGraphView(p: CoordinationPipeline): TemplateResult {
  const elements = assignTiers(p.phases);
  return html`
    <div class="graph-view" role="img" aria-label="Pipeline topology: ${p.name}">
      ${elements.map(el => el.kind === 'tier'
        ? this.renderGraphTier(el)
        : this.renderGraphConnector(el)
      )}
    </div>
  `;
}

private renderGraphTier(tier: GraphTier): TemplateResult {
  return html`
    <div class="graph-tier">
      ${tier.nodes.map(node => this.renderGraphNode(node))}
    </div>
  `;
}

private renderGraphNode(node: GraphNode): TemplateResult {
  const colorClass = node.colorHint ? ` outcome-node ${node.colorHint}` : '';
  const decisionClass = node.routing ? ' decision-node' : '';
  return html`
    <div class="graph-node-wrapper${colorClass}${decisionClass}">
      ${this.renderStepCard(node.step)}
      ${node.extraStepCount
        ? html`<span class="extra-steps">+${node.extraStepCount} more</span>`
        : nothing}
    </div>
  `;
}

private renderGraphConnector(connector: GraphConnector): TemplateResult {
  return html`
    <div class="graph-connector ${connector.type}">
      ${connector.type === 'flow' && connector.barrier
        ? this.renderGraphBarrier(connector.barrier)
        : connector.type === 'fan-out'
          ? html`
              <span class="connector-chevron">▼</span>
              ${connector.barrier ? this.renderGraphBarrier(connector.barrier) : nothing}
              <div class="fan-out-labels">
                ${connector.labels?.map(l => html`<span class="edge-label">${l}</span>`)}
              </div>
            `
          : connector.type === 'fan-in'
            ? html`
                ${connector.barrier ? this.renderGraphBarrier(connector.barrier) : nothing}
                <span class="connector-chevron">▼</span>
              `
            : html`<span class="connector-chevron">▼</span>`
      }
    </div>
  `;
}

private renderGraphBarrier(barrier: GraphConnector['barrier']): TemplateResult {
  if (!barrier) return html``;
  if (barrier.type === 'quorum') {
    return html`
      <div class="graph-quorum">
        <span class="quorum-label">QUORUM</span>
        <span class="quorum-detail">${barrier.required} of ${barrier.of} required</span>
      </div>
    `;
  }
  return html`
    <div class="graph-signal">
      <span class="signal-label">◇ ${barrier.name}</span>
      ${barrier.description ? html`<span class="signal-desc">${barrier.description}</span>` : nothing}
    </div>
  `;
}
```

- [ ] **Step 4: Add CSS for graph view**

```css
/* ─── Graph View ─── */
.graph-view {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0;
  padding: 24px 32px;
}
.graph-tier {
  display: flex;
  justify-content: center;
  gap: 16px;
  padding: 8px 0;
  flex-wrap: wrap;
}
.graph-node-wrapper {
  position: relative;
}
.graph-node-wrapper.decision-node .step-card {
  border: 2px solid #f9e2af;
}
.graph-node-wrapper.outcome-node {
  border-radius: 6px;
}
.graph-node-wrapper.outcome-node.green .step-card {
  border-left: 3px solid #a6e3a1;
  background: rgba(166, 227, 161, 0.08);
}
.graph-node-wrapper.outcome-node.amber .step-card {
  border-left: 3px solid #f9e2af;
  background: rgba(249, 226, 175, 0.08);
}
.graph-node-wrapper.outcome-node.red .step-card {
  border-left: 3px solid #f38ba8;
  background: rgba(243, 139, 168, 0.08);
}
.extra-steps {
  display: block;
  text-align: center;
  font-size: 10px;
  color: #585b70;
  margin-top: 4px;
}

/* ─── Connectors ─── */
.graph-connector {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 4px 0;
  gap: 4px;
}
.connector-chevron {
  font-size: 10px;
  color: #585b70;
}
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
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
}
.graph-quorum {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 6px 16px;
  background: rgba(137, 180, 250, 0.06);
  border: 1px dashed #89b4fa;
  border-radius: 4px;
}
.quorum-label {
  font-size: 11px;
  font-weight: 600;
  color: #89b4fa;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}
.quorum-detail {
  font-size: 11px;
  color: #585b70;
}
.graph-signal {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 4px 12px;
}
.signal-label {
  font-size: 11px;
  font-weight: 600;
  color: #cba6f7;
}
.signal-desc {
  font-size: 11px;
  color: #585b70;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn vitest run packages/agent-viewer/src/coordination-pipeline-view.test.ts`
Expected: ALL PASS

- [ ] **Step 6: Run full typecheck and existing tests**

Run: `yarn typecheck && yarn vitest run packages/agent-viewer/`
Expected: ALL PASS — no regressions in existing lanes tests

- [ ] **Step 7: Commit**

```bash
git add packages/agent-viewer/src/coordination-pipeline-view.ts packages/agent-viewer/src/coordination-pipeline-view.test.ts
git commit -m "feat(#241): render graph tiers, connectors, and outcome nodes"
```

---

## Batch 3: Gallery + Schema

### Task 4: Update gallery and schema registry

**Files:**
- Modify: `packages/agent-viewer/gallery.html`
- Modify: `packages/blocks-ui-schema/src/registry.ts` (if CoordinationPipelineView has schema entry — update for `viewMode`)

**Interfaces:**
- Consumes: `CoordinationPipelineView` with `viewMode` property

- [ ] **Step 1: Update gallery section 5 to show both views**

In `gallery.html`, update section 5 heading:

```html
<h2>5. Coordination Pipeline — Swim Lane + Graph View</h2>
<h3>KYC verification pipeline with Lanes/Graph toggle (auto-detects graph for fan-out/fan-in)</h3>
```

No code changes needed in the `renderComponentSection` call — the component auto-detects graph mode for the KYC pipeline. The toggle in the rendered component lets gallery visitors switch between views.

- [ ] **Step 2: Run gallery manually to verify**

Run: `yarn vite packages/agent-viewer/` and open in browser.
Verify:
- Section 5 shows graph view by default (KYC pipeline has parallel + quorum)
- Lanes/Graph toggle is visible in header
- Clicking "Lanes" switches to the familiar swim lane view
- Clicking "Graph" switches back
- Graph shows: classifier tier → fan-out with lane labels → 3 extractors → quorum barrier → risk-assessor (amber border) → outcomes (green/amber/red)

- [ ] **Step 3: Run schema staleness test**

Run: `yarn vitest run packages/blocks-ui-schema/`
If test fails due to new `viewMode` property, regenerate schemas:
Run: `yarn generate-schemas` (or the project's schema generation command)

- [ ] **Step 4: Run full test suite**

Run: `yarn test`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/agent-viewer/gallery.html
git commit -m "feat(#241): update gallery for graph view and verify schema"
```

## References

- [2026-10-19-agent-pipeline-graph-view-design.md] — design spec this plan implements
- [packages/agent-viewer/src/coordination-pipeline-view.ts] — existing component
- [packages/agent-viewer/src/coordination-types.ts] — CoordinationPipeline, PhaseItem types
- [packages/agent-viewer/src/coordination-pipeline-view.test.ts] — existing test patterns
- [packages/agent-viewer/gallery.html] — existing gallery patterns
- [packages/agent-viewer/src/index.ts] — public exports
- [casehub-pages/docs/mockups/step-review-viewer.html section 14] — graph view mockup
- [GitHub #241] — focal issue
- [GitHub #239] — parent epic
