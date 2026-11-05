# Chain Navigator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #234 — interactive chain navigator with inline editing
**Issue group:** #234

**Goal:** Add a Chain tab to the relationship editor with indented tree
rendering, inline CRUD operations, and cycle detection — all backed by
a shared identity-based changeset model.

**Architecture:** Three layers: (1) pure chain traversal utility in
`graph-stencil-org` with DAG-aware cycle detection; (2) `chain-view`
LitElement sub-component for tree rendering and inline editing; (3)
editor orchestrator migration from index-based to identity-based
changeset tracking with compound operation support.

**Tech Stack:** LitElement, TypeScript, Vitest, `@casehubio/mosaic-graph-stencil-org`

## Global Constraints

- ARIA required on every `@customElement` (role, aria-label, state attrs)
- Events: `bubbles: true` only (not `composed: true`) for internal
  chain-view ↔ editor communication
- BACKS_UP excluded from chain navigation
- EXTENDED sub-grouped by `extendedKind`
- Metadata propagation: scope inherited on insert/reconnect, attestation
  NOT inherited
- All types in `graph-stencil-org/src/types.ts`; operation types in
  `agent-relationship-editor/src/types.ts`

---

## Batch 1: Chain Traversal Utility

### Task 1: ChainNode type and buildChainTree + findChainRoots

**Files:**
- Modify: `packages/graph-stencil-org/src/types.ts` — add `ChainNode`
- Create: `packages/graph-stencil-org/src/adapter/chain-tree.ts` — traversal functions
- Create: `packages/graph-stencil-org/src/adapter/chain-tree.test.ts` — unit tests
- Modify: `packages/graph-stencil-org/src/index.ts` — export new functions and type

**Interfaces:**
- Consumes: `AgentRelationship`, `RelationshipKind` from `types.ts`
- Produces:
  - `ChainNode { agentId: string; children: ChainNode[]; isCycle: boolean; isConvergent: boolean }`
  - `buildChainTree(relationships: AgentRelationship[], rootAgentId: string, kind: RelationshipKind, extendedKind?: string): ChainNode`
  - `findChainRoots(relationships: AgentRelationship[], agentId: string, kind: RelationshipKind, extendedKind?: string): string[]`

- [ ] **Step 1: Add ChainNode type**

In `packages/graph-stencil-org/src/types.ts`, add at the end:

```ts
export interface ChainNode {
  agentId: string;
  children: ChainNode[];
  isCycle: boolean;
  isConvergent: boolean;
}
```

- [ ] **Step 2: Write failing tests for buildChainTree**

Create `packages/graph-stencil-org/src/adapter/chain-tree.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { buildChainTree, findChainRoots } from './chain-tree.js';
import type { AgentRelationship } from '../types.js';

function rel(source: string, target: string, kind = 'SUPERVISES' as const, extra?: Partial<AgentRelationship>): AgentRelationship {
  return { sourceAgentId: source, targetAgentId: target, kind, tenancyId: 't1', ...extra };
}

describe('buildChainTree', () => {
  it('builds a linear chain', () => {
    const rels = [rel('A', 'B'), rel('B', 'C'), rel('C', 'D')];
    const tree = buildChainTree(rels, 'A', 'SUPERVISES');
    expect(tree.agentId).toBe('A');
    expect(tree.children).toHaveLength(1);
    expect(tree.children[0].agentId).toBe('B');
    expect(tree.children[0].children[0].agentId).toBe('C');
    expect(tree.children[0].children[0].children[0].agentId).toBe('D');
    expect(tree.children[0].children[0].children[0].children).toHaveLength(0);
  });

  it('builds a branching tree', () => {
    const rels = [rel('A', 'B'), rel('A', 'C'), rel('B', 'D'), rel('B', 'E')];
    const tree = buildChainTree(rels, 'A', 'SUPERVISES');
    expect(tree.children).toHaveLength(2);
    const childB = tree.children.find(c => c.agentId === 'B')!;
    expect(childB.children).toHaveLength(2);
    expect(childB.children.map(c => c.agentId).sort()).toEqual(['D', 'E']);
  });

  it('detects cycles via ancestor path', () => {
    const rels = [rel('A', 'B'), rel('B', 'C'), rel('C', 'A')];
    const tree = buildChainTree(rels, 'A', 'SUPERVISES');
    const nodeC = tree.children[0].children[0];
    expect(nodeC.agentId).toBe('C');
    expect(nodeC.children).toHaveLength(1);
    const cycleNode = nodeC.children[0];
    expect(cycleNode.agentId).toBe('A');
    expect(cycleNode.isCycle).toBe(true);
    expect(cycleNode.children).toHaveLength(0);
  });

  it('handles DAG convergence without marking as cycle', () => {
    const rels = [rel('A', 'B'), rel('A', 'C'), rel('B', 'D'), rel('C', 'D')];
    const tree = buildChainTree(rels, 'A', 'SUPERVISES');
    const childB = tree.children.find(c => c.agentId === 'B')!;
    const childC = tree.children.find(c => c.agentId === 'C')!;
    const dUnderB = childB.children.find(c => c.agentId === 'D')!;
    const dUnderC = childC.children.find(c => c.agentId === 'D')!;
    expect(dUnderB.isCycle).toBe(false);
    expect(dUnderC.isCycle).toBe(false);
    expect(dUnderB.isConvergent || dUnderC.isConvergent).toBe(true);
  });

  it('distinguishes convergence from cycle', () => {
    const rels = [rel('A', 'B'), rel('A', 'C'), rel('B', 'D'), rel('C', 'D'), rel('D', 'A')];
    const tree = buildChainTree(rels, 'A', 'SUPERVISES');
    const childB = tree.children.find(c => c.agentId === 'B')!;
    const dUnderB = childB.children.find(c => c.agentId === 'D')!;
    expect(dUnderB.isCycle).toBe(false);
    const cycleA = dUnderB.children.find(c => c.agentId === 'A');
    expect(cycleA).toBeTruthy();
    expect(cycleA!.isCycle).toBe(true);
  });

  it('filters by kind', () => {
    const rels = [rel('A', 'B', 'SUPERVISES'), rel('A', 'C', 'DELEGATES_TO')];
    const tree = buildChainTree(rels, 'A', 'SUPERVISES');
    expect(tree.children).toHaveLength(1);
    expect(tree.children[0].agentId).toBe('B');
  });

  it('filters EXTENDED by extendedKind', () => {
    const rels = [
      rel('A', 'B', 'EXTENDED', { extendedKind: 'mentors' }),
      rel('A', 'C', 'EXTENDED', { extendedKind: 'reviews' }),
      rel('B', 'D', 'EXTENDED', { extendedKind: 'mentors' }),
    ];
    const tree = buildChainTree(rels, 'A', 'EXTENDED', 'mentors');
    expect(tree.children).toHaveLength(1);
    expect(tree.children[0].agentId).toBe('B');
    expect(tree.children[0].children[0].agentId).toBe('D');
  });

  it('returns leaf node for agent with no outgoing edges', () => {
    const rels = [rel('A', 'B')];
    const tree = buildChainTree(rels, 'B', 'SUPERVISES');
    expect(tree.agentId).toBe('B');
    expect(tree.children).toHaveLength(0);
    expect(tree.isCycle).toBe(false);
  });

  it('returns isolated root for agent not in any relationship', () => {
    const tree = buildChainTree([], 'X', 'SUPERVISES');
    expect(tree.agentId).toBe('X');
    expect(tree.children).toHaveLength(0);
  });
});

describe('findChainRoots', () => {
  it('finds root of a linear chain from mid-chain agent', () => {
    const rels = [rel('A', 'B'), rel('B', 'C'), rel('C', 'D')];
    const roots = findChainRoots(rels, 'C', 'SUPERVISES');
    expect(roots).toEqual(['A']);
  });

  it('returns agent itself when it is the root', () => {
    const rels = [rel('A', 'B'), rel('B', 'C')];
    const roots = findChainRoots(rels, 'A', 'SUPERVISES');
    expect(roots).toEqual(['A']);
  });

  it('returns multiple roots for agent in multiple chains', () => {
    const rels = [rel('R1', 'X'), rel('R2', 'X')];
    const roots = findChainRoots(rels, 'X', 'SUPERVISES');
    expect(roots.sort()).toEqual(['R1', 'R2']);
  });

  it('handles fully cyclic chain by returning focal agent', () => {
    const rels = [rel('A', 'B'), rel('B', 'C'), rel('C', 'A')];
    const roots = findChainRoots(rels, 'B', 'SUPERVISES');
    expect(roots).toEqual(['B']);
  });

  it('filters by kind', () => {
    const rels = [rel('R1', 'X', 'SUPERVISES'), rel('R2', 'X', 'DELEGATES_TO')];
    const roots = findChainRoots(rels, 'X', 'SUPERVISES');
    expect(roots).toEqual(['R1']);
  });

  it('filters EXTENDED by extendedKind', () => {
    const rels = [
      rel('R1', 'X', 'EXTENDED', { extendedKind: 'mentors' }),
      rel('R2', 'X', 'EXTENDED', { extendedKind: 'reviews' }),
    ];
    const roots = findChainRoots(rels, 'X', 'EXTENDED', 'mentors');
    expect(roots).toEqual(['R1']);
  });

  it('returns agent itself when no relationships of that kind exist', () => {
    const roots = findChainRoots([], 'X', 'SUPERVISES');
    expect(roots).toEqual(['X']);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/mosaic-graph-stencil-org run test -- --run chain-tree`
