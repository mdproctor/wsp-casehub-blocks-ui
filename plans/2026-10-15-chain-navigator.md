# Chain Navigator Tab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #234 — Interactive chain navigator with inline editing
**Issue group:** #234

**Goal:** Add a third "Chain" tab to the relationship editor that renders relationship chains as indented trees with inline CRUD operations.

**Architecture:** Chain tab is inline in `agent-relationship-editor.ts` (same pattern as table tab). Core tree-building logic is a pure function in `chain-tree.ts`. All edits share the existing `_pendingAdditions` / `_pendingRemovals` changeset model. Tree is rebuilt eagerly whenever `_effectiveRelationships` changes or the selected kind changes.

**Tech Stack:** Lit 3.x, TypeScript, vitest + jsdom

## Global Constraints

- Events use `chain:` namespace prefix per `custom-event-namespace-prefix` protocol
- ARIA attributes required on every interactive element
- Full avatar pills (thumbnail + name + profession) for agent rendering — reuse existing `_renderAgentCell()`
- All edits go through `_pendingAdditions` / `_pendingRemovals` — no new changeset types
- Run tests with `npx vitest run` from `components/agent-relationship-editor/`

---

## Batch 1: Chain tree pure function + tests

### Task 1: buildChainForest pure function with tests

**Files:**
- Create: `components/agent-relationship-editor/src/chain-tree.ts`
- Create: `components/agent-relationship-editor/src/chain-tree.test.ts`

**Interfaces:**
- Consumes: `AgentRelationship` and `RelationshipKind` types from `@casehubio/graph-stencil-org`
- Produces: `ChainNode` interface and `buildChainForest(relationships: AgentRelationship[], kind: RelationshipKind): ChainNode[]` function — used by Task 2 for rendering

- [ ] **Step 1: Write the ChainNode interface and buildChainForest stub**

Create `components/agent-relationship-editor/src/chain-tree.ts`:

```typescript
import type { AgentRelationship, RelationshipKind } from '@casehubio/graph-stencil-org';

export interface ChainNode {
  agentId: string;
  children: ChainNode[];
  isCycle: boolean;
  relationshipIndex: number;
  isPending: boolean;
}

export function buildChainForest(
  relationships: AgentRelationship[],
  kind: RelationshipKind,
  confirmedCount: number,
): ChainNode[] {
  return [];
}
```

The `confirmedCount` parameter is the length of the original `this.relationships` array (before pending additions). Relationships at index >= confirmedCount are pending additions (isPending = true). Relationships whose index is in the removals set should be excluded by the caller before passing to this function.

- [ ] **Step 2: Write failing tests for linear chain, branching, roots, empty**

Create `components/agent-relationship-editor/src/chain-tree.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { buildChainForest } from './chain-tree.js';
import type { AgentRelationship } from '@casehubio/graph-stencil-org';

const rel = (src: string, tgt: string, kind: string, idx?: number): AgentRelationship => ({
  sourceAgentId: src,
  targetAgentId: tgt,
  kind: kind as any,
  tenancyId: 't1',
});

describe('buildChainForest', () => {
  it('returns empty forest for empty relationships', () => {
    expect(buildChainForest([], 'SUPERVISES', 0)).toEqual([]);
  });

  it('returns empty forest when no relationships match the kind', () => {
    const rels = [rel('a', 'b', 'SUPERVISES')];
    expect(buildChainForest(rels, 'ESCALATES_TO', 1)).toEqual([]);
  });

  it('builds a linear chain with correct root', () => {
    const rels = [
      rel('a', 'b', 'SUPERVISES'),
      rel('b', 'c', 'SUPERVISES'),
    ];
    const forest = buildChainForest(rels, 'SUPERVISES', 2);
    expect(forest).toHaveLength(1);
    expect(forest[0].agentId).toBe('a');
    expect(forest[0].children).toHaveLength(1);
    expect(forest[0].children[0].agentId).toBe('b');
    expect(forest[0].children[0].children).toHaveLength(1);
    expect(forest[0].children[0].children[0].agentId).toBe('c');
    expect(forest[0].children[0].children[0].children).toHaveLength(0);
  });

  it('builds a branching tree', () => {
    const rels = [
      rel('a', 'b', 'SUPERVISES'),
      rel('a', 'c', 'SUPERVISES'),
      rel('b', 'd', 'SUPERVISES'),
    ];
    const forest = buildChainForest(rels, 'SUPERVISES', 3);
    expect(forest).toHaveLength(1);
    expect(forest[0].agentId).toBe('a');
    expect(forest[0].children).toHaveLength(2);
    const childIds = forest[0].children.map(c => c.agentId).sort();
    expect(childIds).toEqual(['b', 'c']);
    const bNode = forest[0].children.find(c => c.agentId === 'b')!;
    expect(bNode.children).toHaveLength(1);
    expect(bNode.children[0].agentId).toBe('d');
  });

  it('detects multiple roots', () => {
    const rels = [
      rel('a', 'b', 'SUPERVISES'),
      rel('c', 'd', 'SUPERVISES'),
    ];
    const forest = buildChainForest(rels, 'SUPERVISES', 2);
    expect(forest).toHaveLength(2);
    const rootIds = forest.map(n => n.agentId).sort();
    expect(rootIds).toEqual(['a', 'c']);
  });

  it('detects cycles and marks cycle node', () => {
    const rels = [
      rel('a', 'b', 'SUPERVISES'),
      rel('b', 'a', 'SUPERVISES'),
    ];
    const forest = buildChainForest(rels, 'SUPERVISES', 2);
    expect(forest).toHaveLength(1);
    const root = forest[0];
    expect(root.children).toHaveLength(1);
    const child = root.children[0];
    expect(child.isCycle).toBe(true);
    expect(child.children).toHaveLength(0);
  });

  it('handles BACKS_UP as bidirectional pairs — both agents are roots', () => {
    const rels = [
      rel('a', 'b', 'BACKS_UP'),
    ];
    const forest = buildChainForest(rels, 'BACKS_UP', 1);
    expect(forest).toHaveLength(2);
    const rootIds = forest.map(n => n.agentId).sort();
    expect(rootIds).toEqual(['a', 'b']);
  });

  it('marks pending additions with isPending=true', () => {
    const rels = [
      rel('a', 'b', 'SUPERVISES'),
      rel('b', 'c', 'SUPERVISES'),
    ];
    const forest = buildChainForest(rels, 'SUPERVISES', 1);
    const root = forest[0];
    expect(root.isPending).toBe(false);
    expect(root.children[0].isPending).toBe(true);
  });

  it('sets relationshipIndex correctly', () => {
    const rels = [
      rel('x', 'y', 'DELEGATES_TO'),
      rel('a', 'b', 'SUPERVISES'),
      rel('b', 'c', 'SUPERVISES'),
    ];
    const forest = buildChainForest(rels, 'SUPERVISES', 3);
    expect(forest[0].relationshipIndex).toBe(-1);
    expect(forest[0].children[0].relationshipIndex).toBe(1);
    expect(forest[0].children[0].children[0].relationshipIndex).toBe(2);
  });

  it('root nodes have relationshipIndex -1', () => {
    const rels = [rel('a', 'b', 'SUPERVISES')];
    const forest = buildChainForest(rels, 'SUPERVISES', 1);
    expect(forest[0].relationshipIndex).toBe(-1);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `npx vitest run src/chain-tree.test.ts` from `components/agent-relationship-editor/`
Expected: FAIL — `buildChainForest` returns empty array for all cases

- [ ] **Step 4: Implement buildChainForest**

Update `components/agent-relationship-editor/src/chain-tree.ts`:

```typescript
import type { AgentRelationship, RelationshipKind } from '@casehubio/graph-stencil-org';

