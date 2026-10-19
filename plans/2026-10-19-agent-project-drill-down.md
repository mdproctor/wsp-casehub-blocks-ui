# Agent Project Drill-Down Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #242 — Agent Project Drill-Down
**Issue group:** #242 (part of epic #239)

**Goal:** Wire drill-down from coordination-pipeline-view to agent-viewer, add a Steps tab, and make breadcrumb segments emit navigation events.

**Architecture:** Three components modified: coordination-pipeline-view emits `coordination:drill-down` CustomEvent on drillable card click; agent-viewer gains a Steps tab (renders CoordinationStep[] as step cards) and breadcrumb click events; gallery adds a drill-down demo section.

**Tech Stack:** TypeScript, Lit 3, Vitest

## Global Constraints

- All components must have ARIA attributes (role, aria-label, state attrs)
- Reuse existing step-card visual patterns from coordination-pipeline-view
- Follow existing test patterns from agent-viewer.test.ts
- CustomEvent pattern consistent with other blocks-ui events (e.g., `catalog:template:selected`)

---

## Batch 1: Drill-Down Event + Breadcrumb Navigation

### Task 1: Emit drill-down event from coordination-pipeline-view

**Files:**
- Modify: `packages/agent-viewer/src/coordination-pipeline-view.ts`
- Modify: `packages/agent-viewer/src/coordination-graph-model.ts`
- Modify: `packages/agent-viewer/src/coordination-types.ts`
- Test: `packages/agent-viewer/src/coordination-pipeline-view.test.ts`

**Interfaces:**
- Consumes: `CoordinationStep`, `SwimLanes`, `PhaseItem` from coordination-types.ts; `GraphNode` from coordination-graph-model.ts
- Produces: `DrillDownDetail` interface, `coordination:drill-down` CustomEvent

- [ ] **Step 1: Add DrillDownDetail type to coordination-types.ts**

Add at the end of coordination-types.ts:

```typescript
export interface DrillDownDetail {
  agentName: string;
  steps: CoordinationStep[];
}
```

- [ ] **Step 2: Add laneName and laneSteps to GraphNode**

In coordination-graph-model.ts, add two optional fields to `GraphNode`:

```typescript
export interface GraphNode {
  step: CoordinationStep;
  routing?: MatchRouting;
  colorHint?: 'green' | 'amber' | 'red';
  extraStepCount?: number;
  laneName?: string;
  laneSteps?: CoordinationStep[];
}
```

In `assignTiers()`, inside the `case 'parallel'` block where nodes are created, set these fields:

```typescript
const nodes: GraphNode[] = laneNames.map(name => {
  const steps = phase.lanes[name];
  const node: GraphNode = {
    step: steps[0],
    laneName: name,
    laneSteps: steps,
  };
  if (steps.length > 1) {
    node.extraStepCount = steps.length - 1;
  }
  return node;
});
```

- [ ] **Step 3: Write failing tests for drill-down event**

Add to coordination-pipeline-view.test.ts:

```typescript
describe('drill-down events', () => {
  let el: CoordinationPipelineView;

  beforeEach(async () => {
    el = document.createElement('coordination-pipeline-view') as CoordinationPipelineView;
    el.pipeline = kycPipeline();
    el.viewMode = 'lanes';
    document.body.appendChild(el);
    await el.updateComplete;
    el.viewMode = 'lanes';
    await el.updateComplete;
  });

  afterEach(() => el.remove());

  it('emits coordination:drill-down on drillable card click', async () => {
    let detail: any = null;
    el.addEventListener('coordination:drill-down', ((e: CustomEvent) => {
      detail = e.detail;
    }) as EventListener);
    const drillable = el.shadowRoot!.querySelector('.step-card.drillable') as HTMLElement;
    drillable.click();
    expect(detail).not.toBeNull();
    expect(detail.agentName).toBeDefined();
    expect(detail.steps).toBeDefined();
    expect(Array.isArray(detail.steps)).toBe(true);
  });

  it('does not emit on non-drillable card click', () => {
    let emitted = false;
    el.addEventListener('coordination:drill-down', () => { emitted = true; });
    const cards = el.shadowRoot!.querySelectorAll('.step-card:not(.drillable)');
    if (cards.length > 0) {
      (cards[0] as HTMLElement).click();
    }
    expect(emitted).toBe(false);
  });

  it('emits drill-down with correct lane steps', async () => {
    let detail: any = null;
    el.addEventListener('coordination:drill-down', ((e: CustomEvent) => {
      detail = e.detail;
    }) as EventListener);
    const drillables = el.shadowRoot!.querySelectorAll('.step-card.drillable');
    (drillables[0] as HTMLElement).click();
    expect(detail.steps.length).toBe(2);
  });

  it('emits drill-down from graph view drillable nodes', async () => {
    el.viewMode = 'graph';
    await el.updateComplete;
    let detail: any = null;
    el.addEventListener('coordination:drill-down', ((e: CustomEvent) => {
      detail = e.detail;
    }) as EventListener);
    const drillable = el.shadowRoot!.querySelector('.step-card.drillable') as HTMLElement;
    if (drillable) {
      drillable.click();
      expect(detail).not.toBeNull();
    }
  });

  it('drillable cards have role button and keyboard activation', async () => {
    const drillable = el.shadowRoot!.querySelector('.step-card.drillable') as HTMLElement;
    expect(drillable.getAttribute('role')).toBe('button');
    let detail: any = null;
    el.addEventListener('coordination:drill-down', ((e: CustomEvent) => {
      detail = e.detail;
    }) as EventListener);
    drillable.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', bubbles: true }));
    expect(detail).not.toBeNull();
  });
});
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `yarn workspace @casehubio/agent-viewer test`
Expected: FAIL — no click handler, no event emitted

- [ ] **Step 5: Implement drill-down click handler**

In coordination-pipeline-view.ts, modify `renderStepCard()` to accept an optional drill-down context and add click/keyboard handlers:

The `renderStepCard` method currently takes `step: CoordinationStep`. It needs to also accept drill-down context (lane name and lane steps) for when the card is inside a lane or graph node. Add a private method `_handleDrillDown` and modify `renderStepCard`:

```typescript
private _emitDrillDown(agentName: string, steps: CoordinationStep[]) {
  this.dispatchEvent(new CustomEvent('coordination:drill-down', {
    bubbles: true,
    composed: true,
    detail: { agentName, steps } as DrillDownDetail,
  }));
}
```

For the step card rendering, add `role="button"`, `tabindex="0"`, click handler, and keydown handler (Enter/Space) when `step.drillDown` is true. The handler needs the lane context — pass it via closure from `renderLane()` and `_renderGraphNode()`.

Update `renderStepCard` signature to:
```typescript
private renderStepCard(step: CoordinationStep, drillContext?: { name: string; steps: CoordinationStep[] }): TemplateResult
```

Add to the `.step-card` div when `step.drillDown && drillContext`:
```typescript
role="${step.drillDown ? 'button' : nothing}"
tabindex="${step.drillDown ? '0' : nothing}"
@click="${step.drillDown && drillContext ? () => this._emitDrillDown(drillContext.name, drillContext.steps) : nothing}"
@keydown="${step.drillDown && drillContext ? (e: KeyboardEvent) => {
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault();
    this._emitDrillDown(drillContext.name, drillContext.steps);
  }
} : nothing}"
```

Update call sites:
- In `renderLane(name, steps)`: pass `{ name, steps }` as drillContext
- In `_renderGraphNode(node)`: pass `{ name: node.laneName ?? node.step.target ?? '', steps: node.laneSteps ?? [node.step] }` as drillContext
- In `renderPhase` for standalone steps: pass `{ name: step.target ?? step.action, steps: [step] }` as drillContext

- [ ] **Step 6: Export DrillDownDetail from index.ts**

Add to index.ts exports:
```typescript
export type { DrillDownDetail } from './coordination-types.js';
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn workspace @casehubio/agent-viewer test`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add packages/agent-viewer/src/
git commit -m "feat(#242): emit coordination:drill-down event on drillable card click"
```

### Task 2: Breadcrumb back-navigation events

**Files:**
- Modify: `packages/agent-viewer/src/agent-viewer.ts`
- Test: `packages/agent-viewer/src/agent-viewer.test.ts`