Expected: FAIL — module `./chain-tree.js` not found

- [ ] **Step 4: Implement buildChainTree and findChainRoots**

Create `packages/graph-stencil-org/src/adapter/chain-tree.ts`:

```ts
import type { AgentRelationship, RelationshipKind, ChainNode } from '../types.js';

function matchesKind(rel: AgentRelationship, kind: RelationshipKind, extendedKind?: string): boolean {
  if (rel.kind !== kind) return false;
  if (kind === 'EXTENDED' && extendedKind !== undefined) {
    return rel.extendedKind === extendedKind;
  }
  return true;
}

export function buildChainTree(
  relationships: AgentRelationship[],
  rootAgentId: string,
  kind: RelationshipKind,
  extendedKind?: string,
): ChainNode {
  const seen = new Set<string>();

  function walk(agentId: string, ancestorPath: Set<string>): ChainNode {
    if (ancestorPath.has(agentId)) {
      return { agentId, children: [], isCycle: true, isConvergent: false };
    }

    const isConvergent = seen.has(agentId);
    seen.add(agentId);

    const nextPath = new Set(ancestorPath);
    nextPath.add(agentId);

    const outgoing = relationships.filter(
      r => r.sourceAgentId === agentId && matchesKind(r, kind, extendedKind),
    );

    const children = outgoing.map(r => walk(r.targetAgentId, nextPath));
    return { agentId, children, isCycle: false, isConvergent };
  }

  return walk(rootAgentId, new Set());
}

export function findChainRoots(
  relationships: AgentRelationship[],
  agentId: string,
  kind: RelationshipKind,
  extendedKind?: string,
): string[] {
  const matching = relationships.filter(r => matchesKind(r, kind, extendedKind));

  function walkUp(current: string, visited: Set<string>): string[] {
    const parents = matching
      .filter(r => r.targetAgentId === current)
      .map(r => r.sourceAgentId);

    if (parents.length === 0) return [current];

    const roots: string[] = [];
    for (const parent of parents) {
      if (visited.has(parent)) {
        return [agentId];
      }
      const nextVisited = new Set(visited);
      nextVisited.add(parent);
      roots.push(...walkUp(parent, nextVisited));
    }
    return [...new Set(roots)];
  }

  return walkUp(agentId, new Set([agentId]));
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/mosaic-graph-stencil-org run test -- --run chain-tree`
Expected: all tests PASS

- [ ] **Step 6: Export from package index**

In `packages/graph-stencil-org/src/index.ts`, add after the `extractEgoSubgraph` export:

```ts
export { buildChainTree, findChainRoots } from './adapter/chain-tree.js';
export type { ChainNode } from './types.js';
```

- [ ] **Step 7: Run full package tests and typecheck**

Run: `yarn workspace @casehubio/mosaic-graph-stencil-org run test -- --run`
Run: `yarn typecheck`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git add packages/graph-stencil-org/src/types.ts packages/graph-stencil-org/src/adapter/chain-tree.ts packages/graph-stencil-org/src/adapter/chain-tree.test.ts packages/graph-stencil-org/src/index.ts
git commit -m "feat(#234): add buildChainTree and findChainRoots with DAG-aware cycle detection

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Types and Editor Changeset Migration

### Task 2: RelationshipIdentity, ChainOperation, and identityKey

**Files:**
- Modify: `components/agent-relationship-editor/src/types.ts` — add new types
- Create: `components/agent-relationship-editor/src/identity.ts` — identity helper
- Create: `components/agent-relationship-editor/src/identity.test.ts` — helper tests

**Interfaces:**
- Consumes: `AgentRelationship`, `RelationshipKind`, `RelationshipScope` from `graph-stencil-org`
- Produces:
  - `RelationshipIdentity { sourceAgentId, targetAgentId, kind, extendedKind?, scope? }`
  - `ChainOperation { type, removals: RelationshipIdentity[], additions: AgentRelationship[], description }`
  - `identityKey(id: RelationshipIdentity): string`
  - `identityOf(rel: AgentRelationship): RelationshipIdentity`
  - `identityMatches(rel: AgentRelationship, id: RelationshipIdentity): boolean`
  - `flattenOperations(ops: ChainOperation[], confirmed: AgentRelationship[]): RelationshipChangeset`

- [ ] **Step 1: Write failing tests for identity helpers**

Create `components/agent-relationship-editor/src/identity.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { identityKey, identityOf, identityMatches, flattenOperations } from './identity.js';
import type { AgentRelationship } from '@casehubio/mosaic-graph-stencil-org';
import type { ChainOperation } from './types.js';

function rel(source: string, target: string, kind = 'SUPERVISES' as const, extra?: Partial<AgentRelationship>): AgentRelationship {
  return { sourceAgentId: source, targetAgentId: target, kind, tenancyId: 't1', ...extra };
}

describe('identityKey', () => {
  it('produces same key for same identity', () => {
    const id = identityOf(rel('A', 'B'));
    expect(identityKey(id)).toBe(identityKey(identityOf(rel('A', 'B'))));
  });

  it('produces different key for different targets', () => {
    expect(identityKey(identityOf(rel('A', 'B')))).not.toBe(identityKey(identityOf(rel('A', 'C'))));
  });

  it('distinguishes by extendedKind', () => {
    const a = identityOf(rel('A', 'B', 'EXTENDED', { extendedKind: 'mentors' }));
    const b = identityOf(rel('A', 'B', 'EXTENDED', { extendedKind: 'reviews' }));
    expect(identityKey(a)).not.toBe(identityKey(b));
  });

  it('distinguishes by scope domain', () => {
    const a = identityOf(rel('A', 'B', 'DELEGATES_TO', { scope: { domain: 'billing' } }));
    const b = identityOf(rel('A', 'B', 'DELEGATES_TO', { scope: { domain: 'shipping' } }));
    expect(identityKey(a)).not.toBe(identityKey(b));
  });
});

describe('identityMatches', () => {
  it('matches by content, not reference', () => {
    const r = rel('A', 'B');
    const id = identityOf(rel('A', 'B'));
    expect(identityMatches(r, id)).toBe(true);
  });

  it('does not match different kind', () => {
    const r = rel('A', 'B', 'SUPERVISES');
    const id = identityOf(rel('A', 'B', 'DELEGATES_TO'));
    expect(identityMatches(r, id)).toBe(false);
  });
});

describe('flattenOperations', () => {
  it('collects simple additions and removals', () => {
    const ops: ChainOperation[] = [
      { type: 'delete', removals: [identityOf(rel('A', 'B'))], additions: [], description: '' },
      { type: 'add-leaf', removals: [], additions: [rel('A', 'C')], description: '' },
    ];
    const cs = flattenOperations(ops, [rel('A', 'B')]);
    expect(cs.removals).toHaveLength(1);
    expect(cs.removals[0].targetAgentId).toBe('B');
    expect(cs.additions).toHaveLength(1);
    expect(cs.additions[0].targetAgentId).toBe('C');
  });

  it('cancels add-then-remove (net zero)', () => {
    const ops: ChainOperation[] = [
      { type: 'add-leaf', removals: [], additions: [rel('A', 'C')], description: '' },
      { type: 'delete', removals: [identityOf(rel('A', 'C'))], additions: [], description: '' },
    ];
    const cs = flattenOperations(ops, []);
    expect(cs.additions).toHaveLength(0);
    expect(cs.removals).toHaveLength(0);
  });

  it('cancels remove-then-re-add (net zero)', () => {
    const confirmed = [rel('A', 'B')];
    const ops: ChainOperation[] = [
      { type: 'delete', removals: [identityOf(rel('A', 'B'))], additions: [], description: '' },
      { type: 'add-leaf', removals: [], additions: [rel('A', 'B')], description: '' },
    ];
    const cs = flattenOperations(ops, confirmed);
    expect(cs.additions).toHaveLength(0);
    expect(cs.removals).toHaveLength(0);
  });

  it('deduplicates duplicate removals', () => {
    const ops: ChainOperation[] = [
      { type: 'delete', removals: [identityOf(rel('A', 'B'))], additions: [], description: '' },
      { type: 'delete', removals: [identityOf(rel('A', 'B'))], additions: [], description: '' },
    ];
    const cs = flattenOperations(ops, [rel('A', 'B')]);
    expect(cs.removals).toHaveLength(1);
  });

  it('suppresses additions that duplicate confirmed relationships', () => {
    const confirmed = [rel('A', 'C')];
    const ops: ChainOperation[] = [
      { type: 'remove-reconnect', removals: [identityOf(rel('A', 'B')), identityOf(rel('B', 'C'))], additions: [rel('A', 'C')], description: '' },
    ];
    const cs = flattenOperations(ops, confirmed);
    expect(cs.removals).toHaveLength(2);
    expect(cs.additions).toHaveLength(0);
  });

  it('handles insert-between compound operation', () => {
    const confirmed = [rel('A', 'B')];
    const ops: ChainOperation[] = [
      { type: 'insert-between', removals: [identityOf(rel('A', 'B'))], additions: [rel('A', 'C'), rel('C', 'B')], description: '' },
    ];
    const cs = flattenOperations(ops, confirmed);
    expect(cs.removals).toHaveLength(1);
    expect(cs.additions).toHaveLength(2);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/mosaic-agent-relationship-editor run test -- --run identity`