export interface ChainNode {
  agentId: string;
  children: ChainNode[];
  isCycle: boolean;
  relationshipIndex: number;
  isPending: boolean;
}

export function buildChainForest(
  relationships: AgentRelationship[],
  kind: RelationshipKind,
  confirmedCount: number,
): ChainNode[] {
  const kindRels: { rel: AgentRelationship; index: number }[] = [];
  for (let i = 0; i < relationships.length; i++) {
    if (relationships[i].kind === kind) {
      kindRels.push({ rel: relationships[i], index: i });
    }
  }

  if (kindRels.length === 0) return [];

  const childrenMap = new Map<string, { targetId: string; relIndex: number }[]>();
  const targetSet = new Set<string>();

  for (const { rel, index } of kindRels) {
    const existing = childrenMap.get(rel.sourceAgentId) ?? [];
    existing.push({ targetId: rel.targetAgentId, relIndex: index });
    childrenMap.set(rel.sourceAgentId, existing);
    targetSet.add(rel.targetAgentId);
  }

  // For BACKS_UP, add reverse edges so both agents appear as roots
  if (kind === 'BACKS_UP') {
    for (const { rel, index } of kindRels) {
      const existing = childrenMap.get(rel.targetAgentId) ?? [];
      if (!existing.some(e => e.targetId === rel.sourceAgentId)) {
        existing.push({ targetId: rel.sourceAgentId, relIndex: index });
        childrenMap.set(rel.targetAgentId, existing);
      }
      // Both agents are roots for BACKS_UP
    }
    targetSet.clear();
  }

  const rootIds = [...childrenMap.keys()].filter(id => !targetSet.has(id));

  function buildNode(agentId: string, relIndex: number, visited: Set<string>): ChainNode {
    if (visited.has(agentId)) {
      return { agentId, children: [], isCycle: true, relationshipIndex: relIndex, isPending: relIndex >= 0 && relIndex >= confirmedCount };
    }
    visited.add(agentId);
    const edges = childrenMap.get(agentId) ?? [];
    const children = edges.map(e => buildNode(e.targetId, e.relIndex, new Set(visited)));
    return {
      agentId,
      children,
      isCycle: false,
      relationshipIndex: relIndex,
      isPending: relIndex >= 0 && relIndex >= confirmedCount,
    };
  }

  return rootIds.map(id => buildNode(id, -1, new Set()));
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `npx vitest run src/chain-tree.test.ts` from `components/agent-relationship-editor/`
Expected: All 9 tests PASS

- [ ] **Step 6: Commit**

```bash
git add components/agent-relationship-editor/src/chain-tree.ts components/agent-relationship-editor/src/chain-tree.test.ts
git commit -m "feat(#234): add buildChainForest pure function with tests

Builds a forest of ChainNode trees from flat AgentRelationship array
for a given RelationshipKind. Handles cycle detection, BACKS_UP pairs,
pending addition marking, and multi-root forests.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Chain tab rendering in relationship editor

### Task 2: Add chain tab with kind picker and tree rendering

**Files:**
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.ts`
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.test.ts`

**Interfaces:**
- Consumes: `buildChainForest` and `ChainNode` from `./chain-tree.js` (Task 1)
- Produces: Chain tab UI with kind picker and tree rendering — used by Task 3 for inline editing

- [ ] **Step 1: Write failing tests for chain tab rendering**

Add to `components/agent-relationship-editor/src/agent-relationship-editor.test.ts`:

```typescript
// Add to SAMPLE_RELATIONSHIPS for chain tests
const CHAIN_RELATIONSHIPS: AgentRelationship[] = [
  { sourceAgentId: 'alice', targetAgentId: 'bob', kind: 'SUPERVISES', tenancyId: 't1' },
  { sourceAgentId: 'bob', targetAgentId: 'carol', kind: 'SUPERVISES', tenancyId: 't1' },
  { sourceAgentId: 'bob', targetAgentId: 'dave', kind: 'SUPERVISES', tenancyId: 't1' },
  { sourceAgentId: 'alice', targetAgentId: 'carol', kind: 'ESCALATES_TO', tenancyId: 't1' },
];

describe('chain tab', () => {
  let el: EditorEl;

  beforeEach(async () => {
    el = document.createElement('agent-relationship-editor') as EditorEl;
    el.agentId = 'alice';
    el.relationships = CHAIN_RELATIONSHIPS;
    el.roster = SAMPLE_ROSTER;
    document.body.appendChild(el);
    await el.updateComplete;

    // Switch to chain tab
    const chainTab = Array.from(el.shadowRoot!.querySelectorAll('[role="tab"]'))
      .find(t => t.textContent!.trim() === 'Chain') as HTMLElement;
    chainTab.click();
    await el.updateComplete;
  });

  afterEach(() => el.remove());

  it('renders three tabs including Chain', async () => {
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    expect(tabs).toHaveLength(3);
    const labels = Array.from(tabs).map(t => t.textContent!.trim());
    expect(labels).toEqual(['Table', 'Chain', 'Diagram']);
  });

  it('chain tab has aria-controls and aria-selected', async () => {
    const chainTab = Array.from(el.shadowRoot!.querySelectorAll('[role="tab"]'))
      .find(t => t.textContent!.trim() === 'Chain') as HTMLElement;
    expect(chainTab.getAttribute('aria-selected')).toBe('true');
    expect(chainTab.getAttribute('aria-controls')).toBe('panel-chain');
  });

  it('renders kind picker pills', async () => {
    const kindPicker = el.shadowRoot!.querySelector('.chain-kind-picker');
    expect(kindPicker).toBeTruthy();
    const pills = kindPicker!.querySelectorAll('.kind-pill');
    expect(pills.length).toBeGreaterThan(0);
  });

  it('kind pills show count badges', async () => {
    const pills = el.shadowRoot!.querySelectorAll('.kind-pill');
    const supervisesPill = Array.from(pills).find(p => p.textContent!.includes('SUPERVISES'));
    expect(supervisesPill).toBeTruthy();
    expect(supervisesPill!.querySelector('.kind-count')!.textContent!.trim()).toBe('3');
  });

  it('renders tree container with role="tree"', async () => {
    const tree = el.shadowRoot!.querySelector('[role="tree"]');
    expect(tree).toBeTruthy();
  });

  it('renders tree items with role="treeitem" and aria-level', async () => {
    const items = el.shadowRoot!.querySelectorAll('[role="treeitem"]');
    expect(items.length).toBeGreaterThan(0);
    const firstItem = items[0];
    expect(firstItem.getAttribute('aria-level')).toBeTruthy();
  });

  it('renders agent names in tree nodes', async () => {
    const items = el.shadowRoot!.querySelectorAll('[role="treeitem"]');
    const names = Array.from(items).map(i => i.querySelector('.agent-name')?.textContent?.trim());
    expect(names).toContain('Alice');
    expect(names).toContain('Bob');
  });

  it('renders connector arrows between parent and child', async () => {
    const connectors = el.shadowRoot!.querySelectorAll('.chain-connector');
    expect(connectors.length).toBeGreaterThan(0);
  });

  it('switches kind when pill is clicked', async () => {
    const pills = el.shadowRoot!.querySelectorAll('.kind-pill');
    const escalatePill = Array.from(pills).find(p => p.textContent!.includes('ESCALATES'));
    expect(escalatePill).toBeTruthy();
    (escalatePill as HTMLElement).click();
    await el.updateComplete;

    const items = el.shadowRoot!.querySelectorAll('[role="treeitem"]');
    const names = Array.from(items).map(i => i.querySelector('.agent-name')?.textContent?.trim());
    expect(names).toContain('Alice');
    expect(names).toContain('Carol');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/agent-relationship-editor.test.ts` from `components/agent-relationship-editor/`
Expected: FAIL — no Chain tab exists yet

- [ ] **Step 3: Update ViewTab type and add chain tab button**

In `agent-relationship-editor.ts`, change:
- `type ViewTab = 'table' | 'diagram';` → `type ViewTab = 'table' | 'chain' | 'diagram';`
- Add the import: `import { buildChainForest, type ChainNode } from './chain-tree.js';`
- Add state: `@state() private _chainKind: RelationshipKind = 'SUPERVISES';`
- Add the Chain tab button between Table and Diagram in the `render()` method
- Add the chain tab panel with `_renderChainTab()` method

Update the `render()` tab bar to include three tabs:

```typescript
<button role="tab" id="tab-table" aria-selected=${this._viewTab === 'table'} aria-controls="panel-table" class="view-tab ${this._viewTab === 'table' ? 'active' : ''}" @click=${() => { this._viewTab = 'table'; }}>Table</button>
<button role="tab" id="tab-chain" aria-selected=${this._viewTab === 'chain'} aria-controls="panel-chain" class="view-tab ${this._viewTab === 'chain' ? 'active' : ''}" @click=${() => { this._viewTab = 'chain'; }}>Chain</button>
<button role="tab" id="tab-diagram" aria-selected=${this._viewTab === 'diagram'} aria-controls="panel-diagram" class="view-tab ${this._viewTab === 'diagram' ? 'active' : ''}" @click=${() => { this._viewTab = 'diagram'; }}>Diagram</button>
```

Update the panel rendering to handle all three tabs (table, chain, diagram).

- [ ] **Step 4: Implement kind picker rendering**

Add `_renderChainTab()` method:

```typescript
private _renderChainTab() {
  const kindCounts = new Map<RelationshipKind, number>();
  for (const rel of this._effectiveRelationships) {
    kindCounts.set(rel.kind, (kindCounts.get(rel.kind) ?? 0) + 1);
  }

  return html`
    <div class="chain-kind-picker" role="tablist" aria-label="Relationship kinds">
      ${RELATIONSHIP_KINDS.map(kind => {
        const count = kindCounts.get(kind) ?? 0;
        const desc = lookupRelationshipType(kindToRegistryKey(kind));
        return html`
          <button role="tab" class="kind-pill ${this._chainKind === kind ? 'active' : ''}"
            aria-selected=${this._chainKind === kind}
            @click=${() => { this._chainKind = kind; this._emitChainKindSelected(kind); }}>
            <span class="kind-dot" style="background: ${desc.color}"></span>
            ${desc.label ?? kind}
            <span class="kind-count">${count}</span>
          </button>
        `;
      })}
    </div>
    ${this._renderChainTree()}
  `;
}
```

- [ ] **Step 5: Implement tree rendering**

Add `_renderChainTree()` and `_renderChainNode()` methods:

```typescript
private _renderChainTree() {
  const forest = buildChainForest(this._effectiveRelationships, this._chainKind, this.relationships.length);
  if (forest.length === 0) {
    return html`<div class="empty-state">No ${this._chainKind} chains.</div>`;
  }
  return html`
    <div role="tree" aria-label="${this._chainKind} chain hierarchy">
      ${forest.map(node => this._renderChainNode(node, 1))}
    </div>
  `;
}

private _renderChainNode(node: ChainNode, level: number) {
  const hasChildren = node.children.length > 0 && !node.isCycle;
  return html`
    <div role="treeitem" aria-level=${level}
      aria-expanded=${hasChildren ? 'true' : nothing}
      class="chain-node ${node.isPending ? 'pending-addition' : ''} ${node.isCycle ? 'chain-cycle' : ''}"
      style="padding-left: ${(level - 1) * 24}px">
      ${node.relationshipIndex >= 0 ? html`
        <span class="chain-connector" role="button"
          aria-label="Remove relationship to ${this._getAgentName(node.agentId)}">→</span>
      ` : nothing}
      ${this._renderAgentCell(node.agentId)}
      ${node.isCycle ? html`<span class="cycle-indicator" aria-label="Cycle detected — ${this._getAgentName(node.agentId)} already appears earlier in this chain">↻</span>` : nothing}
    </div>
    ${hasChildren ? node.children.map(child => this._renderChainNode(child, level + 1)) : nothing}
  `;
}
```

- [ ] **Step 6: Add chain:kind:selected event emission**

Add method to emit the namespaced event per protocol:

```typescript
private _emitChainKindSelected(kind: RelationshipKind) {
  this.dispatchEvent(new CustomEvent('pages-event', {
    bubbles: true,
    composed: true,
    detail: { topic: 'chain:kind:selected', payload: { kind } },
  }));
}
```

- [ ] **Step 7: Add CSS styles for chain tab**

Add to `static styles`:

```css
.chain-kind-picker {
  display: flex;
  gap: 4px;
  margin-bottom: var(--pages-space-3, 0.75rem);
  flex-wrap: wrap;
}
.kind-pill {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 10px;
  border: 1px solid var(--pages-neutral-4, #e5e5e5);
  border-radius: 12px;
  background: var(--pages-neutral-2, #fafafa);
  cursor: pointer;
  font-size: 12px;
  font-family: inherit;
  font-weight: 500;
}
.kind-pill.active {
  background: var(--pages-accent-3, #e0f2fe);
  border-color: var(--pages-accent-7, #0066cc);
}
.kind-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}
.kind-count {
  font-size: 11px;
  color: var(--pages-neutral-9, #737373);
}
.chain-node {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 4px 0;
  font-size: 13px;
}
.chain-node.chain-cycle {
  opacity: 0.5;
  border: 1px dashed var(--pages-neutral-6, #a3a3a3);
  border-radius: 4px;
  padding: 4px;
}
.chain-connector {
  color: var(--pages-neutral-9, #737373);
  cursor: pointer;
  font-size: 14px;
  padding: 2px 4px;
  border-radius: 3px;
  border: none;
  background: none;
  font-family: inherit;
}
.chain-connector:hover {
  background: var(--pages-danger-2, #fef2f2);
  color: var(--pages-danger-9, #dc2626);
}
.cycle-indicator {
  font-size: 14px;
  color: var(--pages-danger-9, #dc2626);
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `npx vitest run src/agent-relationship-editor.test.ts` from `components/agent-relationship-editor/`
Expected: All tests PASS (existing + new chain tab tests)

- [ ] **Step 9: Commit**

```bash
git add components/agent-relationship-editor/src/agent-relationship-editor.ts components/agent-relationship-editor/src/agent-relationship-editor.test.ts
git commit -m "feat(#234): add chain tab with kind picker and tree rendering

Third tab (Table | Chain | Diagram) renders relationship chains as
indented trees. Kind picker pill bar with count badges. Full avatar
pills for agent identity. Cycle detection with visual indicator.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Inline editing operations

### Task 3: Inline delete, insert, and add operations

**Files:**
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.ts`
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.test.ts`

**Interfaces:**
- Consumes: Chain tab rendering from Task 2, `_pendingAdditions` / `_pendingRemovals` / `_effectiveRelationships` from existing editor
- Produces: Complete inline editing UI — delete (reconnect/disconnect), insert between, add to leaf, add new chain root

- [ ] **Step 1: Write failing tests for delete operations**

Add to `agent-relationship-editor.test.ts` inside the `chain tab` describe block:

```typescript
describe('inline delete', () => {
  it('clicking connector shows action bar with three buttons', async () => {
    const connector = el.shadowRoot!.querySelector('.chain-connector') as HTMLElement;
    connector.click();
    await el.updateComplete;

    const actionBar = el.shadowRoot!.querySelector('.chain-action-bar');
    expect(actionBar).toBeTruthy();
    expect(actionBar!.getAttribute('role')).toBe('group');
    expect(actionBar!.getAttribute('aria-label')).toBe('Relationship action');
    const buttons = actionBar!.querySelectorAll('button');
    expect(buttons).toHaveLength(3);
  });

  it('remove-and-disconnect adds one pending removal', async () => {
    const connector = el.shadowRoot!.querySelector('.chain-connector') as HTMLElement;
    connector.click();
    await el.updateComplete;

    const disconnectBtn = el.shadowRoot!.querySelector('.chain-action-bar .btn-disconnect') as HTMLElement;
    disconnectBtn.click();
    await el.updateComplete;

    const removals = el.shadowRoot!.querySelectorAll('.pending-removal');
    expect(removals.length).toBe(1);
  });

  it('remove-and-reconnect adds removal plus additions for grandchildren', async () => {
    // Bob supervises Carol and Dave. Removing Alice->Bob should reconnect Alice->Carol and Alice->Dave.
    const connectors = el.shadowRoot!.querySelectorAll('.chain-connector');
    const bobConnector = connectors[0] as HTMLElement; // first connector is Alice->Bob
    bobConnector.click();
    await el.updateComplete;

    const reconnectBtn = el.shadowRoot!.querySelector('.chain-action-bar .btn-reconnect') as HTMLElement;
    reconnectBtn.click();
    await el.updateComplete;

    // Should have pending removal + pending additions
    const confirmBar = el.shadowRoot!.querySelector('.confirm-bar');
    expect(confirmBar).toBeTruthy();
  });

  it('cancel dismisses action bar', async () => {
    const connector = el.shadowRoot!.querySelector('.chain-connector') as HTMLElement;
    connector.click();
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.chain-action-bar')).toBeTruthy();

    const cancelBtn = el.shadowRoot!.querySelector('.chain-action-bar .btn-action-cancel') as HTMLElement;
    cancelBtn.click();
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.chain-action-bar')).toBeFalsy();
  });
});
```

- [ ] **Step 2: Write failing tests for add-to-leaf and add-new-chain**

```typescript
describe('inline add', () => {
  it('leaf nodes show add-to-leaf button', async () => {
    const addBtns = el.shadowRoot!.querySelectorAll('.chain-add-leaf');
    expect(addBtns.length).toBeGreaterThan(0);
    expect(addBtns[0].getAttribute('role')).toBe('button');
  });

  it('clicking add-to-leaf shows agent picker', async () => {
    const addBtn = el.shadowRoot!.querySelector('.chain-add-leaf') as HTMLElement;
    addBtn.click();
    await el.updateComplete;

    const picker = el.shadowRoot!.querySelector('.chain-agent-picker');
    expect(picker).toBeTruthy();
  });

  it('selecting agent from picker adds pending addition', async () => {
    const addBtn = el.shadowRoot!.querySelector('.chain-add-leaf') as HTMLElement;
    addBtn.click();
    await el.updateComplete;

    const select = el.shadowRoot!.querySelector('.chain-agent-picker select') as HTMLSelectElement;
    select.value = 'eve';
    select.dispatchEvent(new Event('change'));
    await el.updateComplete;

    const confirmBar = el.shadowRoot!.querySelector('.confirm-bar');
    expect(confirmBar).toBeTruthy();
  });

  it('renders new-chain button below forest', async () => {
    const newChainBtn = el.shadowRoot!.querySelector('.chain-new-root');
    expect(newChainBtn).toBeTruthy();
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `npx vitest run src/agent-relationship-editor.test.ts` from `components/agent-relationship-editor/`
Expected: FAIL — no action bar, no add-to-leaf, no agent picker

- [ ] **Step 4: Add state for inline editing**

Add to the editor class:

```typescript
@state() private _chainActionTarget: { node: ChainNode; parentAgentId: string } | null = null;
@state() private _chainPickerTarget: { type: 'leaf' | 'insert' | 'new-source' | 'new-target'; node?: ChainNode; parentAgentId?: string } | null = null;
@state() private _chainNewSourceId = '';
```

- [ ] **Step 5: Implement delete action bar**

Update `_renderChainNode()` to show the action bar when a connector is clicked. Add `_handleChainConnectorClick(node, parentAgentId)` method:

```typescript
private _handleChainConnectorClick(node: ChainNode, parentAgentId: string) {
  this._chainActionTarget = { node, parentAgentId };
  this._chainPickerTarget = null;
}

private _handleReconnect() {
  if (!this._chainActionTarget) return;
  const { node, parentAgentId } = this._chainActionTarget;
  // Mark the clicked relationship for removal
  if (node.relationshipIndex < this.relationships.length) {
    this._pendingRemovals = new Set([...this._pendingRemovals, node.relationshipIndex]);
  } else {
    this._pendingAdditions = this._pendingAdditions.filter((_, i) => i !== node.relationshipIndex - this.relationships.length);
  }
  // Add reconnections: parent -> each grandchild
  const grandchildren = node.children.filter(c => !c.isCycle);
  const newRels: AgentRelationship[] = grandchildren.map(gc => ({
    sourceAgentId: parentAgentId,
    targetAgentId: gc.agentId,
    kind: this._chainKind,
    tenancyId: this.tenancyId,
  }));
  this._pendingAdditions = [...this._pendingAdditions, ...newRels];
  this._chainActionTarget = null;
}

private _handleDisconnect() {
  if (!this._chainActionTarget) return;
  const { node } = this._chainActionTarget;
  if (node.relationshipIndex < this.relationships.length) {
    this._pendingRemovals = new Set([...this._pendingRemovals, node.relationshipIndex]);
  } else {
    this._pendingAdditions = this._pendingAdditions.filter((_, i) => i !== node.relationshipIndex - this.relationships.length);
  }
  this._chainActionTarget = null;
}
```

Update `_renderChainNode()` to pass parentAgentId down and render the action bar when `_chainActionTarget` matches:

```typescript
private _renderChainNode(node: ChainNode, level: number, parentAgentId?: string) {
  const hasChildren = node.children.length > 0 && !node.isCycle;
  const isLeaf = node.children.length === 0 && !node.isCycle;
  const isActionTarget = this._chainActionTarget?.node.relationshipIndex === node.relationshipIndex && node.relationshipIndex >= 0;

  return html`
    <div role="treeitem" aria-level=${level}
      aria-expanded=${hasChildren ? 'true' : nothing}
      class="chain-node ${node.isPending ? 'pending-addition' : ''} ${node.isCycle ? 'chain-cycle' : ''}"
      style="padding-left: ${(level - 1) * 24}px">
      ${node.relationshipIndex >= 0 ? html`
        <span class="chain-connector" role="button"
          aria-label="Remove relationship from ${this._getAgentName(parentAgentId ?? '')} to ${this._getAgentName(node.agentId)}"
          @click=${() => this._handleChainConnectorClick(node, parentAgentId ?? '')}>→</span>
      ` : nothing}
      ${this._renderAgentCell(node.agentId)}
      ${node.isCycle ? html`<span class="cycle-indicator" aria-label="Cycle detected — ${this._getAgentName(node.agentId)} already appears earlier in this chain">↻</span>` : nothing}
      ${isLeaf ? html`
        <button class="chain-add-leaf" role="button"
          aria-label="Add relationship from ${this._getAgentName(node.agentId)}"
          @click=${() => { this._chainPickerTarget = { type: 'leaf', node }; this._chainActionTarget = null; }}>+ add</button>
      ` : nothing}
    </div>
    ${isActionTarget ? html`
      <div class="chain-action-bar" role="group" aria-label="Relationship action"
        style="padding-left: ${(level - 1) * 24 + 20}px">
        <button class="btn-reconnect" @click=${() => this._handleReconnect()}>Reconnect</button>
        <button class="btn-disconnect" @click=${() => this._handleDisconnect()}>Disconnect</button>
        <button class="btn-action-cancel" @click=${() => { this._chainActionTarget = null; }}>Cancel</button>
      </div>
    ` : nothing}
    ${this._chainPickerTarget?.type === 'leaf' && this._chainPickerTarget.node?.agentId === node.agentId ? html`
      <div class="chain-agent-picker" style="padding-left: ${(level - 1) * 24 + 20}px">
        <select @change=${(e: Event) => this._handleLeafAdd(node.agentId, (e.target as HTMLSelectElement).value)}>
          <option value="">Select agent...</option>
          ${this.roster.filter(r => r.agentId !== node.agentId).map(r => html`<option value=${r.agentId}>${r.name}</option>`)}
        </select>
      </div>
    ` : nothing}
    ${hasChildren ? node.children.map(child => this._renderChainNode(child, level + 1, node.agentId)) : nothing}
  `;
}
```

- [ ] **Step 6: Implement add-to-leaf and add-new-chain**

```typescript
private _handleLeafAdd(fromAgentId: string, toAgentId: string) {
  if (!toAgentId) return;
  const newRel: AgentRelationship = {
    sourceAgentId: fromAgentId,
    targetAgentId: toAgentId,
    kind: this._chainKind,
    tenancyId: this.tenancyId,
  };
  this._pendingAdditions = [...this._pendingAdditions, newRel];
  this._chainPickerTarget = null;
}
```

Add the `[+ new chain]` button in `_renderChainTab()` after the tree:

```typescript
<button class="chain-new-root add-relationship"
  @click=${() => { this._chainPickerTarget = { type: 'new-source' }; this._chainActionTarget = null; }}>
  + New Chain
</button>
${this._chainPickerTarget?.type === 'new-source' ? html`
  <div class="chain-agent-picker">
    <select @change=${(e: Event) => { this._chainNewSourceId = (e.target as HTMLSelectElement).value; if (this._chainNewSourceId) this._chainPickerTarget = { type: 'new-target' }; }}>
      <option value="">Source agent...</option>
      ${this.roster.map(r => html`<option value=${r.agentId}>${r.name}</option>`)}
    </select>
  </div>
` : nothing}
${this._chainPickerTarget?.type === 'new-target' ? html`
  <div class="chain-agent-picker">
    <select @change=${(e: Event) => this._handleLeafAdd(this._chainNewSourceId, (e.target as HTMLSelectElement).value)}>
      <option value="">Target agent...</option>
      ${this.roster.filter(r => r.agentId !== this._chainNewSourceId).map(r => html`<option value=${r.agentId}>${r.name}</option>`)}
    </select>
  </div>
` : nothing}
```

- [ ] **Step 7: Add CSS styles for action bar and picker**

```css
.chain-action-bar {
  display: flex;
  gap: 4px;
  padding: 4px 0;
  font-size: 12px;
}
.chain-action-bar button {
  padding: 3px 8px;
  border: 1px solid var(--pages-neutral-4, #e5e5e5);
  border-radius: 4px;
  font-size: 12px;
  cursor: pointer;
  font-family: inherit;
  background: var(--pages-neutral-1, #fff);
}
.btn-reconnect { color: var(--pages-accent-9, #0066cc); }
.btn-disconnect { color: var(--pages-danger-9, #dc2626); }
.chain-add-leaf {
  padding: 2px 8px;
  border: 1px dashed var(--pages-neutral-6, #a3a3a3);
  border-radius: 4px;
  cursor: pointer;
  font-size: 11px;
  color: var(--pages-neutral-9, #737373);
  background: none;
  font-family: inherit;
}
.chain-add-leaf:hover {
  border-color: var(--pages-accent-7, #0066cc);
  color: var(--pages-accent-9, #0066cc);
}
.chain-agent-picker {
  padding: 4px 0;
}
.chain-agent-picker select {
  padding: 6px 8px;
  border: 1px solid var(--pages-neutral-4, #e5e5e5);
  border-radius: 4px;
  font-size: 13px;
  font-family: inherit;
  background: var(--pages-neutral-1, #fff);
  color: var(--pages-neutral-12, #111);
}
.chain-new-root {
  margin-top: var(--pages-space-2, 0.5rem);
}
```

- [ ] **Step 8: Run all tests**

Run: `npx vitest run` from `components/agent-relationship-editor/`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git add components/agent-relationship-editor/src/agent-relationship-editor.ts components/agent-relationship-editor/src/agent-relationship-editor.test.ts
git commit -m "feat(#234): add inline delete, insert, and add operations to chain tab

Delete via connector click with reconnect/disconnect/cancel action bar.
Add-to-leaf via inline agent picker on leaf nodes. New chain root via
two-step source/target picker. All edits use shared pending changeset.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Demo update and insert-between

### Task 4: Insert-between and demo update

**Files:**
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.ts`
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.test.ts`
- Modify: `components/agent-relationship-editor/demo.html`

**Interfaces:**
- Consumes: Chain tab and editing from Tasks 2-3
- Produces: Complete feature — insert-between hover UI, updated demo

- [ ] **Step 1: Write failing test for insert-between**

Add to `agent-relationship-editor.test.ts` chain tab describe:

```typescript
describe('insert between', () => {
  it('connector shows insert button on hover state', async () => {
    // Insert buttons are always rendered but visually hidden until hover.
    // We test they exist in DOM.
    const insertBtns = el.shadowRoot!.querySelectorAll('.chain-insert-btn');
    expect(insertBtns.length).toBeGreaterThan(0);
    expect(insertBtns[0].getAttribute('role')).toBe('button');
    expect(insertBtns[0].getAttribute('aria-label')).toContain('Insert agent between');
  });

  it('clicking insert button shows agent picker', async () => {
    const insertBtn = el.shadowRoot!.querySelector('.chain-insert-btn') as HTMLElement;
    insertBtn.click();
    await el.updateComplete;

    const picker = el.shadowRoot!.querySelector('.chain-agent-picker');
    expect(picker).toBeTruthy();
  });

  it('selecting agent from insert picker produces correct pending ops', async () => {
    const insertBtn = el.shadowRoot!.querySelector('.chain-insert-btn') as HTMLElement;
    insertBtn.click();
    await el.updateComplete;

    const select = el.shadowRoot!.querySelector('.chain-agent-picker select') as HTMLSelectElement;
    select.value = 'eve';
    select.dispatchEvent(new Event('change'));
    await el.updateComplete;

    // Should have confirm bar (pending changes exist)
    const confirmBar = el.shadowRoot!.querySelector('.confirm-bar');
    expect(confirmBar).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/agent-relationship-editor.test.ts` from `components/agent-relationship-editor/`
Expected: FAIL — no `.chain-insert-btn` exists

- [ ] **Step 3: Implement insert-between**

Add an insert button alongside each connector in `_renderChainNode()`. When clicked, it shows an agent picker. When an agent is selected, it removes the old edge and adds two new ones (parent → inserted, inserted → child).

```typescript
private _handleInsertBetween(parentAgentId: string, childNode: ChainNode, insertedAgentId: string) {
  if (!insertedAgentId) return;
  // Remove old edge
  if (childNode.relationshipIndex < this.relationships.length) {
    this._pendingRemovals = new Set([...this._pendingRemovals, childNode.relationshipIndex]);
  } else {
    this._pendingAdditions = this._pendingAdditions.filter((_, i) => i !== childNode.relationshipIndex - this.relationships.length);
  }
  // Add parent -> inserted and inserted -> child
  const rel1: AgentRelationship = {
    sourceAgentId: parentAgentId,
    targetAgentId: insertedAgentId,
    kind: this._chainKind,
    tenancyId: this.tenancyId,
  };
  const rel2: AgentRelationship = {
    sourceAgentId: insertedAgentId,
    targetAgentId: childNode.agentId,
    kind: this._chainKind,
    tenancyId: this.tenancyId,
  };
  this._pendingAdditions = [...this._pendingAdditions, rel1, rel2];
  this._chainPickerTarget = null;
}
```

Add the insert button and picker in `_renderChainNode()` next to the connector arrow:

```typescript
${node.relationshipIndex >= 0 ? html`
  <button class="chain-insert-btn" role="button"
    aria-label="Insert agent between ${this._getAgentName(parentAgentId ?? '')} and ${this._getAgentName(node.agentId)}"
    @click=${() => { this._chainPickerTarget = { type: 'insert', node, parentAgentId }; this._chainActionTarget = null; }}>+</button>
` : nothing}
```

Add the insert picker rendering below the node:

```typescript
${this._chainPickerTarget?.type === 'insert' && this._chainPickerTarget.node?.relationshipIndex === node.relationshipIndex ? html`
  <div class="chain-agent-picker" style="padding-left: ${(level - 1) * 24 + 20}px">
    <select @change=${(e: Event) => this._handleInsertBetween(parentAgentId ?? '', node, (e.target as HTMLSelectElement).value)}>
      <option value="">Insert agent...</option>
      ${this.roster.filter(r => r.agentId !== node.agentId && r.agentId !== (parentAgentId ?? '')).map(r => html`<option value=${r.agentId}>${r.name}</option>`)}
    </select>
  </div>
` : nothing}
```

- [ ] **Step 4: Add CSS for insert button**

```css
.chain-insert-btn {
  width: 18px;
  height: 18px;
  border: 1px dashed var(--pages-neutral-5, #4a4a62);
  border-radius: 50%;
  background: none;
  cursor: pointer;
  font-size: 12px;
  line-height: 1;
  color: var(--pages-neutral-9, #737373);
  padding: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transition: opacity 0.15s;
  font-family: inherit;
}
.chain-node:hover .chain-insert-btn {
  opacity: 1;
}
.chain-insert-btn:hover {
  border-color: var(--pages-accent-7, #0066cc);
  color: var(--pages-accent-9, #0066cc);
  opacity: 1;
}
```

- [ ] **Step 5: Run all tests**

Run: `npx vitest run` from `components/agent-relationship-editor/`
Expected: All tests PASS

- [ ] **Step 6: Update demo.html with chain tab**

No code changes needed to `demo.html` — the chain tab appears automatically because the component now renders three tabs. But verify the demo works by checking the tab renders with the Gastown Org data (6 SUPERVISES relationships form a clear tree: Boot → Deacon → Witness Alpha/Beta → Polecats).

- [ ] **Step 7: Run typecheck**

Run: `npx tsc --noEmit` from `components/agent-relationship-editor/`
Expected: No type errors

- [ ] **Step 8: Commit**

```bash
git add components/agent-relationship-editor/src/agent-relationship-editor.ts components/agent-relationship-editor/src/agent-relationship-editor.test.ts
git commit -m "feat(#234): add insert-between operation and complete chain navigator

Insert button appears on hover between connected agents. Selecting an
agent replaces the old edge with two new ones. Demo works with Gastown
Org data showing SUPERVISES tree.

Closes #234

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-10-15-chain-navigator-design.md] — design spec this plan implements
- `components/agent-relationship-editor/src/agent-relationship-editor.ts` — existing editor (456 lines)
- `components/agent-relationship-editor/src/types.ts` — AgentRosterEntry, RelationshipChangeset
- `packages/graph-stencil-org/src/types.ts:40-57` — AgentRelationship, RelationshipKind
- `packages/blocks-ui-core/src/types/relationship.ts` — lookupRelationshipType, RelationshipTypeDescriptor
- `components/agent-relationship-editor/src/agent-relationship-editor.test.ts` — existing test patterns
- `components/agent-relationship-editor/demo.html` — Gastown Org demo data
- `docs/protocols/blocks-ui/custom-event-namespace-prefix.md` — event naming
- GitHub #234 — focal issue
- GitHub #233 — relationship editor (landed)
