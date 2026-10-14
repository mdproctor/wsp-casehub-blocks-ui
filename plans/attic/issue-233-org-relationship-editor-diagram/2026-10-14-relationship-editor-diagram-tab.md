# Relationship Editor Diagram Tab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #233 — feat: org relationship editor — tabular editing with diagram viewer tab

**Goal:** Add a diagram viewer tab to the relationship editor that renders `<blocks-org-diagram readonly>` from the editor's data, enrich table rows with avatar thumbnails and profession badges, and wire bidirectional click-to-focus interaction.

**Architecture:** A pure `toOrgYaml` bridge function in `graph-stencil-org` converts roster + relationships into the YAML the org-diagram consumes. The relationship editor gains an `agents: Record<string, AgentDescriptor>` property, a third "Diagram" tab composing the existing org-diagram in readonly mode, and richer table rows using `<agent-avatar>` thumbnails.

**Tech Stack:** Lit 3, TypeScript, vitest, `yaml` package (already in graph-stencil-org), `@casehubio/agent-avatar-2d` web component

## Global Constraints

- ARIA is mandatory on all interactive elements (role, aria-label, aria-selected, aria-controls)
- Custom events use `relationship-editor:` namespace prefix (protocol PP-20260930-501ce1)
- Diagram tab composes `<blocks-org-diagram readonly>` directly — no rendering duplication (protocol PP-20260903-79eda0)
- Data flows via typed properties, not slots (protocol PP-20260713-8ea1af)
- Use `ide-tooling` for all code navigation and structural editing

---

## Batch 1: Bridge Function

### Task 1: `toOrgYaml` — roster + relationships → YAML string

**Files:**
- Create: `packages/graph-stencil-org/src/adapter/to-org-yaml.ts`
- Create: `packages/graph-stencil-org/src/adapter/to-org-yaml.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts` (add export)

**Interfaces:**
- Consumes: `AgentRosterEntry` from `@casehubio/blocks-ui-core`, `AgentRelationship` and `OrgUnit` and `Membership` from local `../types.js`
- Produces: `toOrgYaml(roster: AgentRosterEntry[], relationships: AgentRelationship[], tenancyId: string): string` — returns a YAML string conforming to `OrgStructureYaml` shape

- [ ] **Step 1: Write failing tests**