Expected: FAIL — module not found

- [ ] **Step 3: Add types to types.ts**

In `components/agent-relationship-editor/src/types.ts`, add:

```ts
import type { AgentRelationship, RelationshipKind, RelationshipScope } from '@casehubio/mosaic-graph-stencil-org';

export interface RelationshipIdentity {
  sourceAgentId: string;
  targetAgentId: string;
  kind: RelationshipKind;
  extendedKind?: string;
  scope?: RelationshipScope;
}

export interface ChainOperation {
  type: 'add-leaf' | 'add-root' | 'insert-between'
      | 'remove-disconnect' | 'remove-reconnect' | 'delete';
  removals: RelationshipIdentity[];
  additions: AgentRelationship[];
  description: string;
}
```

- [ ] **Step 4: Implement identity helpers**

Create `components/agent-relationship-editor/src/identity.ts`:

```ts
import type { AgentRelationship } from '@casehubio/mosaic-graph-stencil-org';
import type { RelationshipIdentity, RelationshipChangeset, ChainOperation } from './types.js';

export function identityOf(rel: AgentRelationship): RelationshipIdentity {
  const id: RelationshipIdentity = {
    sourceAgentId: rel.sourceAgentId,
    targetAgentId: rel.targetAgentId,
    kind: rel.kind,
  };
  if (rel.extendedKind) id.extendedKind = rel.extendedKind;
  if (rel.scope) id.scope = rel.scope;
  return id;
}

export function identityKey(id: RelationshipIdentity): string {
  const parts = [id.sourceAgentId, id.targetAgentId, id.kind];
  if (id.extendedKind) parts.push(id.extendedKind);
  if (id.scope) parts.push(JSON.stringify(id.scope));
  return parts.join('::');
}

export function identityMatches(rel: AgentRelationship, id: RelationshipIdentity): boolean {
  return identityKey(identityOf(rel)) === identityKey(id);
}

export function flattenOperations(ops: ChainOperation[], confirmed: AgentRelationship[]): RelationshipChangeset {
  const removalMap = new Map<string, AgentRelationship>();
  const additionMap = new Map<string, AgentRelationship>();

  for (const op of ops) {
    for (const removal of op.removals) {
      const key = identityKey(removal);
      if (additionMap.has(key)) {
        additionMap.delete(key);
      } else {
        const match = confirmed.find(r => identityMatches(r, removal));
        if (match) removalMap.set(key, match);
      }
    }
    for (const addition of op.additions) {
      const key = identityKey(identityOf(addition));
      if (removalMap.has(key)) {
        removalMap.delete(key);
      } else {
        additionMap.set(key, addition);
      }
    }
  }

  const confirmedKeys = new Set(
    confirmed
      .filter(r => !removalMap.has(identityKey(identityOf(r))))
      .map(r => identityKey(identityOf(r))),
  );
  for (const [key] of additionMap) {
    if (confirmedKeys.has(key)) additionMap.delete(key);
  }

  return {
    additions: [...additionMap.values()],
    removals: [...removalMap.values()],
  };
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/mosaic-agent-relationship-editor run test -- --run identity`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add components/agent-relationship-editor/src/types.ts components/agent-relationship-editor/src/identity.ts components/agent-relationship-editor/src/identity.test.ts
git commit -m "feat(#234): add RelationshipIdentity, ChainOperation, and flattenOperations

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: Migrate editor from index-based to identity-based changeset

**Files:**
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.ts` — replace `_pendingRemovals`/`_pendingAdditions` with `_pendingOperations`
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.test.ts` — update tests
- Modify: `components/agent-relationship-editor/src/index.ts` — export new types

**Interfaces:**
- Consumes: `identityOf`, `identityMatches`, `flattenOperations`, `ChainOperation`, `RelationshipIdentity` from Task 2
- Produces: Editor with `_pendingOperations: ChainOperation[]` state, same external `relationship:changed` event

- [ ] **Step 1: Write a failing test for identity-based removal**

In `agent-relationship-editor.test.ts`, add a new test:

```ts
it('marks removal by identity, not index', async () => {
  await el.updateComplete;
  const rows = el.shadowRoot!.querySelectorAll('.relationship-row');
  const removeBtn = rows[0].querySelector('.btn-delete') as HTMLElement;
  removeBtn.click();
  await el.updateComplete;
  const pending = el.shadowRoot!.querySelectorAll('.pending-removal');
  expect(pending).toHaveLength(1);
  expect(pending[0].querySelector('.agent-name')!.textContent).toContain('Alice');
});
```

- [ ] **Step 2: Run test to verify current behavior (should pass with existing code)**

Run: `yarn workspace @casehubio/mosaic-agent-relationship-editor run test -- --run agent-relationship-editor`
Expected: PASS (existing index-based removal still works for this test)

- [ ] **Step 3: Refactor editor internals to use `_pendingOperations`**

In `agent-relationship-editor.ts`:

1. Remove `_pendingAdditions: AgentRelationship[]` and `_pendingRemovals: Set<number>` state properties
2. Add `@state() private _pendingOperations: ChainOperation[] = [];`
3. Add import: `import { identityOf, identityMatches, flattenOperations } from './identity.js';`
4. Add import: `import type { ChainOperation } from './types.js';`
5. Replace `_markForRemoval(index)`:
```ts
private _markForRemoval(index: number) {
  const rel = this.relationships[index];
  if (!rel) return;
  const op: ChainOperation = {
    type: 'delete',
    removals: [identityOf(rel)],
    additions: [],
    description: `Remove ${rel.sourceAgentId} → ${rel.targetAgentId}`,
  };
  this._pendingOperations = [...this._pendingOperations, op];
}
```
6. Replace `_effectiveRelationships` getter to use flattened operations:
```ts
private get _effectiveRelationships(): AgentRelationship[] {
  const cs = flattenOperations(this._pendingOperations, this.relationships);
  const confirmed = this.relationships.filter(r =>
    !cs.removals.some(rem => identityMatches(r, identityOf(rem)))
  );
  return [...confirmed, ...cs.additions];
}
```
7. Replace `_hasPendingChanges()`:
```ts
private _hasPendingChanges(): boolean {
  return this._pendingOperations.length > 0;
}
```
8. Replace `_isPendingAddition` and related checks to use identity matching:
```ts
private _isPendingRemoval(rel: AgentRelationship): boolean {
  const cs = flattenOperations(this._pendingOperations, this.relationships);
  return cs.removals.some(r => identityMatches(rel, identityOf(r)));
}

private _isPendingAddition(rel: AgentRelationship): boolean {
  const cs = flattenOperations(this._pendingOperations, this.relationships);
  return cs.additions.some(a => identityMatches(rel, identityOf(a)));
}
```
9. Replace `_confirm()`:
```ts
private _confirm() {
  const changeset = flattenOperations(this._pendingOperations, this.relationships);
  this.dispatchEvent(new CustomEvent('pages-event', {
    bubbles: true,
    composed: true,
    detail: { topic: 'relationship:changed', payload: changeset },
  }));
  this._pendingOperations = [];
}
```
10. Replace `_reset()`:
```ts
private _reset() {
  this._pendingOperations = [];
  this._showAddForm = false;
}
```
11. Update `_addRelationship()` to create a `ChainOperation`:
```ts
private _addRelationship() {
  const source = this._addSourceId || this.agentId;
  if (!this._addTargetId || !source) return;
  const newRel: AgentRelationship = {
    sourceAgentId: source,
    targetAgentId: this._addTargetId,
    kind: this._addKind,
    tenancyId: this.tenancyId,
  };
  if (this._addScope) newRel.scope = { domain: this._addScope };
  const op: ChainOperation = {
    type: 'add-leaf',
    removals: [],
    additions: [newRel],
    description: `Add ${source} → ${this._addTargetId}`,
  };
  this._pendingOperations = [...this._pendingOperations, op];
  this._showAddForm = false;
  this._addSourceId = '';
  this._addTargetId = '';
  this._addScope = '';
}
```
12. Update the render method to use `_isPendingRemoval(rel)` and `_isPendingAddition(rel)` instead of index-based checks