**Interfaces:**
- Consumes: `AgentViewerData` (existing)
- Produces: `BreadcrumbBackDetail` interface, `viewer:breadcrumb-back` CustomEvent

- [ ] **Step 1: Write failing tests for breadcrumb events**

Add to agent-viewer.test.ts:

```typescript
describe('breadcrumb navigation', () => {
  let el: AgentViewer;

  beforeEach(async () => {
    el = document.createElement('agent-viewer') as AgentViewer;
    el.data = { ...fixture(), breadcrumb: ['KYC Pipeline', 'identity-extractor'] };
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => el.remove());

  it('emits viewer:breadcrumb-back on segment click', () => {
    let detail: any = null;
    el.addEventListener('viewer:breadcrumb-back', ((e: CustomEvent) => {
      detail = e.detail;
    }) as EventListener);
    const segment = el.shadowRoot!.querySelector('.breadcrumb-segment') as HTMLElement;
    segment.click();
    expect(detail).not.toBeNull();
    expect(detail.index).toBe(0);
    expect(detail.segment).toBe('KYC Pipeline');
  });

  it('does not emit on current (last) breadcrumb segment', () => {
    let emitted = false;
    el.addEventListener('viewer:breadcrumb-back', () => { emitted = true; });
    const current = el.shadowRoot!.querySelector('.breadcrumb-current') as HTMLElement;
    current.click();
    expect(emitted).toBe(false);
  });

  it('breadcrumb has navigation ARIA', () => {
    const nav = el.shadowRoot!.querySelector('[role="navigation"]');
    expect(nav).not.toBeNull();
    expect(nav!.getAttribute('aria-label')).toBe('Breadcrumb');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/agent-viewer test`
Expected: FAIL — no event emitted, no navigation role

- [ ] **Step 3: Implement breadcrumb click handler and ARIA**

In agent-viewer.ts, update `renderBreadcrumb()`:

```typescript
private renderBreadcrumb(): TemplateResult | typeof nothing {
  const bc = this.data.breadcrumb;
  if (!bc?.length) return nothing;
  const segments = bc.slice(0, -1);
  const current = bc[bc.length - 1];
  return html`
    <nav class="breadcrumb" role="navigation" aria-label="Breadcrumb">
      ${segments.map((s, i) => html`
        <span class="breadcrumb-segment" role="link" tabindex="0"
              @click="${() => this._emitBreadcrumbBack(i, s)}"
              @keydown="${(e: KeyboardEvent) => {
                if (e.key === 'Enter' || e.key === ' ') {
                  e.preventDefault();
                  this._emitBreadcrumbBack(i, s);
                }
              }}">${s}</span>
        <span class="breadcrumb-sep" aria-hidden="true">›</span>
      `)}
      <span class="breadcrumb-current" aria-current="page">${current}</span>
    </nav>
  `;
}

private _emitBreadcrumbBack(index: number, segment: string) {
  this.dispatchEvent(new CustomEvent('viewer:breadcrumb-back', {
    bubbles: true,
    composed: true,
    detail: { index, segment },
  }));
}
```

Export the detail type from agent-viewer.ts:
```typescript
export interface BreadcrumbBackDetail {
  index: number;
  segment: string;
}
```

Add to index.ts:
```typescript
export type { AgentTab, AgentViewerData, BreadcrumbBackDetail } from './agent-viewer.js';
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/agent-viewer test`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add packages/agent-viewer/src/
git commit -m "feat(#242): add breadcrumb back-navigation events with ARIA"
```

---

## Batch 2: Steps Tab + Gallery

### Task 3: Add Steps tab to agent-viewer

**Files:**
- Modify: `packages/agent-viewer/src/agent-viewer.ts`
- Test: `packages/agent-viewer/src/agent-viewer.test.ts`

**Interfaces:**
- Consumes: `CoordinationStep` from coordination-types.ts
- Produces: Extended `AgentViewerData` with `steps?: CoordinationStep[]`, `AgentTab` with `'steps'`

- [ ] **Step 1: Write failing tests for Steps tab**

Add to agent-viewer.test.ts:

```typescript
import type { CoordinationStep } from './coordination-types.js';