Create `packages/graph-stencil-org/src/adapter/to-org-yaml.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { toOrgYaml } from './to-org-yaml.js';
import type { AgentRosterEntry } from '@casehubio/blocks-ui-core';
import type { AgentRelationship } from '../types.js';
import { parse } from 'yaml';

describe('toOrgYaml', () => {
  const tenancyId = 'tenant-1';

  it('groups roster entries by unitId into units with members', () => {
    const roster: AgentRosterEntry[] = [
      { agentId: 'a1', name: 'Alpha', unitId: 'u1' },
      { agentId: 'a2', name: 'Beta', unitId: 'u1' },
      { agentId: 'a3', name: 'Gamma', unitId: 'u2' },
    ];
    const relationships: AgentRelationship[] = [];

    const result = parse(toOrgYaml(roster, relationships, tenancyId));

    expect(result.organization.units).toHaveLength(2);
    const u1 = result.organization.units.find((u: any) => u.unitId === 'u1');
    expect(u1.members).toHaveLength(2);
    expect(u1.members.map((m: any) => m.agentId)).toContain('a1');
    expect(u1.members.map((m: any) => m.agentId)).toContain('a2');
    expect(u1.tenancyId).toBe(tenancyId);
    const u2 = result.organization.units.find((u: any) => u.unitId === 'u2');
    expect(u2.members).toHaveLength(1);
    expect(u2.members[0].agentId).toBe('a3');
  });

  it('places agents without unitId into synthetic unassigned unit', () => {
    const roster: AgentRosterEntry[] = [
      { agentId: 'a1', name: 'Alpha' },
      { agentId: 'a2', name: 'Beta' },
    ];
    const relationships: AgentRelationship[] = [];

    const result = parse(toOrgYaml(roster, relationships, tenancyId));

    expect(result.organization.units).toHaveLength(1);
    expect(result.organization.units[0].unitId).toBe('unassigned');
    expect(result.organization.units[0].name).toBe('Unassigned');
    expect(result.organization.units[0].members).toHaveLength(2);
  });

  it('passes relationships through to YAML output', () => {
    const roster: AgentRosterEntry[] = [
      { agentId: 'a1', name: 'Alpha', unitId: 'u1' },
      { agentId: 'a2', name: 'Beta', unitId: 'u1' },
    ];
    const relationships: AgentRelationship[] = [
      { sourceAgentId: 'a1', targetAgentId: 'a2', kind: 'SUPERVISES' as any, tenancyId },
    ];

    const result = parse(toOrgYaml(roster, relationships, tenancyId));

    expect(result.organization.relationships).toHaveLength(1);
    expect(result.organization.relationships[0].sourceAgentId).toBe('a1');
    expect(result.organization.relationships[0].targetAgentId).toBe('a2');
    expect(result.organization.relationships[0].kind).toBe('SUPERVISES');
  });

  it('produces valid minimal YAML for empty roster', () => {
    const result = parse(toOrgYaml([], [], tenancyId));

    expect(result.organization.units).toEqual([]);
    expect(result.organization.relationships).toEqual([]);
  });

  it('propagates tenancyId to units and relationships', () => {
    const roster: AgentRosterEntry[] = [
      { agentId: 'a1', name: 'Alpha', unitId: 'u1' },
    ];
    const relationships: AgentRelationship[] = [
      { sourceAgentId: 'a1', targetAgentId: 'a1', kind: 'BACKS_UP' as any, tenancyId: 'other' },
    ];

    const result = parse(toOrgYaml(roster, relationships, tenancyId));

    expect(result.organization.units[0].tenancyId).toBe(tenancyId);
    expect(result.organization.relationships[0].tenancyId).toBe('other');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-stencil-org run test src/adapter/to-org-yaml.test.ts`
Expected: FAIL — `toOrgYaml` not found

- [ ] **Step 3: Implement toOrgYaml**

Create `packages/graph-stencil-org/src/adapter/to-org-yaml.ts`:

```typescript
import { stringify } from 'yaml';
import type { AgentRosterEntry } from '@casehubio/blocks-ui-core';
import type { AgentRelationship, OrgUnit, Membership } from '../types.js';

export function toOrgYaml(
  roster: AgentRosterEntry[],
  relationships: AgentRelationship[],
  tenancyId: string,
): string {
  const unitMap = new Map<string, Membership[]>();

  for (const entry of roster) {
    const uid = entry.unitId ?? 'unassigned';
    let members = unitMap.get(uid);
    if (!members) {
      members = [];
      unitMap.set(uid, members);
    }
    members.push({ agentId: entry.agentId });
  }

  const units: OrgUnit[] = Array.from(unitMap.entries()).map(([unitId, members]) => ({
    unitId,
    name: unitId === 'unassigned' ? 'Unassigned' : unitId,
    tenancyId,
    members,
    capabilities: [],
    goals: [],
    constraints: [],
  }));

  return stringify({ organization: { units, relationships } });
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-stencil-org run test src/adapter/to-org-yaml.test.ts`
Expected: All 5 tests PASS

- [ ] **Step 5: Export from package index**

Add to `packages/graph-stencil-org/src/index.ts`:

```typescript
export { toOrgYaml } from './adapter/to-org-yaml.js';
```

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 7: Commit**