- [ ] **Step 4: Run all existing tests to verify the refactor**

Run: `yarn workspace @casehubio/mosaic-agent-relationship-editor run test -- --run`
Expected: all PASS

- [ ] **Step 5: Update index.ts exports**

In `components/agent-relationship-editor/src/index.ts`, add:

```ts
export type { RelationshipIdentity, ChainOperation } from './types.js';
export { identityOf, identityKey, identityMatches, flattenOperations } from './identity.js';
```

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add components/agent-relationship-editor/src/
git commit -m "refactor(#234): migrate editor from index-based to identity-based changeset tracking

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Chain View Component

### Task 4: chain-view element — tree rendering, kind selector, ARIA

**Files:**
- Create: `components/agent-relationship-editor/src/chain-view.ts` — chain view element
- Create: `components/agent-relationship-editor/src/chain-view.test.ts` — chain view tests
- Modify: `components/agent-relationship-editor/src/index.ts` — export chain-view

**Interfaces:**
- Consumes: `buildChainTree`, `findChainRoots`, `ChainNode` from Task 1; `AgentRelationship`, `RelationshipKind`, `AgentDescriptor`, `RelationshipScope` from graph-stencil-org; `AgentRosterEntry`, `ChainOperation`, `RelationshipIdentity` from types.ts; `identityOf`, `identityMatches` from identity.ts
- Produces: `<chain-view>` custom element with kind selector, indented tree, focal agent highlighting, ARIA tree pattern

- [ ] **Step 1: Write failing tests for chain-view rendering**

Create `components/agent-relationship-editor/src/chain-view.test.ts`:

```ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './chain-view.js';
import type { AgentRelationship } from '@casehubio/mosaic-graph-stencil-org';
import type { AgentRosterEntry } from './types.js';

function rel(source: string, target: string, kind = 'SUPERVISES' as const, extra?: Partial<AgentRelationship>): AgentRelationship {
  return { sourceAgentId: source, targetAgentId: target, kind, tenancyId: 't1', ...extra };
}

const RELS: AgentRelationship[] = [
  rel('A', 'B'), rel('B', 'C'), rel('B', 'D'), rel('A', 'E', 'DELEGATES_TO'),
];

const ROSTER: AgentRosterEntry[] = [
  { agentId: 'A', name: 'Root' },
  { agentId: 'B', name: 'Manager' },
  { agentId: 'C', name: 'Worker1' },
  { agentId: 'D', name: 'Worker2' },
  { agentId: 'E', name: 'Delegate' },
];

type ChainEl = HTMLElement & {
  relationships?: AgentRelationship[];
  roster?: AgentRosterEntry[];
  agents?: Record<string, any>;
  agentId?: string;
  tenancyId?: string;
  pendingOperations?: any[];
  updateComplete: Promise<boolean>;
};

describe('chain-view', () => {
  let el: ChainEl;

  beforeEach(() => {
    el = document.createElement('chain-view') as ChainEl;
    el.relationships = RELS;
    el.roster = ROSTER;
    el.agents = {};
    el.agentId = 'A';
    el.tenancyId = 't1';
    el.pendingOperations = [];
    document.body.appendChild(el);
  });

  afterEach(() => { el.remove(); });

  it('has role=tree and aria-label', async () => {
    await el.updateComplete;
    const tree = el.shadowRoot!.querySelector('[role="tree"]');
    expect(tree).toBeTruthy();
    expect(tree!.getAttribute('aria-label')).toContain('Supervises');
  });

  it('renders kind selector excluding BACKS_UP', async () => {
    el.relationships = [...RELS, rel('A', 'B', 'BACKS_UP')];
    await el.updateComplete;
    const selector = el.shadowRoot!.querySelector('[role="radiogroup"]');
    expect(selector).toBeTruthy();
    const labels = [...selector!.querySelectorAll('[role="radio"]')].map(r => r.textContent!.trim());
    expect(labels).toContain('Supervises');
    expect(labels).toContain('Delegates To');
    expect(labels).not.toContain('Backs Up');
  });

  it('renders tree items with aria-level', async () => {
    await el.updateComplete;
    const items = el.shadowRoot!.querySelectorAll('[role="treeitem"]');
    expect(items.length).toBeGreaterThanOrEqual(3);
    expect(items[0].getAttribute('aria-level')).toBe('1');
  });

  it('highlights focal agent row', async () => {
    el.agentId = 'B';
    await el.updateComplete;
    const highlighted = el.shadowRoot!.querySelector('.focal-agent');
    expect(highlighted).toBeTruthy();
    expect(highlighted!.textContent).toContain('Manager');
  });

  it('shows cycle marker for cyclic nodes', async () => {
    el.relationships = [rel('A', 'B'), rel('B', 'C'), rel('C', 'A')];
    await el.updateComplete;
    const cycleMarker = el.shadowRoot!.querySelector('.cycle-marker');
    expect(cycleMarker).toBeTruthy();
  });

  it('renders multiple trees for multiple roots', async () => {
    el.relationships = [rel('R1', 'X'), rel('R2', 'X')];
    el.roster = [{ agentId: 'R1', name: 'Root1' }, { agentId: 'R2', name: 'Root2' }, { agentId: 'X', name: 'X' }];
    el.agentId = 'X';
    await el.updateComplete;
    const trees = el.shadowRoot!.querySelectorAll('.chain-tree');
    expect(trees.length).toBe(2);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/mosaic-agent-relationship-editor run test -- --run chain-view`
Expected: FAIL — module not found

- [ ] **Step 3: Implement chain-view element — class skeleton and kind selector**

Create `components/agent-relationship-editor/src/chain-view.ts`:

```ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import type { AgentRelationship, RelationshipKind, AgentDescriptor, ChainNode } from '@casehubio/mosaic-graph-stencil-org';
import { buildChainTree, findChainRoots, lookupRelationshipType } from '@casehubio/mosaic-graph-stencil-org';
import { identityOf, identityMatches } from './identity.js';
import type { AgentRosterEntry, ChainOperation } from './types.js';
import '@casehubio/mosaic-agent-avatar-2d';

interface KindEntry {
  kind: RelationshipKind;
  extendedKind?: string;
  label: string;
}

@customElement('chain-view')
export class ChainView extends LitElement {
  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui); }
    .kind-selector { display: flex; gap: 2px; margin-bottom: var(--pages-space-3, 0.75rem); }
    .kind-btn { padding: 6px 12px; border: 1px solid var(--pages-neutral-4, #e5e5e5); border-radius: 4px; background: var(--pages-neutral-2, #fafafa); cursor: pointer; font-size: 13px; font-family: inherit; }
    .kind-btn.active { background: var(--pages-accent-9, #0066cc); color: var(--pages-accent-contrast, #fff); border-color: var(--pages-accent-9, #0066cc); }
    .tree-node { display: flex; align-items: center; gap: 6px; padding: 4px 8px; font-size: 13px; border-radius: 4px; cursor: default; }
    .tree-node:focus { outline: 2px solid var(--pages-accent-7, #0066cc); outline-offset: -2px; }
    .tree-node.focal-agent { background: var(--pages-accent-2, #eff6ff); }
    .tree-node.pending-removal { background: var(--pages-danger-2, #fef2f2); text-decoration: line-through; opacity: 0.6; }
    .tree-node.pending-addition { background: var(--pages-success-2, #f0fdf4); }
    .expand-toggle { background: none; border: none; cursor: pointer; padding: 0 4px; font-size: 14px; color: var(--pages-neutral-9, #737373); }
    .agent-name { font-weight: 500; }
    .agent-role { color: var(--pages-text-secondary, #6b7280); font-size: 0.85em; }
    .cycle-marker { color: var(--pages-warning-9, #d97706); border: 1px dashed var(--pages-warning-6, #f59e0b); border-radius: 4px; padding: 2px 6px; font-size: 11px; }
    .action-btn { background: none; border: 1px solid var(--pages-neutral-4, #e5e5e5); padding: 2px 6px; border-radius: 4px; cursor: pointer; font-size: 11px; margin-left: 4px; opacity: 0; transition: opacity 0.15s; }
    .tree-node:hover .action-btn, .tree-node:focus-within .action-btn { opacity: 1; }
    .chain-tree { margin-bottom: var(--pages-space-3, 0.75rem); }
    .tree-separator { border-top: 1px dashed var(--pages-neutral-4, #e5e5e5); margin: var(--pages-space-2, 0.5rem) 0; }
  `;

  @property({ attribute: false }) relationships: AgentRelationship[] = [];
  @property({ attribute: false }) roster: AgentRosterEntry[] = [];
  @property({ attribute: false }) agents: Record<string, AgentDescriptor> = {};
  @property({ type: String, attribute: 'agent-id' }) agentId = '';
  @property({ type: String, attribute: 'tenancy-id' }) tenancyId = 'default';
  @property({ attribute: false }) pendingOperations: ChainOperation[] = [];

  @state() private _selectedKind: RelationshipKind = 'SUPERVISES';
  @state() private _selectedExtendedKind?: string;
  @state() private _expandedNodes = new Set<string>();

  override connectedCallback() {
    super.connectedCallback();
    this._selectFirstPopulatedKind();
  }

  override updated(changed: Map<string, unknown>) {
    if (changed.has('relationships')) this._selectFirstPopulatedKind();
  }

  private _selectFirstPopulatedKind() {
    const kinds = this._getAvailableKinds();
    if (kinds.length > 0 && !kinds.some(k => k.kind === this._selectedKind && k.extendedKind === this._selectedExtendedKind)) {
      this._selectedKind = kinds[0].kind;
      this._selectedExtendedKind = kinds[0].extendedKind;
    }
  }

  private _getAvailableKinds(): KindEntry[] {
    const seen = new Set<string>();
    const entries: KindEntry[] = [];
    for (const rel of this.relationships) {
      if (rel.kind === 'BACKS_UP') continue;
      const key = rel.kind === 'EXTENDED' ? `${rel.kind}::${rel.extendedKind ?? ''}` : rel.kind;
      if (seen.has(key)) continue;
      seen.add(key);
      if (rel.kind === 'EXTENDED' && rel.extendedKind) {
        const label = `${rel.extendedKind.charAt(0).toUpperCase()}${rel.extendedKind.slice(1)} (Extended)`;
        entries.push({ kind: rel.kind, extendedKind: rel.extendedKind, label });
      } else {
        const desc = lookupRelationshipType(rel.kind.toLowerCase());
        entries.push({ kind: rel.kind, label: desc.label ?? rel.kind });
      }
    }
    return entries;
  }

  private _getAgentName(agentId: string): string {
    return this.roster.find(r => r.agentId === agentId)?.name ?? agentId;
  }

  private _getEffectiveRelationships(): AgentRelationship[] {
    const removedKeys = new Set<string>();
    const added: AgentRelationship[] = [];
    for (const op of this.pendingOperations) {
      for (const r of op.removals) removedKeys.add(`${r.sourceAgentId}::${r.targetAgentId}::${r.kind}`);
      added.push(...op.additions);
    }
    return [...this.relationships.filter(r => !removedKeys.has(`${r.sourceAgentId}::${r.targetAgentId}::${r.kind}`)), ...added];
  }

  private _isPendingRemoval(rel: AgentRelationship): boolean {
    for (const op of this.pendingOperations) {
      if (op.removals.some(r => identityMatches(rel, r))) return true;
    }
    return false;
  }

  private _isPendingAddition(rel: AgentRelationship): boolean {
    for (const op of this.pendingOperations) {
      if (op.additions.some(a => identityMatches(rel, identityOf(a)))) return true;
    }
    return false;
  }

  private _buildTrees(): ChainNode[] {
    const effective = this._getEffectiveRelationships();
    const roots = findChainRoots(effective, this.agentId, this._selectedKind, this._selectedExtendedKind);
    return roots.map(root => buildChainTree(effective, root, this._selectedKind, this._selectedExtendedKind));
  }

  override render() {
    const kinds = this._getAvailableKinds();
    const trees = this._buildTrees();
    const kindLabel = kinds.find(k => k.kind === this._selectedKind && k.extendedKind === this._selectedExtendedKind)?.label ?? this._selectedKind;

    return html`
      ${this._renderKindSelector(kinds)}
      <div role="tree" aria-label="Chain navigator for ${kindLabel} relationships">
        ${trees.map((tree, i) => html`
          ${i > 0 ? html`<div class="tree-separator"></div>` : nothing}
          <div class="chain-tree">${this._renderNode(tree, 1)}</div>
        `)}
      </div>
    `;
  }

  private _renderKindSelector(kinds: KindEntry[]) {
    if (kinds.length <= 1) return nothing;
    return html`
      <div class="kind-selector" role="radiogroup" aria-label="Relationship kind">
        ${kinds.map(k => html`
          <button role="radio" class="kind-btn ${k.kind === this._selectedKind && k.extendedKind === this._selectedExtendedKind ? 'active' : ''}"
            aria-checked=${k.kind === this._selectedKind && k.extendedKind === this._selectedExtendedKind}
            @click=${() => { this._selectedKind = k.kind; this._selectedExtendedKind = k.extendedKind; this._expandedNodes = new Set(); }}>
            ${k.label}
          </button>
        `)}
      </div>
    `;
  }

  private _renderNode(node: ChainNode, level: number, parentId?: string): unknown {
    if (node.isCycle) {
      return html`
        <div role="treeitem" aria-level=${level} class="tree-node cycle-marker"
          style="padding-left: ${(level - 1) * 16 + 8}px">
          ↩ ${this._getAgentName(node.agentId)}
        </div>
      `;
    }

    const hasChildren = node.children.length > 0;
    const isExpanded = !this._expandedNodes.has(`collapsed:${node.agentId}:${level}`);
    const isFocal = node.agentId === this.agentId;
    const descriptor = this.agents?.[node.agentId];

    const matchingRel = parentId
      ? this.relationships.find(r => r.sourceAgentId === parentId && r.targetAgentId === node.agentId && r.kind === this._selectedKind)
      : undefined;
    const isRemoval = matchingRel ? this._isPendingRemoval(matchingRel) : false;
    const isAddition = matchingRel ? this._isPendingAddition(matchingRel) : false;

    const nodeClasses = ['tree-node', isFocal ? 'focal-agent' : '', isRemoval ? 'pending-removal' : '', isAddition ? 'pending-addition' : ''].filter(Boolean).join(' ');

    return html`
      <div role="treeitem" aria-level=${level} aria-expanded=${hasChildren ? isExpanded : nothing}
        class=${nodeClasses} tabindex="-1"
        style="padding-left: ${(level - 1) * 16 + 8}px">
        ${hasChildren ? html`
          <button class="expand-toggle" aria-hidden="true"
            @click=${() => this._toggleExpand(node.agentId, level)}>
            ${isExpanded ? '▾' : '▸'}
          </button>
        ` : html`<span style="width: 20px; display: inline-block"></span>`}
        ${descriptor?.avatar ? html`<agent-avatar .code=${descriptor.avatar} size="xs" aria-hidden="true"></agent-avatar>` : nothing}
        <span class="agent-name">${this._getAgentName(node.agentId)}</span>
        ${descriptor?.role ? html`<span class="agent-role">${descriptor.role}</span>` : nothing}
        <span style="flex: 1"></span>
        ${!node.isConvergent || level === 1 ? html`
          ${parentId ? html`<button class="action-btn btn-insert" aria-label="Insert before ${this._getAgentName(node.agentId)}">+ insert</button>` : nothing}
          <button class="action-btn btn-add" aria-label="Add child to ${this._getAgentName(node.agentId)}">+ add</button>
          <button class="action-btn btn-remove" aria-label="Remove ${this._getAgentName(node.agentId)}">×</button>
        ` : nothing}
      </div>
      ${hasChildren && isExpanded ? node.children.map(child => this._renderNode(child, level + 1, node.agentId)) : nothing}
    `;
  }

  private _toggleExpand(agentId: string, level: number) {
    const key = `collapsed:${agentId}:${level}`;
    const next = new Set(this._expandedNodes);
    if (next.has(key)) next.delete(key); else next.add(key);
    this._expandedNodes = next;
  }
}
```

This renders the tree, kind selector, and ARIA. Editing interactions are added in Task 5.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/mosaic-agent-relationship-editor run test -- --run chain-view`
Expected: all PASS

- [ ] **Step 5: Export from index.ts**

In `components/agent-relationship-editor/src/index.ts`, add:

```ts
export { ChainView } from './chain-view.js';
```

- [ ] **Step 6: Commit**

```bash
git add components/agent-relationship-editor/src/chain-view.ts components/agent-relationship-editor/src/chain-view.test.ts components/agent-relationship-editor/src/index.ts
git commit -m "feat(#234): add chain-view element with tree rendering, kind selector, and ARIA

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 5: chain-view inline editing operations

**Files:**
- Modify: `components/agent-relationship-editor/src/chain-view.ts` — add editing interactions
- Modify: `components/agent-relationship-editor/src/chain-view.test.ts` — add editing tests

**Interfaces:**
- Consumes: all from Task 4, plus `identityOf` from identity.ts
- Produces: `chain:operation` events with correct `ChainOperation` payloads for all 6 operation types

- [ ] **Step 1: Write failing tests for editing operations**

Add to `chain-view.test.ts`:

```ts
describe('chain-view editing', () => {
  let el: ChainEl;
  let operations: any[];

  beforeEach(() => {
    el = document.createElement('chain-view') as ChainEl;
    el.relationships = [rel('A', 'B'), rel('B', 'C'), rel('B', 'D')];
    el.roster = ROSTER;
    el.agents = {};
    el.agentId = 'A';
    el.tenancyId = 't1';
    el.pendingOperations = [];
    operations = [];
    el.addEventListener('pages-event', ((e: CustomEvent) => {
      if (e.detail?.topic === 'chain:operation') operations.push(e.detail.payload);
    }) as EventListener);
    document.body.appendChild(el);
  });

  afterEach(() => { el.remove(); });

  it('delete on leaf emits delete operation', async () => {
    await el.updateComplete;
    const leafRow = [...el.shadowRoot!.querySelectorAll('[role="treeitem"]')]
      .find(r => r.textContent!.includes('Worker1'));
    const deleteBtn = leafRow!.querySelector('.btn-remove') as HTMLElement;
    deleteBtn.click();
    await el.updateComplete;
    expect(operations).toHaveLength(1);
    expect(operations[0].type).toBe('delete');
    expect(operations[0].removals).toHaveLength(1);
  });

  it('remove on non-leaf non-root shows reconnect/disconnect options', async () => {
    await el.updateComplete;
    const managerRow = [...el.shadowRoot!.querySelectorAll('[role="treeitem"]')]
      .find(r => r.textContent!.includes('Manager'));
    const deleteBtn = managerRow!.querySelector('.btn-remove') as HTMLElement;
    deleteBtn.click();
    await el.updateComplete;
    const options = managerRow!.querySelectorAll('.removal-option');
    expect(options.length).toBeGreaterThanOrEqual(2);
  });

  it('remove-and-reconnect emits compound operation', async () => {
    await el.updateComplete;
    const managerRow = [...el.shadowRoot!.querySelectorAll('[role="treeitem"]')]
      .find(r => r.textContent!.includes('Manager'));
    managerRow!.querySelector('.btn-remove')!.dispatchEvent(new Event('click'));
    await el.updateComplete;
    const reconnectBtn = managerRow!.querySelector('.btn-reconnect') as HTMLElement;
    reconnectBtn.click();
    await el.updateComplete;
    expect(operations).toHaveLength(1);
    expect(operations[0].type).toBe('remove-reconnect');
    expect(operations[0].removals.length).toBeGreaterThanOrEqual(2);
    expect(operations[0].additions.length).toBeGreaterThanOrEqual(1);
  });

  it('add-leaf emits add-leaf operation', async () => {
    await el.updateComplete;
    const leafRow = [...el.shadowRoot!.querySelectorAll('[role="treeitem"]')]
      .find(r => r.textContent!.includes('Worker1'));
    const addBtn = leafRow!.querySelector('.btn-add') as HTMLElement;
    addBtn.click();
    await el.updateComplete;
    const picker = el.shadowRoot!.querySelector('.agent-picker-dropdown');
    expect(picker).toBeTruthy();
  });

  it('insert-between emits insert-between operation', async () => {
    await el.updateComplete;
    const childRow = [...el.shadowRoot!.querySelectorAll('[role="treeitem"]')]
      .find(r => r.textContent!.includes('Manager'));
    const insertBtn = childRow!.querySelector('.btn-insert') as HTMLElement;
    insertBtn.click();
    await el.updateComplete;
    const picker = el.shadowRoot!.querySelector('.agent-picker-dropdown');
    expect(picker).toBeTruthy();
  });

  it('root node removal shows only disconnect option', async () => {
    await el.updateComplete;
    const rootRow = [...el.shadowRoot!.querySelectorAll('[role="treeitem"]')]
      .find(r => r.textContent!.includes('Root'));
    const deleteBtn = rootRow!.querySelector('.btn-remove') as HTMLElement;
    deleteBtn.click();
    await el.updateComplete;
    expect(rootRow!.querySelector('.btn-reconnect')).toBeFalsy();
    expect(rootRow!.querySelector('.btn-disconnect')).toBeTruthy();
  });

  it('agent picker for add-leaf excludes ancestors', async () => {
    await el.updateComplete;
    const childRow = [...el.shadowRoot!.querySelectorAll('[role="treeitem"]')]
      .find(r => r.textContent!.includes('Worker1'));
    const addBtn = childRow!.querySelector('.btn-add') as HTMLElement;
    addBtn.click();
    await el.updateComplete;
    const options = el.shadowRoot!.querySelectorAll('.agent-picker-dropdown option');
    const values = [...options].map(o => (o as HTMLOptionElement).value);
    expect(values).not.toContain('A');
    expect(values).not.toContain('B');
    expect(values).not.toContain('C');
  });

  it('pending removal renders with strikethrough', async () => {
    el.pendingOperations = [{
      type: 'delete',
      removals: [{ sourceAgentId: 'B', targetAgentId: 'C', kind: 'SUPERVISES' as const }],
      additions: [],
      description: '',
    }];
    await el.updateComplete;
    const pendingRow = el.shadowRoot!.querySelector('.pending-removal');
    expect(pendingRow).toBeTruthy();
  });

  it('insert-between inherits scope but not attestation', async () => {
    el.relationships = [rel('A', 'B', 'SUPERVISES', { scope: { domain: 'billing' }, attestation: { dimensions: ['trust'] } })];
    el.roster = [...ROSTER, { agentId: 'F', name: 'Inserted' }];
    await el.updateComplete;
    const childRow = [...el.shadowRoot!.querySelectorAll('[role="treeitem"]')]
      .find(r => r.textContent!.includes('Manager'));
    childRow!.querySelector('.btn-insert')!.dispatchEvent(new Event('click'));
    await el.updateComplete;
    const picker = el.shadowRoot!.querySelector('.agent-picker-dropdown select') as HTMLSelectElement;
    picker.value = 'F';
    picker.dispatchEvent(new Event('change'));
    await el.updateComplete;
    expect(operations).toHaveLength(1);
    expect(operations[0].additions[0].scope).toEqual({ domain: 'billing' });
    expect(operations[0].additions[0].attestation).toBeUndefined();
  });

  it('edit creating a cycle shows cycle marker in pending state', async () => {
    el.relationships = [rel('A', 'B'), rel('B', 'C')];
    el.pendingOperations = [{
      type: 'add-leaf',
      removals: [],
      additions: [rel('C', 'A')],
      description: 'Add C → A',
    }];
    await el.updateComplete;
    const cycleMarker = el.shadowRoot!.querySelector('.cycle-marker');
    expect(cycleMarker).toBeTruthy();
  });

  it('EXTENDED sub-grouping shows separate kind entries', async () => {
    el.relationships = [
      rel('A', 'B', 'EXTENDED', { extendedKind: 'mentors' }),
      rel('A', 'C', 'EXTENDED', { extendedKind: 'reviews' }),
    ];
    await el.updateComplete;
    const selector = el.shadowRoot!.querySelector('[role="radiogroup"]');
    const labels = [...selector!.querySelectorAll('[role="radio"]')].map(r => r.textContent!.trim());
    expect(labels).toContain('Mentors (Extended)');
    expect(labels).toContain('Reviews (Extended)');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/mosaic-agent-relationship-editor run test -- --run chain-view`
Expected: FAIL — editing methods not implemented

- [ ] **Step 3: Implement editing interactions**

Add the following methods to `chain-view.ts`:

```ts
// --- State for editing ---
@state() private _editingNodeId: string | null = null;
@state() private _pickerTarget: { agentId: string; parentId?: string; mode: 'add' | 'insert' } | null = null;

// --- Event emission ---
private _emitOperation(op: ChainOperation) {
  this.dispatchEvent(new CustomEvent('pages-event', {
    bubbles: true,
    detail: { topic: 'chain:operation', payload: op },
  }));
  this._editingNodeId = null;
  this._pickerTarget = null;
}

// --- Removal handlers ---
private _handleRemoveClick(agentId: string, parentId: string | undefined, children: ChainNode[]) {
  if (children.length === 0) {
    if (!parentId) return;
    const id = { sourceAgentId: parentId, targetAgentId: agentId, kind: this._selectedKind } as RelationshipIdentity;
    this._emitOperation({ type: 'delete', removals: [id], additions: [], description: `Remove ${parentId} → ${agentId}` });
  } else {
    this._editingNodeId = agentId;
  }
}

private _handleReconnect(agentId: string, parentId: string, children: ChainNode[]) {
  const removals: RelationshipIdentity[] = [
    { sourceAgentId: parentId, targetAgentId: agentId, kind: this._selectedKind },
  ];
  const additions: AgentRelationship[] = [];
  for (const child of children) {
    const childRel = this.relationships.find(r => r.sourceAgentId === agentId && r.targetAgentId === child.agentId && r.kind === this._selectedKind);
    removals.push({ sourceAgentId: agentId, targetAgentId: child.agentId, kind: this._selectedKind });
    const newRel: AgentRelationship = {
      sourceAgentId: parentId, targetAgentId: child.agentId, kind: this._selectedKind, tenancyId: this.tenancyId,
    };
    if (childRel?.scope) newRel.scope = childRel.scope;
    if (childRel?.extendedKind) newRel.extendedKind = childRel.extendedKind;
    if (childRel?.kindVocabulary) newRel.kindVocabulary = childRel.kindVocabulary;
    additions.push(newRel);
  }
  this._emitOperation({ type: 'remove-reconnect', removals, additions, description: `Remove ${this._getAgentName(agentId)}, reconnect children to ${this._getAgentName(parentId)}` });
}

private _handleDisconnect(agentId: string, parentId: string | undefined, children: ChainNode[]) {
  const removals: RelationshipIdentity[] = [];
  if (parentId) removals.push({ sourceAgentId: parentId, targetAgentId: agentId, kind: this._selectedKind });
  for (const child of children) {
    removals.push({ sourceAgentId: agentId, targetAgentId: child.agentId, kind: this._selectedKind });
  }
  this._emitOperation({ type: 'remove-disconnect', removals, additions: [], description: `Remove ${this._getAgentName(agentId)} and disconnect` });
}

// --- Insert/Add handlers ---
private _showPicker(agentId: string, parentId: string | undefined, mode: 'add' | 'insert') {
  this._pickerTarget = { agentId, parentId, mode };
}

private _handlePickerSelect(selectedAgentId: string) {
  if (!this._pickerTarget) return;
  const { agentId, parentId, mode } = this._pickerTarget;
  if (mode === 'add') {
    const newRel: AgentRelationship = { sourceAgentId: agentId, targetAgentId: selectedAgentId, kind: this._selectedKind, tenancyId: this.tenancyId };
    if (this._selectedExtendedKind) newRel.extendedKind = this._selectedExtendedKind;
    this._emitOperation({ type: 'add-leaf', removals: [], additions: [newRel], description: `Add ${this._getAgentName(selectedAgentId)} under ${this._getAgentName(agentId)}` });
  } else if (mode === 'insert' && parentId) {
    const existingRel = this.relationships.find(r => r.sourceAgentId === parentId && r.targetAgentId === agentId && r.kind === this._selectedKind);
    const removal: RelationshipIdentity = { sourceAgentId: parentId, targetAgentId: agentId, kind: this._selectedKind };
    const rel1: AgentRelationship = { sourceAgentId: parentId, targetAgentId: selectedAgentId, kind: this._selectedKind, tenancyId: this.tenancyId };
    const rel2: AgentRelationship = { sourceAgentId: selectedAgentId, targetAgentId: agentId, kind: this._selectedKind, tenancyId: this.tenancyId };
    if (existingRel?.scope) { rel1.scope = existingRel.scope; rel2.scope = existingRel.scope; }
    if (existingRel?.extendedKind) { rel1.extendedKind = existingRel.extendedKind; rel2.extendedKind = existingRel.extendedKind; }
    this._emitOperation({ type: 'insert-between', removals: [removal], additions: [rel1, rel2], description: `Insert ${this._getAgentName(selectedAgentId)} between ${this._getAgentName(parentId)} and ${this._getAgentName(agentId)}` });
  }
}

// --- Picker candidate filtering ---
private _getPickerCandidates(nodeId: string, parentId: string | undefined, mode: 'add' | 'insert'): AgentRosterEntry[] {
  const ancestors = this._getAncestors(nodeId);
  return this.roster.filter(r => {
    if (mode === 'add') return !ancestors.has(r.agentId) && r.agentId !== nodeId;
    if (mode === 'insert') return !ancestors.has(r.agentId) && r.agentId !== nodeId && r.agentId !== parentId;
    return true;
  });
}

private _getAncestors(agentId: string): Set<string> {
  const ancestors = new Set<string>();
  const matching = this._getEffectiveRelationships().filter(r => r.kind === this._selectedKind);
  function walkUp(current: string, visited: Set<string>) {
    for (const r of matching) {
      if (r.targetAgentId === current && !visited.has(r.sourceAgentId)) {
        ancestors.add(r.sourceAgentId);
        visited.add(r.sourceAgentId);
        walkUp(r.sourceAgentId, visited);
      }
    }
  }
  walkUp(agentId, new Set([agentId]));
  return ancestors;
}

// --- Rendering helpers for editing UI ---
private _renderRemovalOptions(agentId: string, parentId: string | undefined, children: ChainNode[], isRoot: boolean) {
  if (this._editingNodeId !== agentId) return nothing;
  return html`
    <div class="removal-options" role="group" aria-label="Removal options for ${this._getAgentName(agentId)}">
      ${!isRoot ? html`<button class="removal-option btn-reconnect" @click=${() => this._handleReconnect(agentId, parentId!, children)}>Remove and reconnect</button>` : nothing}
      <button class="removal-option btn-disconnect" @click=${() => this._handleDisconnect(agentId, parentId, children)}>Remove and disconnect</button>
      <button class="removal-option" @click=${() => { this._editingNodeId = null; }}>Cancel</button>
    </div>
  `;
}

private _renderAgentPicker(nodeId: string, parentId: string | undefined, mode: 'add' | 'insert') {
  if (!this._pickerTarget || this._pickerTarget.agentId !== nodeId || this._pickerTarget.mode !== mode) return nothing;
  const candidates = this._getPickerCandidates(nodeId, parentId, mode);
  return html`
    <div class="agent-picker-dropdown">
      <select @change=${(e: Event) => { const v = (e.target as HTMLSelectElement).value; if (v) this._handlePickerSelect(v); }}>
        <option value="">Select agent...</option>
        ${candidates.map(r => html`<option value=${r.agentId}>${r.name}</option>`)}
      </select>
      <button @click=${() => { this._pickerTarget = null; }}>Cancel</button>
    </div>
  `;
}
```

Update `_renderNode` to wire the action buttons to these handlers: `btn-remove` calls `_handleRemoveClick`, `btn-add` calls `_showPicker(agentId, parentId, 'add')`, `btn-insert` calls `_showPicker(agentId, parentId, 'insert')`. Render `_renderRemovalOptions` and `_renderAgentPicker` inline after the action buttons.

Add a `_renderPendingOperationsPanel()` at the bottom of `render()`:

```ts
private _renderPendingOperationsPanel() {
  if (this.pendingOperations.length === 0) return nothing;
  return html`
    <div class="pending-ops-panel" role="log" aria-label="Pending operations">
      ${this.pendingOperations.map((op, i) => html`
        <div class="pending-op-row">
          <span>${op.description}</span>
          <button class="btn-dismiss" aria-label="Dismiss operation" @click=${() => {
            this.dispatchEvent(new CustomEvent('pages-event', { bubbles: true, detail: { topic: 'chain:dismiss-operation', payload: { index: i } } }));
          }}>×</button>
        </div>
      `)}
    </div>
  `;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/mosaic-agent-relationship-editor run test -- --run chain-view`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add components/agent-relationship-editor/src/chain-view.ts components/agent-relationship-editor/src/chain-view.test.ts
git commit -m "feat(#234): add inline editing operations to chain-view

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Editor Integration and Polish

### Task 6: Wire chain-view into editor as third tab

**Files:**
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.ts` — add Chain tab, wire events
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.test.ts` — integration tests

**Interfaces:**
- Consumes: `<chain-view>` element from Task 4/5, `ChainOperation` from types, `flattenOperations` from identity
- Produces: Complete editor with Table/Chain/Diagram tabs, shared changeset, cross-tab consistency

- [ ] **Step 1: Write failing integration tests**

Add to `agent-relationship-editor.test.ts`:

```ts
describe('chain tab integration', () => {
  it('renders Chain tab button', async () => {
    await el.updateComplete;
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    const labels = [...tabs].map(t => t.textContent!.trim());
    expect(labels).toContain('Chain');
  });

  it('switches to chain view on Chain tab click', async () => {
    await el.updateComplete;
    const chainTab = [...el.shadowRoot!.querySelectorAll('[role="tab"]')]
      .find(t => t.textContent!.trim() === 'Chain') as HTMLElement;
    chainTab.click();
    await el.updateComplete;
    const chainView = el.shadowRoot!.querySelector('chain-view');
    expect(chainView).toBeTruthy();
  });

  it('chain-view receives correct properties', async () => {
    await el.updateComplete;
    const chainTab = [...el.shadowRoot!.querySelectorAll('[role="tab"]')]
      .find(t => t.textContent!.trim() === 'Chain') as HTMLElement;
    chainTab.click();
    await el.updateComplete;
    const chainView = el.shadowRoot!.querySelector('chain-view') as any;
    expect(chainView.relationships).toBe(el.relationships);
    expect(chainView.roster).toBe(el.roster);
    expect(chainView.agentId).toBe('alice');
    expect(chainView.tenancyId).toBe('default');
  });

  it('chain operation events update pending state', async () => {
    await el.updateComplete;
    const chainTab = [...el.shadowRoot!.querySelectorAll('[role="tab"]')]
      .find(t => t.textContent!.trim() === 'Chain') as HTMLElement;
    chainTab.click();
    await el.updateComplete;
    const chainView = el.shadowRoot!.querySelector('chain-view') as HTMLElement;
    chainView.dispatchEvent(new CustomEvent('pages-event', {
      bubbles: true,
      detail: {
        topic: 'chain:operation',
        payload: {
          type: 'delete',
          removals: [{ sourceAgentId: 'alice', targetAgentId: 'bob', kind: 'SUPERVISES' }],
          additions: [],
          description: 'Remove Alice → Bob',
        },
      },
    }));
    await el.updateComplete;
    const confirmBar = el.shadowRoot!.querySelector('.confirm-bar');
    expect(confirmBar).toBeTruthy();
  });

  it('pending state visible across tabs', async () => {
    await el.updateComplete;
    // Add operation via chain tab
    const chainTab = [...el.shadowRoot!.querySelectorAll('[role="tab"]')]
      .find(t => t.textContent!.trim() === 'Chain') as HTMLElement;
    chainTab.click();
    await el.updateComplete;
    const chainView = el.shadowRoot!.querySelector('chain-view') as HTMLElement;
    chainView.dispatchEvent(new CustomEvent('pages-event', {
      bubbles: true,
      detail: {
        topic: 'chain:operation',
        payload: {
          type: 'delete',
          removals: [{ sourceAgentId: 'alice', targetAgentId: 'bob', kind: 'SUPERVISES' }],
          additions: [],
          description: 'Remove Alice → Bob',
        },
      },
    }));
    // Switch to table tab
    const tableTab = [...el.shadowRoot!.querySelectorAll('[role="tab"]')]
      .find(t => t.textContent!.trim() === 'Table') as HTMLElement;
    tableTab.click();
    await el.updateComplete;
    const pendingRow = el.shadowRoot!.querySelector('.pending-removal');
    expect(pendingRow).toBeTruthy();
  });

  it('confirm dispatches flattened changeset', async () => {
    await el.updateComplete;
    let changeset: any = null;
    el.addEventListener('pages-event', ((e: CustomEvent) => {
      if (e.detail?.topic === 'relationship:changed') changeset = e.detail.payload;
    }) as EventListener);

    // Remove via table
    const rows = el.shadowRoot!.querySelectorAll('.relationship-row');
    const removeBtn = rows[0].querySelector('.btn-delete') as HTMLElement;
    removeBtn.click();
    await el.updateComplete;

    const confirmBtn = el.shadowRoot!.querySelector('.btn-confirm') as HTMLElement;
    confirmBtn.click();
    await el.updateComplete;

    expect(changeset).toBeTruthy();
    expect(changeset.removals).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/mosaic-agent-relationship-editor run test -- --run agent-relationship-editor`
Expected: FAIL — no Chain tab rendered

- [ ] **Step 3: Add Chain tab and wire chain-view**

In `agent-relationship-editor.ts`:

1. Update `ViewTab` type: `type ViewTab = 'table' | 'chain' | 'diagram';`
2. Add import: `import './chain-view.js';`
3. Add Chain tab button in the view-tabs section:
```ts
<button role="tab" id="tab-chain" aria-selected=${this._viewTab === 'chain'} aria-controls="panel-chain" class="view-tab ${this._viewTab === 'chain' ? 'active' : ''}" @click=${() => { this._viewTab = 'chain'; }}>Chain</button>
```
4. Add chain panel rendering:
```ts
${this._viewTab === 'chain' ? html`
  <div role="tabpanel" id="panel-chain" aria-labelledby="tab-chain">
    <chain-view
      .relationships=${this.relationships}
      .roster=${this.roster}
      .agents=${this.agents}
      .agentId=${this.agentId}
      .tenancyId=${this.tenancyId}
      .pendingOperations=${this._pendingOperations}
    ></chain-view>
  </div>
` : nothing}
```
5. Add `chain:operation` event handler in `connectedCallback`:
```ts
this.addEventListener('pages-event', this._handleChainOperation as EventListener);
```
6. Implement handler:
```ts
private _handleChainOperation = (e: CustomEvent): void => {
  if (e.detail?.topic !== 'chain:operation') return;
  const op = e.detail.payload as ChainOperation;
  this._pendingOperations = [...this._pendingOperations, op];
};
```
7. Move confirm bar outside the table-specific panel so it's visible on all tabs

- [ ] **Step 4: Run all tests**

Run: `yarn workspace @casehubio/mosaic-agent-relationship-editor run test -- --run`
Expected: all PASS

- [ ] **Step 5: Run full build and typecheck**

Run: `yarn build`
Run: `yarn typecheck`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add components/agent-relationship-editor/src/
git commit -m "feat(#234): wire chain-view as third tab in relationship editor

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 7: Playwright visual verification

**Files:** None — browser testing only

**Interfaces:**
- Consumes: Complete editor with Chain tab from Task 6

- [ ] **Step 1: Start dev server and open editor demo**

Run the project's dev server or demo page that renders `agent-relationship-editor`. Navigate to it in Playwright MCP.

- [ ] **Step 2: Verify Chain tab renders**

Click the Chain tab. Verify:
- Kind selector appears with populated kinds
- Tree renders with correct indentation
- Focal agent is highlighted
- Expand/collapse toggles work

- [ ] **Step 3: Test inline editing**

- Click `[×]` on a leaf node — verify it gets struck through
- Click `[×]` on a mid-chain node — verify reconnect/disconnect options appear
- Click `[+ add]` on a leaf — verify agent picker appears
- Select an agent — verify green pending addition appears
- Click Confirm — verify event fires

- [ ] **Step 4: Test cross-tab consistency**

- Make an edit in Chain tab
- Switch to Table tab — verify pending state shows
- Switch back to Chain tab — verify pending state preserved
- Reset — verify all pending state clears

- [ ] **Step 5: Commit any fixes**

If visual testing reveals issues, fix and commit:
```bash
git add components/agent-relationship-editor/src/
git commit -m "fix(#234): visual polish from Playwright verification

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-11-05-chain-navigator-design.md] — design spec this plan implements
- `components/agent-relationship-editor/src/agent-relationship-editor.ts` — existing editor (456 lines, table+diagram tabs)
- `components/agent-relationship-editor/src/arc-view.ts` — sub-element precedent
- `components/agent-relationship-editor/src/types.ts:1-12` — existing types
- `components/agent-relationship-editor/src/index.ts:1-3` — current exports
- `components/agent-relationship-editor/src/agent-relationship-editor.test.ts` — existing test patterns
- `packages/graph-stencil-org/src/types.ts:40-57` — `RelationshipKind`, `AgentRelationship`
- `packages/graph-stencil-org/src/adapter/ego-subgraph.ts` — single-hop extraction precedent
- `packages/graph-stencil-org/src/adapter/derived-data.ts:61-75` — `walkEscalation` (flat visited-set)
- `packages/graph-stencil-org/src/index.ts` — current package exports
- `docs/protocols/mosaic/custom-event-namespace-prefix.md` — event naming protocol
- `docs/protocols/mosaic/component-customisation-pattern.md` — typed config + callbacks
- GitHub #234 — feature request
- GitHub #233 — relationship editor (landed)