function stepsFixture(): CoordinationStep[] {
  return [
    { action: 'agent-delegate', model: 'claude-opus-5', outputFormat: 'structured', drillDown: true,
      params: { descriptor: 'kyc-identity', docs: '${from.identity_docs}' } },
    { action: 'graphql', target: 'sanctions / screenEntity' },
  ];
}

describe('steps tab', () => {
  let el: AgentViewer;

  beforeEach(async () => {
    el = document.createElement('agent-viewer') as AgentViewer;
    el.data = { ...fixture(), steps: stepsFixture() };
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => el.remove());

  it('shows Steps tab when steps data provided', () => {
    const tabs = el.shadowRoot!.querySelectorAll('.tab');
    expect(tabs.length).toBe(3);
    expect(tabs[2]?.textContent).toBe('Steps');
  });

  it('hides Steps tab when no steps data', async () => {
    el.data = { ...fixture(), steps: undefined };
    await el.updateComplete;
    const tabs = el.shadowRoot!.querySelectorAll('.tab');
    expect(tabs.length).toBe(2);
  });

  it('renders step cards in Steps tab', async () => {
    const tabs = el.shadowRoot!.querySelectorAll('.tab');
    (tabs[2] as HTMLElement).click();
    await el.updateComplete;
    const cards = el.shadowRoot!.querySelectorAll('.step-card');
    expect(cards.length).toBe(2);
  });

  it('renders action and target on step cards', async () => {
    const tabs = el.shadowRoot!.querySelectorAll('.tab');
    (tabs[2] as HTMLElement).click();
    await el.updateComplete;
    const actions = el.shadowRoot!.querySelectorAll('.step-action');
    expect(actions[0]?.textContent).toBe('agent-delegate');
    expect(actions[1]?.textContent).toBe('graphql');
  });

  it('renders model badge on step cards', async () => {
    const tabs = el.shadowRoot!.querySelectorAll('.tab');
    (tabs[2] as HTMLElement).click();
    await el.updateComplete;
    const badge = el.shadowRoot!.querySelector('.badge-model');
    expect(badge?.textContent).toBe('claude-opus-5');
  });

  it('Steps tab has tabpanel ARIA', async () => {
    const tabs = el.shadowRoot!.querySelectorAll('.tab');
    (tabs[2] as HTMLElement).click();
    await el.updateComplete;
    const panel = el.shadowRoot!.querySelector('.tab-content');
    expect(panel?.getAttribute('role')).toBe('tabpanel');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/agent-viewer test`
Expected: FAIL — only 2 tabs, no Steps tab

- [ ] **Step 3: Extend AgentTab and AgentViewerData**

In agent-viewer.ts:

```typescript
export type AgentTab = 'eidos' | 'pattern' | 'steps';

export interface AgentViewerData {
  agent: AgentDescriptor;
  pattern?: PatternSpec;
  breadcrumb?: string[];
  steps?: CoordinationStep[];
}
```

Add import:
```typescript
import type { CoordinationStep, CoordinationStepParams } from './coordination-types.js';
import { parseExpression } from './coordination-types.js';
```

- [ ] **Step 4: Update renderTabBar to include Steps tab**

```typescript
private renderTabBar(): TemplateResult {
  const hasPattern = !!this.data.pattern;
  const hasSteps = !!this.data.steps?.length;
  return html`
    <div class="tab-bar" role="tablist">
      <div class="tab ${this.activeTab === 'eidos' ? 'active' : ''}" role="tab"
           aria-selected="${this.activeTab === 'eidos'}"
           @click=${() => this.selectTab('eidos')}>Eidos</div>
      <div class="tab ${this.activeTab === 'pattern' ? 'active' : ''} ${hasPattern ? '' : 'disabled'}" role="tab"
           aria-selected="${this.activeTab === 'pattern'}"
           @click=${() => hasPattern && this.selectTab('pattern')}>Pattern</div>
      ${hasSteps ? html`
        <div class="tab ${this.activeTab === 'steps' ? 'active' : ''}" role="tab"
             aria-selected="${this.activeTab === 'steps'}"
             @click=${() => this.selectTab('steps')}>Steps</div>
      ` : nothing}
    </div>
  `;
}
```

- [ ] **Step 5: Add renderActiveTab case for steps and step card rendering**

Update `renderActiveTab`:
```typescript
private renderActiveTab(): TemplateResult {
  switch (this.activeTab) {
    case 'eidos':
      return html`<eidos-view .agent=${this.data.agent}></eidos-view>`;
    case 'pattern':
      return this.renderPattern();
    case 'steps':
      return this.renderSteps();
  }
}
```

Add `renderSteps()` and step card rendering methods. Add `role="tabpanel"` to the `.tab-content` div in `render()`:

```typescript
private renderSteps(): TemplateResult {
  const steps = this.data.steps;
  if (!steps?.length) return html`<div class="empty-state">No steps data available</div>`;
  return html`
    <div class="steps-list">
      ${steps.map(step => this._renderStepCard(step))}
    </div>
  `;
}

private _renderStepCard(step: CoordinationStep): TemplateResult {
  const ACTION_ICONS: Record<string, { icon: string; cssClass: string }> = {
    'agent-delegate': { icon: '⬡', cssClass: 'agent' },
    'rest': { icon: 'R', cssClass: 'rest' },
    'graphql': { icon: 'G', cssClass: 'graphql' },
    'mcp-call': { icon: 'M', cssClass: 'mcp' },
  };
  const actionDef = ACTION_ICONS[step.action] ?? { icon: '•', cssClass: 'default' };
  let iconClass = actionDef.cssClass;
  let iconChar = step.icon ?? actionDef.icon;
  if (step.iconColor) {
    const colorMap: Record<string, string> = { green: 'approve', red: 'reject', amber: 'warn' };
    iconClass = colorMap[step.iconColor] ?? iconClass;
  }
  return html`
    <div class="step-card">
      <div class="step-icon ${iconClass}">${iconChar}</div>
      <span class="step-action">${step.action}</span>
      ${step.target ? html`<span class="step-target">${step.target}</span>` : nothing}
      <span class="step-badges">
        ${step.model ? html`<span class="badge-model">${step.model}</span>` : nothing}
        ${step.outputFormat ? html`<span class="badge-format">${step.outputFormat}</span>` : nothing}
        ${step.channel ? html`<span class="badge-channel">→ ${step.channel}</span>` : nothing}
      </span>
    </div>
  `;
}
```

- [ ] **Step 6: Add step card CSS to agent-viewer styles**

Add the step-card styles (duplicated from coordination-pipeline-view since shadow DOM isolation prevents sharing):

```css
/* ─── Step Cards (Steps tab) ─── */
.steps-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.step-card {
  display: flex;
  align-items: center;
  gap: 8px;
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 6px;
  padding: 8px 12px;
  font-size: 13px;
}
.step-icon {
  width: 22px; height: 22px; border-radius: 4px;
  display: flex; align-items: center; justify-content: center;
  font-size: 11px; flex-shrink: 0; font-weight: 600;
}
.step-icon.agent   { background: rgba(250, 179, 135, 0.15); color: #fab387; }
.step-icon.rest    { background: rgba(137, 180, 250, 0.15); color: #89b4fa; }
.step-icon.graphql { background: rgba(245, 194, 231, 0.15); color: #f5c2e7; }
.step-icon.mcp     { background: rgba(203, 166, 247, 0.15); color: #cba6f7; }
.step-icon.approve { background: rgba(166, 227, 161, 0.15); color: #a6e3a1; }
.step-icon.reject  { background: rgba(243, 139, 168, 0.15); color: #f38ba8; }
.step-icon.warn    { background: rgba(249, 226, 175, 0.15); color: #f9e2af; }
.step-action { font-weight: 500; color: #cdd6f4; }
.step-target { color: #7f849c; font-size: 12px; }
.step-badges { margin-left: auto; display: flex; gap: 6px; align-items: center; }
.badge-model {
  font-size: 9px; padding: 1px 6px; border-radius: 3px;
  background: rgba(137, 180, 250, 0.2); color: #89b4fa;
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
}
.badge-format {
  font-size: 9px; padding: 1px 6px; border-radius: 3px;
  background: rgba(166, 227, 161, 0.2); color: #a6e3a1;
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
}
.badge-channel {
  font-size: 9px; padding: 1px 6px; border-radius: 3px;
  background: rgba(203, 166, 247, 0.2); color: #cba6f7;
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn workspace @casehubio/agent-viewer test`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add packages/agent-viewer/src/
git commit -m "feat(#242): add Steps tab to agent-viewer with step card rendering"
```

### Task 4: Gallery drill-down section and schema

**Files:**
- Modify: `packages/agent-viewer/gallery.html`
- Modify: `packages/agent-viewer/src/index.ts` (if not already updated in Task 1/2)

**Interfaces:**
- Consumes: `AgentViewerData` with steps, `CoordinationStep` from coordination-types.ts

- [ ] **Step 1: Add section 6 to gallery.html**

After the coordination section, add:

```html
<!-- ═══════════════════════════════════════════════════════ -->
<h2>6. Agent Drill-Down — Project View</h2>
<h3>drill-down from pipeline with breadcrumb, eidos, pattern and steps tabs</h3>
<div id="drilldown-section"></div>
```

In the script section, add:

```javascript
// ═══════════════════════════════════════════════════════
// 6. Agent Drill-Down — Project View
// ═══════════════════════════════════════════════════════
const drilldownContainer = document.getElementById('drilldown-section');
const drilldownPanel = document.createElement('div');
drilldownPanel.className = 'tab-panel';
const ddId = panelId++;
const drilldownSteps = [
  { action: 'agent-delegate', model: 'claude-opus-5', outputFormat: 'structured', drillDown: true,
    params: { descriptor: 'kyc-identity', docs: '${from.identity_docs}' } },
  { action: 'graphql', target: 'sanctions / screenEntity' },
];
drilldownPanel.innerHTML = `
  <div class="tab-bar">
    <button class="tab-btn active" data-tab="rendered-${ddId}">Rendered</button>
    <button class="tab-btn" data-tab="yaml-${ddId}">Data</button>
  </div>
  <div class="tab-content active" id="rendered-${ddId}"></div>
  <div class="tab-content yaml-tab" id="yaml-${ddId}">
    <pre>${highlightYaml(JSON.stringify({ breadcrumb: ['KYC Pipeline', 'identity-extractor'], steps: drilldownSteps }, null, 2))}</pre>
  </div>
`;
drilldownContainer.appendChild(drilldownPanel);
drilldownPanel.querySelectorAll('.tab-btn').forEach(btn => {
  btn.addEventListener('click', () => {
    drilldownPanel.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
    drilldownPanel.querySelectorAll('.tab-content').forEach(c => c.classList.remove('active'));
    btn.classList.add('active');
    document.getElementById(btn.dataset.tab).classList.add('active');
  });
});
const drilldownViewer = document.createElement('agent-viewer');
drilldownViewer.data = {
  agent: descriptor,
  pattern: pattern,
  breadcrumb: ['KYC Pipeline', 'identity-extractor'],
  steps: drilldownSteps,
};
document.getElementById(`rendered-${ddId}`).appendChild(drilldownViewer);
```

- [ ] **Step 2: Run schema staleness test**

Run: `yarn workspace @casehubio/blocks-ui-schema test`
If stale, regenerate: `yarn workspace @casehubio/blocks-ui-schema generate`

- [ ] **Step 3: Run full test suite**

Run: `yarn workspace @casehubio/agent-viewer test`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add packages/agent-viewer/gallery.html packages/blocks-ui-schema/src/component-schemas.generated.ts packages/agent-viewer/src/index.ts
git commit -m "feat(#242): add drill-down gallery section and update schema"
```

## References

- [2026-10-19-agent-project-drill-down-design.md] — design spec this plan implements
- [packages/agent-viewer/src/agent-viewer.ts] — existing agent-viewer component
- [packages/agent-viewer/src/agent-viewer.test.ts] — existing test patterns
- [packages/agent-viewer/src/coordination-pipeline-view.ts] — drill-down source
- [packages/agent-viewer/src/coordination-types.ts] — CoordinationStep types
- [packages/agent-viewer/src/coordination-graph-model.ts] — GraphNode type
- [packages/agent-viewer/gallery.html] — gallery patterns
- [GitHub #242] — focal issue
- [GitHub #239] — parent epic