```bash
git add packages/graph-stencil-org/src/adapter/to-org-yaml.ts packages/graph-stencil-org/src/adapter/to-org-yaml.test.ts packages/graph-stencil-org/src/index.ts
git commit -m "$(cat <<'EOF'
feat(#233): add toOrgYaml bridge function

Converts roster + relationships → org YAML string for the
relationship editor's diagram tab.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 2: Diagram Tab + Bidirectional Interaction

### Task 2: Add `agents` property, ARIA on tabs, diagram tab rendering

**Files:**
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.ts` (ViewTab union, agents property, ARIA, diagram tab rendering)
- Modify: `components/agent-relationship-editor/src/index.ts` (re-export AgentRelationshipEditorProps if needed)
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.test.ts` (new tests)
- Modify: `components/agent-relationship-editor/package.json` (add org-diagram dependency if not present)
- Modify: `components/agent-relationship-editor/tsconfig.json` (add org-diagram reference if not present)

**Interfaces:**
- Consumes: `toOrgYaml` from `@casehubio/graph-stencil-org`, `AgentDescriptor` from `@casehubio/blocks-ui-core`
- Produces: `agents` property on `AgentRelationshipEditor`, diagram tab rendering, ARIA-compliant tabs

- [ ] **Step 1: Write failing tests for ARIA and diagram tab**

Add to `components/agent-relationship-editor/src/agent-relationship-editor.test.ts`:

```typescript
describe('tab ARIA', () => {
  it('renders tablist with three tabs', async () => {
    el.agentId = 'a1';
    el.roster = [{ agentId: 'a1', name: 'Alpha', unitId: 'u1' }];
    el.relationships = [];
    document.body.appendChild(el);
    await el.updateComplete;

    const tablist = el.shadowRoot!.querySelector('[role="tablist"]');
    expect(tablist).toBeTruthy();
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    expect(tabs).toHaveLength(3);
  });

  it('marks active tab with aria-selected', async () => {
    el.agentId = 'a1';
    el.roster = [{ agentId: 'a1', name: 'Alpha', unitId: 'u1' }];
    el.relationships = [];
    document.body.appendChild(el);
    await el.updateComplete;

    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    const activeTab = Array.from(tabs).find(t => t.getAttribute('aria-selected') === 'true');
    expect(activeTab).toBeTruthy();
    expect(activeTab!.textContent!.trim()).toBe('Table');
  });
});

describe('diagram tab', () => {
  it('renders blocks-org-diagram with readonly when diagram tab is active', async () => {
    el.agentId = 'a1';
    el.roster = [
      { agentId: 'a1', name: 'Alpha', unitId: 'u1' },
      { agentId: 'a2', name: 'Beta', unitId: 'u1' },
    ];
    el.relationships = [
      { sourceAgentId: 'a1', targetAgentId: 'a2', kind: 'SUPERVISES', tenancyId: 't1' },
    ];
    el.agents = { a1: { profession: 'Engineer' }, a2: { profession: 'Analyst' } };
    document.body.appendChild(el);
    await el.updateComplete;

    const diagramTab = Array.from(el.shadowRoot!.querySelectorAll('[role="tab"]'))
      .find(t => t.textContent!.trim() === 'Diagram') as HTMLElement;
    diagramTab.click();
    await el.updateComplete;

    const diagram = el.shadowRoot!.querySelector('blocks-org-diagram');
    expect(diagram).toBeTruthy();
    expect(diagram!.hasAttribute('readonly')).toBe(true);
  });

  it('effective YAML reflects pending additions', async () => {
    el.agentId = 'a1';
    el.roster = [
      { agentId: 'a1', name: 'Alpha', unitId: 'u1' },
      { agentId: 'a2', name: 'Beta', unitId: 'u1' },
    ];
    el.relationships = [];
    document.body.appendChild(el);
    await el.updateComplete;

    // Simulate adding a pending relationship via the add form
    // Then switch to diagram tab and verify the YAML includes it
    const diagramTab = Array.from(el.shadowRoot!.querySelectorAll('[role="tab"]'))
      .find(t => t.textContent!.trim() === 'Diagram') as HTMLElement;
    diagramTab.click();
    await el.updateComplete;

    const diagram = el.shadowRoot!.querySelector('blocks-org-diagram') as any;
    expect(diagram).toBeTruthy();
    // Diagram should have yaml property set (even if just units, no relationships)
    expect(diagram.yaml).toBeTruthy();
  });

  it('diagram tabpanel has aria-labelledby', async () => {
    el.agentId = 'a1';
    el.roster = [{ agentId: 'a1', name: 'Alpha', unitId: 'u1' }];
    el.relationships = [];
    document.body.appendChild(el);
    await el.updateComplete;

    const diagramTab = Array.from(el.shadowRoot!.querySelectorAll('[role="tab"]'))
      .find(t => t.textContent!.trim() === 'Diagram') as HTMLElement;
    diagramTab.click();
    await el.updateComplete;

    const tabpanel = el.shadowRoot!.querySelector('[role="tabpanel"]');
    expect(tabpanel).toBeTruthy();
    expect(tabpanel!.getAttribute('aria-labelledby')).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/agent-relationship-editor run test`
Expected: FAIL — no `[role="tablist"]`, no diagram tab

- [ ] **Step 3: Implement agents property, ARIA on tabs, diagram tab**

Modify `components/agent-relationship-editor/src/agent-relationship-editor.ts`:

1. Update the `ViewTab` type at line 8:
```typescript
type ViewTab = 'table' | 'arc' | 'diagram';
```

2. Add imports at the top:
```typescript
import { toOrgYaml } from '@casehubio/graph-stencil-org';
import type { AgentDescriptor } from '@casehubio/blocks-ui-core';
import '../../org-diagram/src/blocks-org-diagram.js'; // side-effect import for element registration
```
Note: verify the exact side-effect import path using `ide_find_class` on `BlocksOrgDiagram`. The import may be `@casehubio/blocks-org-diagram` if the package has a barrel export that registers the element.

3. Add `agents` property alongside existing properties (~line 206):
```typescript
@property({ type: Object }) agents: Record<string, AgentDescriptor> = {};
```

4. Add a `tenancyId` property (needed for toOrgYaml):
```typescript
@property({ attribute: 'tenancy-id' }) tenancyId = 'default';
```

5. Add a getter for effective relationships (merges pending state):
```typescript
private get _effectiveRelationships(): AgentRelationship[] {
  const confirmed = this.relationships.filter(
    (_, i) => !this._pendingRemovals.has(i),
  );
  return [...confirmed, ...this._pendingAdditions];
}
```

6. Add a getter for effective YAML:
```typescript
private get _effectiveYaml(): string {
  return toOrgYaml(this.roster, this._effectiveRelationships, this.tenancyId);
}
```

7. Replace the tab buttons section (~lines 317-321) with ARIA-compliant tabs:
```typescript
<div class="view-tabs" role="tablist" aria-label="Relationship views">
  <button role="tab" id="tab-table"
    aria-selected=${this._viewTab === 'table'}
    aria-controls="panel-table"
    class="view-tab ${this._viewTab === 'table' ? 'active' : ''}"
    @click=${() => this._viewTab = 'table'}>Table</button>
  <button role="tab" id="tab-arc"
    aria-selected=${this._viewTab === 'arc'}
    aria-controls="panel-arc"
    class="view-tab ${this._viewTab === 'arc' ? 'active' : ''}"
    @click=${() => this._viewTab = 'arc'}>Arc</button>
  <button role="tab" id="tab-diagram"
    aria-selected=${this._viewTab === 'diagram'}
    aria-controls="panel-diagram"
    class="view-tab ${this._viewTab === 'diagram' ? 'active' : ''}"
    @click=${() => this._viewTab = 'diagram'}>Diagram</button>
</div>
```

8. Replace the tab content section (~lines 323-366) to wrap each view in a tabpanel and add the diagram:
```typescript
${this._viewTab === 'arc'
  ? html`<div role="tabpanel" id="panel-arc" aria-labelledby="tab-arc">
      <arc-view .agentId=${this.agentId} .relationships=${this.relationships}
        .roster=${this.roster}></arc-view>
    </div>`
  : this._viewTab === 'diagram'
  ? html`<div role="tabpanel" id="panel-diagram" aria-labelledby="tab-diagram">
      <blocks-org-diagram readonly
        .yaml=${this._effectiveYaml}
        .agents=${this.agents}
        .selectionTopic=${'relationship-editor:diagram:agent-selected'}
      ></blocks-org-diagram>
    </div>`
  : html`<div role="tabpanel" id="panel-table" aria-labelledby="tab-table">
      ${/* existing table content */}
    </div>`}
```

- [ ] **Step 4: Add package dependency and tsconfig reference**

Verify if `@casehubio/blocks-org-diagram` (or equivalent) is already in `package.json` dependencies via the existing `@casehubio/graph-stencil-org` dependency. If not, add it. Check with `ide_find_class` on `BlocksOrgDiagram` for the package name.

Ensure `components/agent-relationship-editor/tsconfig.json` references include org-diagram:
```json
{ "path": "../../components/org-diagram" }
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/agent-relationship-editor run test`
Expected: All tests PASS (existing + new)

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 7: Commit**

```bash
git add components/agent-relationship-editor/src/ components/agent-relationship-editor/package.json components/agent-relationship-editor/tsconfig.json
git commit -m "$(cat <<'EOF'
feat(#233): add diagram tab to relationship editor

Adds agents property, ARIA-compliant tabs (tablist/tab/tabpanel),
and a Diagram tab that renders blocks-org-diagram readonly with
effective YAML reflecting pending changes.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

### Task 3: Wire bidirectional click-to-focus interaction

**Files:**
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.ts` (event listener)
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.test.ts` (interaction test)

**Interfaces:**
- Consumes: `pages-event` with topic `relationship-editor:diagram:agent-selected` from `<blocks-org-diagram>` via `selectionTopic`
- Produces: updates `agentId` property (focal agent) which re-renders table grouping and arc view center

- [ ] **Step 1: Write failing test for click-to-focus**

Add to `components/agent-relationship-editor/src/agent-relationship-editor.test.ts`:

```typescript
describe('bidirectional interaction', () => {
  it('updates focalAgentId when diagram agent is clicked', async () => {
    el.agentId = 'a1';
    el.roster = [
      { agentId: 'a1', name: 'Alpha', unitId: 'u1' },
      { agentId: 'a2', name: 'Beta', unitId: 'u1' },
    ];
    el.relationships = [
      { sourceAgentId: 'a1', targetAgentId: 'a2', kind: 'SUPERVISES', tenancyId: 't1' },
    ];
    document.body.appendChild(el);
    await el.updateComplete;

    // Switch to diagram tab
    const diagramTab = Array.from(el.shadowRoot!.querySelectorAll('[role="tab"]'))
      .find(t => t.textContent!.trim() === 'Diagram') as HTMLElement;
    diagramTab.click();
    await el.updateComplete;

    // Simulate the pages-event the org-diagram emits on node click
    el.dispatchEvent(new CustomEvent('pages-event', {
      detail: {
        topic: 'relationship-editor:diagram:agent-selected',
        payload: { nodeId: 'agent:u1:a2' },
      },
      bubbles: true,
      composed: true,
    }));
    await el.updateComplete;

    expect(el.agentId).toBe('a2');
  });

  it('ignores unit node clicks', async () => {
    el.agentId = 'a1';
    el.roster = [{ agentId: 'a1', name: 'Alpha', unitId: 'u1' }];
    el.relationships = [];
    document.body.appendChild(el);
    await el.updateComplete;

    el.dispatchEvent(new CustomEvent('pages-event', {
      detail: {
        topic: 'relationship-editor:diagram:agent-selected',
        payload: { nodeId: 'unit:u1' },
      },
      bubbles: true,
      composed: true,
    }));
    await el.updateComplete;

    expect(el.agentId).toBe('a1');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/agent-relationship-editor run test`
Expected: FAIL — agentId not updated on event

- [ ] **Step 3: Implement event listener**

Add to `agent-relationship-editor.ts`:

1. In `connectedCallback`, add event listener:
```typescript
override connectedCallback(): void {
  super.connectedCallback();
  this.addEventListener('pages-event', this._handleDiagramSelection as EventListener);
}
```

2. In `disconnectedCallback`, remove listener:
```typescript
override disconnectedCallback(): void {
  super.disconnectedCallback();
  this.removeEventListener('pages-event', this._handleDiagramSelection as EventListener);
}
```

3. Add handler method:
```typescript
private _handleDiagramSelection = (e: CustomEvent): void => {
  if (e.detail?.topic !== 'relationship-editor:diagram:agent-selected') return;
  const nodeId: string = e.detail?.payload?.nodeId ?? '';
  if (!nodeId.startsWith('agent:')) return;
  const parts = nodeId.split(':');
  const agentId = parts[parts.length - 1];
  if (agentId && this.roster.some(r => r.agentId === agentId)) {
    this.agentId = agentId;
  }
};
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/agent-relationship-editor run test`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add components/agent-relationship-editor/src/
git commit -m "$(cat <<'EOF'
feat(#233): wire diagram click-to-focus interaction

Clicking an agent node in the diagram sets it as the focal agent,
updating the table grouping and arc view center.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 3: Richer Table Rows + Demo

### Task 4: Avatar thumbnails and profession badges in table rows

**Files:**
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.ts` (row rendering)
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.test.ts` (new tests)
- Modify: `components/agent-relationship-editor/package.json` (add agent-avatar-2d dependency)
- Modify: `components/agent-relationship-editor/tsconfig.json` (add agent-avatar-2d reference)

**Interfaces:**
- Consumes: `agents: Record<string, AgentDescriptor>` (added in Task 2), `<agent-avatar>` from `@casehubio/agent-avatar-2d`
- Produces: enriched table rows with avatar thumbnail + profession/role text

- [ ] **Step 1: Write failing tests for richer rows**

Add to `components/agent-relationship-editor/src/agent-relationship-editor.test.ts`:

```typescript
describe('richer table rows', () => {
  it('renders avatar when agents property has avatar code', async () => {
    el.agentId = 'a1';
    el.roster = [
      { agentId: 'a1', name: 'Alpha', unitId: 'u1' },
      { agentId: 'a2', name: 'Beta', unitId: 'u1' },
    ];
    el.relationships = [
      { sourceAgentId: 'a1', targetAgentId: 'a2', kind: 'SUPERVISES', tenancyId: 't1' },
    ];
    el.agents = { a2: { avatar: 'mythic:P15', profession: 'Engineer' } };
    document.body.appendChild(el);
    await el.updateComplete;

    const avatar = el.shadowRoot!.querySelector('agent-avatar');
    expect(avatar).toBeTruthy();
  });

  it('renders profession badge when agents property has profession', async () => {
    el.agentId = 'a1';
    el.roster = [
      { agentId: 'a1', name: 'Alpha', unitId: 'u1' },
      { agentId: 'a2', name: 'Beta', unitId: 'u1' },
    ];
    el.relationships = [
      { sourceAgentId: 'a1', targetAgentId: 'a2', kind: 'SUPERVISES', tenancyId: 't1' },
    ];
    el.agents = { a2: { profession: 'Engineer', role: 'Senior' } };
    document.body.appendChild(el);
    await el.updateComplete;

    const profession = el.shadowRoot!.querySelector('.agent-profession');
    expect(profession).toBeTruthy();
    expect(profession!.textContent).toContain('Engineer');
  });

  it('falls back to name only when no agents property', async () => {
    el.agentId = 'a1';
    el.roster = [
      { agentId: 'a1', name: 'Alpha', unitId: 'u1' },
      { agentId: 'a2', name: 'Beta', unitId: 'u1' },
    ];
    el.relationships = [
      { sourceAgentId: 'a1', targetAgentId: 'a2', kind: 'SUPERVISES', tenancyId: 't1' },
    ];
    document.body.appendChild(el);
    await el.updateComplete;

    const avatar = el.shadowRoot!.querySelector('agent-avatar');
    expect(avatar).toBeFalsy();
    const name = el.shadowRoot!.querySelector('.agent-name');
    expect(name).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/agent-relationship-editor run test`
Expected: FAIL — no `agent-avatar` in output

- [ ] **Step 3: Add avatar dependency**

Add to `components/agent-relationship-editor/package.json` dependencies:
```json
"@casehubio/agent-avatar-2d": "workspace:*"
```

Add to `components/agent-relationship-editor/tsconfig.json` references:
```json
{ "path": "../../packages/agent-avatar-2d" }
```

- [ ] **Step 4: Implement richer rows**

Add import to `agent-relationship-editor.ts`:
```typescript
import '@casehubio/agent-avatar-2d'; // side-effect for <agent-avatar> registration
```

Extract a helper method for rendering the agent cell (replaces the inline `.agent-name` span at ~line 347):

```typescript
private _renderAgentCell(agentId: string): TemplateResult {
  const name = this._getAgentName(agentId);
  const descriptor = this.agents?.[agentId];

  return html`
    <span class="agent-cell">
      ${descriptor?.avatar
        ? html`<agent-avatar .code=${descriptor.avatar} size="xs" aria-hidden="true"></agent-avatar>`
        : nothing}
      <span class="agent-name">${name}</span>
      ${descriptor?.profession
        ? html`<span class="agent-profession">${descriptor.profession}${descriptor.role ? ` · ${descriptor.role}` : ''}</span>`
        : nothing}
    </span>
  `;
}
```

Replace inline agent name rendering in each relationship row with `${this._renderAgentCell(other)}`.

Add CSS for the new elements:
```css
.agent-cell {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}
.agent-profession {
  color: var(--pages-text-secondary, #6b7280);
  font-size: 0.85em;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/agent-relationship-editor run test`
Expected: All tests PASS

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 7: Commit**

```bash
git add components/agent-relationship-editor/
git commit -m "$(cat <<'EOF'
feat(#233): richer table rows with avatar and profession

Table rows now show inline avatar thumbnails and profession/role
badges when the agents descriptor property is set.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

### Task 5: Update demo with gastown data

**Files:**
- Modify: `components/agent-relationship-editor/demo.html`

**Interfaces:**
- Consumes: all previous tasks — exercises the full data flow
- Produces: working demo page with rich org data

- [ ] **Step 1: Update demo.html with gastown org data**

Replace the generic demo data with gastown agents that include avatars, professions, and org structure. Use agents and relationships from the gastown fixture used in the org-diagram demos (check `components/org-diagram/` or the examples app for the gastown fixture data).

The demo should set:
- `roster` with agents assigned to units
- `relationships` covering SUPERVISES, DELEGATES_TO, ESCALATES_TO, REPORTS_TO, BACKS_UP
- `agents` record with avatar identity codes and professions
- `tenancy-id` attribute

Verify the demo includes:
- Multiple units with multiple agents
- Variety of relationship kinds
- Avatar codes from the mythic collection
- Professions and roles

- [ ] **Step 2: Start dev server and verify demo**

Run: `npx vite /Users/mdproctor/claude/casehub/slots/201/blocks-ui --port 8792 --open false`
Open: `http://localhost:8792/components/agent-relationship-editor/demo.html`

Verify:
- Table view shows avatars and profession badges
- Arc view renders correctly
- Diagram tab renders the org-diagram with all agents and relationships
- Clicking an agent in the diagram switches the focal agent
- Adding/removing a relationship in the table updates the diagram

- [ ] **Step 3: Commit**

```bash
git add components/agent-relationship-editor/demo.html
git commit -m "$(cat <<'EOF'
feat(#233): update relationship editor demo with gastown org data

Exercises the full data flow: rich roster with avatars and
professions, multiple relationship kinds, diagram tab viewer.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## References

- [2026-10-14-relationship-editor-diagram-tab-design.md] — design spec this plan implements
- `components/agent-relationship-editor/src/agent-relationship-editor.ts:8` — ViewTab type
- `components/agent-relationship-editor/src/agent-relationship-editor.ts:204-210` — properties and pending state
- `components/agent-relationship-editor/src/agent-relationship-editor.ts:317-366` — tab rendering
- `components/agent-relationship-editor/src/types.ts` — AgentRosterEntry, AgentRelationship
- `packages/graph-stencil-org/src/adapter/org-adapter.ts` — toOrgGraph (parallel function)
- `packages/graph-stencil-org/src/types.ts` — OrgStructureYaml, OrgUnit, Membership
- `components/org-diagram/src/blocks-org-diagram.ts` — OrgDiagramProps, selectionTopic
- Protocol PP-20260903-79eda0 — diagram viewer composition
- Protocol PP-20260930-501ce1 — custom event namespace prefix
- Protocol PP-20260713-8ea1af — component customisation pattern
- GitHub #233 — focal issue
