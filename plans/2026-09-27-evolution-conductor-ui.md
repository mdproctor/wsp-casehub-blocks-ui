# Evolution Conductor UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/engine#1149 — Evolution conductor UI
**Issue group:** casehubio/blocks-ui#174, #175, #176, #177, #178, #179

**Goal:** Build composable evolution conductor UI components — three configuration editors, a dashboard workbench, and a sample page.

**Architecture:** Two new component packages: `evolution-config` (shared types/API + three independent editor elements) and `evolution-workbench` (dashboard composition + sample page). Editors follow the mute-list CRUD pattern (table + inline add form + delete confirmation). Workbench uses kpi-metric-row summary bar + detail-pane tabs (with standalone mode enhancement). All components support dual data mode (endpoint or property).

**Tech Stack:** Lit 3, TypeScript, pages-table, pages-form (pages-viz), pages-data, PagesConfirmDialog, EventStreamController, vitest

## Global Constraints

- ARIA is mandatory on every `@customElement` — role + aria-label on host or primary container
- Component customisation via typed config + render callbacks (PP-20260713-8ea1af) — no slots for content
- Render callback output uses inline styles (shadow DOM boundary)
- All `pages-event` communication — no direct component-to-component coupling
- Dual data mode on every editor: endpoint property OR inline data property
- TypeScript interfaces mirror engine Java records exactly — no subsetting, no renaming
- Inline mode uses unidirectional data flow — component emits event, parent updates property, component re-renders (no optimistic updates)

---

## Batch 1: Foundation — types, API, events, package scaffolding

### Task 1: Scaffold evolution-config package

**Files:**
- Create: `components/evolution-config/package.json`
- Create: `components/evolution-config/tsconfig.json`
- Create: `components/evolution-config/tsconfig.build.json`
- Create: `components/evolution-config/src/types.ts`
- Create: `components/evolution-config/src/api.ts`
- Create: `components/evolution-config/src/events.ts`
- Create: `components/evolution-config/src/index.ts`
- Create: `components/evolution-config/src/types.test.ts`
- Create: `components/evolution-config/src/api.test.ts`
- Modify: `tsconfig.json` (root) — add project reference
- Modify: `packages/blocks-ui-schema/package.json` — add devDependency

**Interfaces:**
- Produces: All shared types (`DenyPatternView`, `WatchPattern`, `GatePolicy`, `GateMode`, `GateOutcome`, `StageDescriptor`, `CategoryDescriptor`, `ConductorInboxEntry`, `InboxEntryStatus`, `EscalationTrigger`, `ConductorDecision`, `EvolutionStateSnapshot`, `CategoryStateView`, `ImprovementStreamView`, `WatchPatternInput`), `EvolutionApi` class, `EvolutionEventTopics`, `emitEvolutionEvent<T>`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/blocks-ui-evolution-config",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc --project tsconfig.build.json",
    "test": "vitest run"
  },
  "dependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "@casehubio/pages-component": "*",
    "@casehubio/pages-data": "*",
    "@casehubio/pages-primitives": "*",
    "@casehubio/pages-table": "*",
    "@casehubio/pages-ui-components": "*",
    "@casehubio/pages-viz": "*",
    "lit": "^3.2.1"
  },
  "devDependencies": {
    "@vitest/browser": "^2.1.8",
    "jsdom": "^26.0.0",
    "typescript": "~5.7.2",
    "vite": "^6.0.3",
    "vitest": "^2.1.8"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  }
}
```

- [ ] **Step 2: Create tsconfig.json and tsconfig.build.json**

`tsconfig.json`:
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src/**/*"],
  "references": [
    { "path": "../../packages/blocks-ui-core" }
  ]
}
```

`tsconfig.build.json`:
```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "exclude": ["src/**/*.test.ts"]
}
```

- [ ] **Step 3: Add root tsconfig reference**

Add to `tsconfig.json` (root) references array:
```json
{ "path": "components/evolution-config/tsconfig.build.json" }
```

- [ ] **Step 4: Write types.ts**

Write all TypeScript interfaces from spec §2. Every field maps 1:1 to the engine Java records:

```typescript
// --- Deny patterns (mirrors DenyPatternView.java) ---
export interface DynamicDenyEntry {
  readonly pattern: string;
  readonly addedBy: string;
  readonly addedAt: string;
}

export interface DenyPatternView {
  readonly staticPatterns: readonly string[];
  readonly dynamicPatterns: readonly DynamicDenyEntry[];
}

// --- Watch patterns (mirrors WatchPattern.java) ---
export interface WatchPattern {
  readonly id: string;
  readonly category: string | null;
  readonly areaId: string | null;
  readonly targetPattern: string | null;
  readonly minEstimatedSize: number | null;
  readonly createdAt: string;
}

// --- Gate policy (mirrors GatePolicy.java) ---
export type GateMode = 'GATED' | 'AUTO' | 'NOTIFY';
export type GateOutcome = 'APPROVED' | 'REJECTED';

export interface GatePolicy {
  readonly modes: Record<string, GateMode> | null;
  readonly gateTimeoutMinutes: number | null;
}

// --- Domain metadata (mirrors CategoryDescriptor.java, StageDescriptor.java) ---
export interface CategoryDescriptor {
  readonly id: string;
  readonly name: string;
  readonly description: string;
  readonly domainId: string;
}

export interface StageDescriptor {
  readonly id: string;
  readonly name: string;
  readonly ordinal: number;
  readonly gateCheckpoint: boolean;
  readonly domainId: string;
}

// --- Conductor inbox (mirrors ConductorInboxEntry.java — all 14 fields) ---
export type InboxEntryStatus =
  | 'PENDING' | 'APPROVED' | 'REJECTED'
  | 'REDIRECTED' | 'TIMED_OUT' | 'AUTO_APPROVED';

export interface EscalationTrigger {
  readonly layer: 'CATEGORY_RULE' | 'WATCH_PATTERN' | 'CONFIDENCE_SCORE';
  readonly reason: string;
}

export interface ConductorDecision {
  readonly outcome: InboxEntryStatus;
  readonly reason: string | null;
  readonly feedback: string | null;
}

export interface ConductorInboxEntry {
  readonly caseId: string;
  readonly id: string;
  readonly stage: string;
  readonly status: InboxEntryStatus;
  readonly category: string | null;
  readonly areaId: string | null;
  readonly improvementCaseId: string | null;
  readonly summary: string | null;
  readonly escalationTriggers: readonly EscalationTrigger[];
  readonly confidence: number;
  readonly queuedAt: string;
  readonly resolvedAt: string | null;
  readonly timeoutMinutes: number | null;
  readonly decision: ConductorDecision | null;
}

// --- Evolution state (mirrors EvolutionStateSnapshot.java) ---
export interface CategoryStateView {
  readonly successCount: number;
  readonly failureCount: number;
  readonly rejectionCount: number;
  readonly paused: boolean;
  readonly pausedUntil: string | null;
  readonly suppressed: boolean;
}

export interface EvolutionStateSnapshot {
  readonly caseId: string;
  readonly timestamp: string;
  readonly healthScore: number;
  readonly componentScores: Record<string, number>;
  readonly healthDelta: number;
  readonly healthWindowMinutes: number;
  readonly circuitBreakerState: string;
  readonly categoryStates: Record<string, CategoryStateView>;
  readonly projectComplianceLevel: string;
  readonly complianceEvaluatedAt: string | null;
  readonly areaComplianceLevels: Record<string, string>;
  readonly activeImprovementCount: number;
  readonly dailyImprovementCount: number;
  readonly evolutionEnabled: boolean;
  readonly pendingInboxCount: number;
}

// --- Improvement streams (mirrors EvolutionStateSnapshot.ImprovementStreamView) ---
export interface ImprovementStreamView {
  readonly improvementCaseId: string;
  readonly category: string;
  readonly target: string | null;
  readonly currentStage: string;
  readonly blockedBy: string | null;
  readonly conflictBlocked: boolean;
  readonly startedAt: string;
}

// --- API input types ---
export interface WatchPatternInput {
  readonly category?: string;
  readonly areaId?: string;
  readonly targetPattern?: string;
  readonly minEstimatedSize?: number;
}
```

- [ ] **Step 5: Write types.test.ts**

Test that type constants are correct and type guards work:

```typescript
import { describe, it, expect } from 'vitest';
import type {
  DenyPatternView, WatchPattern, GatePolicy, GateMode, GateOutcome,
  InboxEntryStatus, ConductorInboxEntry, EvolutionStateSnapshot,
  StageDescriptor, CategoryDescriptor,
} from './types.js';

describe('types', () => {
  it('GateMode accepts all three values', () => {
    const modes: GateMode[] = ['GATED', 'AUTO', 'NOTIFY'];
    expect(modes).toHaveLength(3);
  });

  it('GateOutcome is APPROVED or REJECTED only', () => {
    const outcomes: GateOutcome[] = ['APPROVED', 'REJECTED'];
    expect(outcomes).toHaveLength(2);
  });

  it('InboxEntryStatus has all 6 engine values', () => {
    const statuses: InboxEntryStatus[] = [
      'PENDING', 'APPROVED', 'REJECTED', 'REDIRECTED', 'TIMED_OUT', 'AUTO_APPROVED',
    ];
    expect(statuses).toHaveLength(6);
  });

  it('ConductorInboxEntry has all 14 fields', () => {
    const entry: ConductorInboxEntry = {
      caseId: '00000000-0000-0000-0000-000000000001',
      id: 'entry-1', stage: 'pr-review', status: 'PENDING',
      category: 'lint-fix', areaId: null, improvementCaseId: null,
      summary: 'Fix lint violations in auth module',
      escalationTriggers: [{ layer: 'CONFIDENCE_SCORE', reason: 'High confidence' }],
      confidence: 0.85, queuedAt: '2026-09-26T10:00:00Z',
      resolvedAt: null, timeoutMinutes: 1440, decision: null,
    };
    expect(Object.keys(entry)).toHaveLength(14);
  });

  it('EvolutionStateSnapshot has all required fields', () => {
    const snapshot: EvolutionStateSnapshot = {
      caseId: '00000000-0000-0000-0000-000000000001',
      timestamp: '2026-09-26T10:00:00Z', healthScore: 0.82,
      componentScores: { 'test-coverage': 0.7, 'ci-stability': 0.9 },
      healthDelta: 0.03, healthWindowMinutes: 60,
      circuitBreakerState: 'CLOSED',
      categoryStates: {},
      projectComplianceLevel: 'BASIC',
      complianceEvaluatedAt: null,
      areaComplianceLevels: {},
      activeImprovementCount: 2, dailyImprovementCount: 5,
      evolutionEnabled: true, pendingInboxCount: 1,
    };
    expect(snapshot.healthScore).toBe(0.82);
  });
});
```

- [ ] **Step 6: Write events.ts**

```typescript
export const EvolutionEventTopics = {
  DENY_PATTERN_CHANGED: 'evolution:deny-pattern-changed',
  WATCH_PATTERN_CHANGED: 'evolution:watch-pattern-changed',
  GATE_POLICY_CHANGED: 'evolution:gate-policy-changed',
  GATE_RESOLVED: 'evolution:gate-resolved',
} as const;

export function emitEvolutionEvent<T>(target: EventTarget, topic: string, payload: T): void {
  target.dispatchEvent(new CustomEvent('pages-event', {
    bubbles: true, composed: true,
    detail: { topic, payload },
  }));
}
```

- [ ] **Step 7: Write api.ts**

```typescript
import type {
  DenyPatternView, WatchPattern, WatchPatternInput, GatePolicy,
  GateOutcome, StageDescriptor, CategoryDescriptor,
  EvolutionStateSnapshot, ImprovementStreamView, ConductorInboxEntry,
} from './types.js';

export class EvolutionApi {
  constructor(
    private readonly baseUrl: string,
    private readonly fetchFn: typeof fetch = fetch,
  ) {}

  async getDenyPatterns(caseId: string, tenancyId: string): Promise<DenyPatternView> {
    return this._get(`${this.baseUrl}/getDenyPatterns?caseId=${caseId}&tenancyId=${enc(tenancyId)}`);
  }

  async addDenyPattern(caseId: string, tenancyId: string, pattern: string): Promise<void> {
    await this._post(`${this.baseUrl}/addDenyPattern`, { caseId, tenancyId, pattern });
  }

  async removeDenyPattern(caseId: string, tenancyId: string, pattern: string): Promise<void> {
    await this._post(`${this.baseUrl}/removeDenyPattern`, { caseId, tenancyId, pattern });
  }

  async getWatchPatterns(caseId: string, tenancyId: string): Promise<WatchPattern[]> {
    return this._get(`${this.baseUrl}/getWatchPatterns?caseId=${caseId}&tenancyId=${enc(tenancyId)}`);
  }

  async addWatchPattern(caseId: string, tenancyId: string, input: WatchPatternInput): Promise<void> {
    await this._post(`${this.baseUrl}/addWatchPattern`, { caseId, tenancyId, ...input });
  }

  async removeWatchPattern(caseId: string, tenancyId: string, patternId: string): Promise<void> {
    await this._post(`${this.baseUrl}/removeWatchPattern`, { caseId, tenancyId, patternId });
  }

  async getGatePolicy(caseId: string, tenancyId: string): Promise<GatePolicy> {
    return this._get(`${this.baseUrl}/getGatePolicy?caseId=${caseId}&tenancyId=${enc(tenancyId)}`);
  }

  async setGatePolicy(caseId: string, tenancyId: string, policy: GatePolicy): Promise<void> {
    await this._post(`${this.baseUrl}/setGatePolicy`, { caseId, tenancyId, policy });
  }

  async getStages(caseId: string): Promise<StageDescriptor[]> {
    return this._get(`${this.baseUrl}/getStages?caseId=${caseId}`);
  }

  async getCategories(caseId: string): Promise<CategoryDescriptor[]> {
    return this._get(`${this.baseUrl}/getCategories?caseId=${caseId}`);
  }

  async getEvolutionState(caseId: string, tenancyId: string): Promise<EvolutionStateSnapshot> {
    return this._get(`${this.baseUrl}/getEvolutionState?caseId=${caseId}&tenancyId=${enc(tenancyId)}`);
  }

  async getStreamProgress(caseId: string, tenancyId: string): Promise<ImprovementStreamView[]> {
    return this._get(`${this.baseUrl}/getStreamProgress?caseId=${caseId}&tenancyId=${enc(tenancyId)}`);
  }

  async getInbox(caseId: string, tenancyId: string): Promise<ConductorInboxEntry[]> {
    return this._get(`${this.baseUrl}/getInbox?caseId=${caseId}&tenancyId=${enc(tenancyId)}`);
  }

  async resolveGate(caseId: string, tenancyId: string, entryId: string,
      outcome: GateOutcome, reason?: string, feedback?: string): Promise<void> {
    await this._post(`${this.baseUrl}/resolveGate`, {
      caseId, tenancyId, entryId, outcome, reason, feedback,
    });
  }

  async pauseCategory(caseId: string, tenancyId: string,
      category: string, durationMinutes: number): Promise<void> {
    await this._post(`${this.baseUrl}/pauseCategory`, {
      caseId, tenancyId, category, durationMinutes,
    });
  }

  async unpauseCategory(caseId: string, tenancyId: string, category: string): Promise<void> {
    await this._post(`${this.baseUrl}/unpauseCategory`, { caseId, tenancyId, category });
  }

  async blockImprovement(caseId: string, tenancyId: string,
      improvementId: string, blockedBy: string): Promise<void> {
    await this._post(`${this.baseUrl}/blockImprovement`, {
      caseId, tenancyId, improvementId, blockedBy,
    });
  }

  async unblockImprovement(caseId: string, tenancyId: string,
      improvementId: string): Promise<void> {
    await this._post(`${this.baseUrl}/unblockImprovement`, {
      caseId, tenancyId, improvementId,
    });
  }

  async resetCircuitBreaker(caseId: string): Promise<void> {
    await this._post(`${this.baseUrl}/resetCircuitBreaker`, { caseId });
  }

  private async _get<T>(url: string): Promise<T> {
    const res = await this.fetchFn(url, {
      method: 'GET', headers: { Accept: 'application/json' },
    });
    if (!res.ok) throw new Error(`${res.status}: ${res.statusText}`);
    return res.json();
  }

  private async _post(url: string, body: unknown): Promise<void> {
    const res = await this.fetchFn(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
    if (!res.ok) throw new Error(`${res.status}: ${res.statusText}`);
  }
}

function enc(s: string): string { return encodeURIComponent(s); }
```

- [ ] **Step 8: Write api.test.ts**

Test each API method with a mock fetchFn:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { EvolutionApi } from './api.js';
import type { DenyPatternView, GatePolicy } from './types.js';

function mockFetch(response: unknown, status = 200): typeof fetch {
  return vi.fn().mockResolvedValue({
    ok: status >= 200 && status < 300,
    status,
    statusText: status === 200 ? 'OK' : 'Error',
    json: () => Promise.resolve(response),
  });
}

describe('EvolutionApi', () => {
  it('getDenyPatterns sends GET with caseId and tenancyId', async () => {
    const view: DenyPatternView = { staticPatterns: ['Foo'], dynamicPatterns: [] };
    const fetch = mockFetch(view);
    const api = new EvolutionApi('/api/evolution', fetch);
    const result = await api.getDenyPatterns('case-1', 'tenant-1');
    expect(result).toEqual(view);
    expect(fetch).toHaveBeenCalledWith(
      expect.stringContaining('/getDenyPatterns?caseId=case-1&tenancyId=tenant-1'),
      expect.objectContaining({ method: 'GET' }),
    );
  });

  it('addDenyPattern sends POST with pattern', async () => {
    const fetch = mockFetch(undefined);
    const api = new EvolutionApi('/api/evolution', fetch);
    await api.addDenyPattern('case-1', 'tenant-1', 'SomeClass');
    expect(fetch).toHaveBeenCalledWith(
      expect.stringContaining('/addDenyPattern'),
      expect.objectContaining({
        method: 'POST',
        body: JSON.stringify({ caseId: 'case-1', tenancyId: 'tenant-1', pattern: 'SomeClass' }),
      }),
    );
  });

  it('resolveGate sends GateOutcome not InboxEntryStatus', async () => {
    const fetch = mockFetch(undefined);
    const api = new EvolutionApi('/api/evolution', fetch);
    await api.resolveGate('case-1', 'tenant-1', 'entry-1', 'APPROVED', 'looks good');
    const body = JSON.parse((fetch as ReturnType<typeof vi.fn>).mock.calls[0][1].body);
    expect(body.outcome).toBe('APPROVED');
    expect(body.reason).toBe('looks good');
  });

  it('setGatePolicy sends full GatePolicy object', async () => {
    const fetch = mockFetch(undefined);
    const api = new EvolutionApi('/api/evolution', fetch);
    const policy: GatePolicy = { modes: { 'pr-review': 'GATED' }, gateTimeoutMinutes: 720 };
    await api.setGatePolicy('case-1', 'tenant-1', policy);
    const body = JSON.parse((fetch as ReturnType<typeof vi.fn>).mock.calls[0][1].body);
    expect(body.policy).toEqual(policy);
  });

  it('throws on non-OK response', async () => {
    const fetch = mockFetch(undefined, 500);
    const api = new EvolutionApi('/api/evolution', fetch);
    await expect(api.getDenyPatterns('c', 't')).rejects.toThrow('500');
  });
});
```

- [ ] **Step 9: Write index.ts**

```typescript
export type * from './types.js';
export { EvolutionApi } from './api.js';
export { EvolutionEventTopics, emitEvolutionEvent } from './events.js';
```

(Editor element exports will be added incrementally in later tasks.)

- [ ] **Step 10: Install dependencies and verify**

Run: `yarn install`
Run: `yarn workspace @casehubio/blocks-ui-evolution-config run test`
Expected: All tests pass.

- [ ] **Step 11: Commit**

```bash
git add components/evolution-config/ tsconfig.json
git commit -m "feat(#175): scaffold evolution-config package — types, API, events

Refs casehubio/engine#1149"
```

---

## Batch 2: deny-pattern-editor (#175)

### Task 2: deny-pattern-editor component

**Files:**
- Create: `components/evolution-config/src/deny-pattern-editor.ts`
- Create: `components/evolution-config/src/deny-pattern-editor.test.ts`
- Modify: `components/evolution-config/src/index.ts` — add export

**Interfaces:**
- Consumes: `DenyPatternView`, `DynamicDenyEntry`, `EvolutionApi`, `emitEvolutionEvent`, `EvolutionEventTopics` from Task 1
- Produces: `DenyPatternEditor` class, `DenyPatternEditorProps` interface, `<blocks-deny-pattern-editor>` custom element

- [ ] **Step 1: Write the test file**

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { html, fixture, fixtureCleanup } from '@open-wc/testing-helpers';
import type { DenyPatternView } from './types.js';
import './deny-pattern-editor.js';
import type { DenyPatternEditor } from './deny-pattern-editor.js';

const SAMPLE_VIEW: DenyPatternView = {
  staticPatterns: ['ImprovementBudgetEnforcer', 'EvolutionTicker'],
  dynamicPatterns: [
    { pattern: 'AuthController', addedBy: 'admin', addedAt: '2026-09-25T12:00:00Z' },
    { pattern: 'PaymentService', addedBy: 'admin', addedAt: '2026-09-26T08:00:00Z' },
  ],
};

describe('blocks-deny-pattern-editor', () => {
  afterEach(() => fixtureCleanup());

  it('has correct ARIA attributes', async () => {
    const el = await fixture<DenyPatternEditor>(
      html`<blocks-deny-pattern-editor .patterns=${SAMPLE_VIEW}></blocks-deny-pattern-editor>`
    );
    expect(el.getAttribute('role')).toBe('region');
    expect(el.getAttribute('aria-label')).toBe('Deny pattern editor');
  });

  it('renders structural patterns as read-only list', async () => {
    const el = await fixture<DenyPatternEditor>(
      html`<blocks-deny-pattern-editor .patterns=${SAMPLE_VIEW}></blocks-deny-pattern-editor>`
    );
    const items = el.shadowRoot!.querySelectorAll('[role="listitem"]');
    expect(items.length).toBe(2);
    expect(items[0]!.textContent).toContain('ImprovementBudgetEnforcer');
  });

  it('renders dynamic patterns in table', async () => {
    const el = await fixture<DenyPatternEditor>(
      html`<blocks-deny-pattern-editor .patterns=${SAMPLE_VIEW}></blocks-deny-pattern-editor>`
    );
    const table = el.shadowRoot!.querySelector('pages-table');
    expect(table).not.toBeNull();
  });

  it('hides add form and delete buttons in readonly mode', async () => {
    const el = await fixture<DenyPatternEditor>(
      html`<blocks-deny-pattern-editor .patterns=${SAMPLE_VIEW} ?readonly=${true}></blocks-deny-pattern-editor>`
    );
    const form = el.shadowRoot!.querySelector('[role="form"]');
    expect(form).toBeNull();
    const deleteButtons = el.shadowRoot!.querySelectorAll('button[aria-label*="Delete"]');
    expect(deleteButtons.length).toBe(0);
  });

  it('emits evolution:deny-pattern-changed on add in inline mode', async () => {
    const el = await fixture<DenyPatternEditor>(
      html`<blocks-deny-pattern-editor .patterns=${SAMPLE_VIEW}></blocks-deny-pattern-editor>`
    );
    const handler = vi.fn();
    el.addEventListener('pages-event', handler);
    // Trigger add via component method (tested as unit test)
    (el as any)._handleAdd('NewPattern');
    expect(handler).toHaveBeenCalledOnce();
    const detail = handler.mock.calls[0][0].detail;
    expect(detail.topic).toBe('evolution:deny-pattern-changed');
    expect(detail.payload.action).toBe('add');
    expect(detail.payload.pattern).toBe('NewPattern');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config run test`
Expected: FAIL — module `./deny-pattern-editor.js` not found

- [ ] **Step 3: Implement deny-pattern-editor.ts**

Full implementation following the mute-list pattern. Two sections: structural (read-only `<ul>`) and dynamic (pages-table with add form and delete). Uses `fromRows()` for table data, `PagesConfirmDialog` for delete confirmation. See spec §5 for column definitions (Pattern 1fr, Added by 120px, Added at 160px, Actions 60px).

Column definitions follow the mute-list pattern with `columnId()`, `ColumnType`, and `getValue` extractors. Column renderers use inline styles per protocol PP-20260713-8ea1af.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config run test`
Expected: All tests pass.

- [ ] **Step 5: Add export to index.ts**

```typescript
export { DenyPatternEditor } from './deny-pattern-editor.js';
export type { DenyPatternEditorProps } from './deny-pattern-editor.js';
```

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: No errors.

- [ ] **Step 7: Commit**

```bash
git add components/evolution-config/src/deny-pattern-editor.ts components/evolution-config/src/deny-pattern-editor.test.ts components/evolution-config/src/index.ts
git commit -m "feat(#175): deny-pattern-editor — static + dynamic deny pattern management

Closes casehubio/blocks-ui#175
Refs casehubio/engine#1149"
```

---

## Batch 3: watch-pattern-editor (#176)

### Task 3: watch-pattern-editor component

**Files:**
- Create: `components/evolution-config/src/watch-pattern-editor.ts`
- Create: `components/evolution-config/src/watch-pattern-editor.test.ts`
- Modify: `components/evolution-config/src/index.ts` — add export

**Interfaces:**
- Consumes: `WatchPattern`, `WatchPatternInput`, `CategoryDescriptor`, `EvolutionApi`, `emitEvolutionEvent`, `EvolutionEventTopics` from Task 1
- Produces: `WatchPatternEditor` class, `WatchPatternEditorProps` interface, `<blocks-watch-pattern-editor>` custom element

- [ ] **Step 1: Write the test file**

Test ARIA, inline data rendering (table with 6 columns: Category, Area, Target pattern, Min size, Created, Actions), add form validation (at least one field non-empty), delete confirmation, readonly mode, event emission with `{ action: 'add' | 'remove', input?: WatchPatternInput, patternId?: string }` payload.

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config run test`
Expected: FAIL

- [ ] **Step 3: Implement watch-pattern-editor.ts**

Same structure as deny-pattern-editor but single section (no structural/dynamic split). Uses pages-form (pages-viz) with JSON Schema for add form: category (optional select from `categories` prop), areaId (optional text), targetPattern (optional text), minEstimatedSize (optional number). Add form validates at least one field is non-empty. See spec §6.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config run test`
Expected: All tests pass.

- [ ] **Step 5: Add export to index.ts and commit**

```bash
git add components/evolution-config/src/watch-pattern-editor.ts components/evolution-config/src/watch-pattern-editor.test.ts components/evolution-config/src/index.ts
git commit -m "feat(#176): watch-pattern-editor — escalation watch pattern CRUD

Closes casehubio/blocks-ui#176
Refs casehubio/engine#1149"
```

---

## Batch 4: gate-policy-editor (#177)

### Task 4: gate-policy-editor component

**Files:**
- Create: `components/evolution-config/src/gate-policy-editor.ts`
- Create: `components/evolution-config/src/gate-policy-editor.test.ts`
- Modify: `components/evolution-config/src/index.ts` — add export

**Interfaces:**
- Consumes: `StageDescriptor`, `GatePolicy`, `GateMode`, `EvolutionApi`, `emitEvolutionEvent`, `EvolutionEventTopics` from Task 1
- Produces: `GatePolicyEditor` class, `GatePolicyEditorProps` interface, `<blocks-gate-policy-editor>` custom element

- [ ] **Step 1: Write the test file**

Test: ARIA (role="form"), domain grouping (stages sorted by domainId then ordinal), mode dropdown constraint (GATED only for gateCheckpoint stages, AUTO/NOTIFY for others), dirty state tracking (Save button disabled until changes), batch save (emits full GatePolicy object), gate timeout input, readonly mode.

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement gate-policy-editor.ts**

Grouped table with `<select>` dropdowns per row. Groups stages by `domainId` with domain header rows. Tracks pending changes in a local `Map<string, GateMode>`. Save button enabled only when dirty. On save in endpoint mode: calls `setGatePolicy()`. On save in inline mode: emits `evolution:gate-policy-changed` with the new GatePolicy. See spec §7.

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Add export to index.ts and commit**

```bash
git add components/evolution-config/src/gate-policy-editor.ts components/evolution-config/src/gate-policy-editor.test.ts components/evolution-config/src/index.ts
git commit -m "feat(#177): gate-policy-editor — per-stage GATED/AUTO/NOTIFY configuration

Closes casehubio/blocks-ui#177
Refs casehubio/engine#1149"
```

---

## Batch 5: detail-pane standalone mode + evolution-workbench (#178)

### Task 5: Add standalone mode to detail-pane

**Files:**
- Modify: `components/detail-pane/src/detail-pane.ts` — add `standalone` property, adjust render guard
- Modify: `components/detail-pane/src/types.ts` — add `standalone` to props if needed
- Modify: `components/detail-pane/src/detail-pane.test.ts` — add standalone mode tests

**Interfaces:**
- Consumes: existing `DetailPane` class
- Produces: `standalone` property on `DetailPane` — when true, tabs render without requiring a selection event

- [ ] **Step 1: Write failing test for standalone mode**

```typescript
it('renders tabs in standalone mode without selection', async () => {
  const tabs: TabDefinition[] = [
    { id: 'tab1', label: 'First', tagName: 'div' },
    { id: 'tab2', label: 'Second', tagName: 'div' },
  ];
  const el = await fixture<DetailPane>(
    html`<blocks-detail-pane .tabs=${tabs} ?standalone=${true}></blocks-detail-pane>`
  );
  const tabButtons = el.shadowRoot!.querySelectorAll('[role="tab"]');
  expect(tabButtons.length).toBe(2);
  const empty = el.shadowRoot!.querySelector('.empty');
  expect(empty).toBeNull();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/blocks-ui-detail-pane run test`
Expected: FAIL — standalone not recognised, renders empty state

- [ ] **Step 3: Add standalone property and modify render guard**

In `detail-pane.ts`:
- Add `@property({ type: Boolean }) standalone = false;`
- Modify `render()`: change `if (!this._item)` to `if (!this._item && !this.standalone)`
- In `connectedCallback`: when `standalone` is true, set `this._item = {}` and activate first tab

- [ ] **Step 4: Run tests to verify they pass (existing + new)**

Run: `yarn workspace @casehubio/blocks-ui-detail-pane run test`
Expected: All tests pass (existing behaviour unchanged, standalone mode works).

- [ ] **Step 5: Commit**

```bash
git add components/detail-pane/
git commit -m "feat(#178): add standalone mode to detail-pane — tabs render without selection

Refs casehubio/engine#1149"
```

### Task 6: Scaffold evolution-workbench package and implement workbench

**Files:**
- Create: `components/evolution-workbench/package.json`
- Create: `components/evolution-workbench/tsconfig.json`
- Create: `components/evolution-workbench/tsconfig.build.json`
- Create: `components/evolution-workbench/src/evolution-workbench.ts`
- Create: `components/evolution-workbench/src/evolution-workbench.test.ts`
- Create: `components/evolution-workbench/src/index.ts`
- Modify: `tsconfig.json` (root) — add project reference

**Interfaces:**
- Consumes: All types from Task 1, `DenyPatternEditor` from Task 2, `WatchPatternEditor` from Task 3, `GatePolicyEditor` from Task 4, `DetailPane` (standalone mode) from Task 5, `kpi-metric-row`, `blocks-timeline`, `compliance-summary`, `pages-table`
- Produces: `EvolutionWorkbench` class, `EvolutionWorkbenchProps`, `<blocks-evolution-workbench>` custom element

- [ ] **Step 1: Scaffold package** (same pattern as Task 1 step 1-3)

Package name: `@casehubio/blocks-ui-evolution-workbench`. Dependencies include `@casehubio/blocks-ui-evolution-config`, `@casehubio/blocks-ui-kpi-metric-row`, `@casehubio/blocks-ui-detail-pane`, `@casehubio/blocks-ui-compliance-summary`. Add root tsconfig reference.

- [ ] **Step 2: Write evolution-workbench.test.ts**

Test: ARIA (role="region", aria-label), summary bar renders kpi-metric-row, tabs render via detail-pane standalone mode, inline data mode renders all tabs, configure() method works, event-driven refresh on child editor changes.

- [ ] **Step 3: Run tests to verify they fail**

- [ ] **Step 4: Implement evolution-workbench.ts**

Summary bar: `kpi-metric-row` configured with 5 metrics from `EvolutionStateSnapshot`. Tabbed content: `detail-pane` with `standalone` mode and 5 `TabDefinition` entries (Timeline, Streams, Inbox, Compliance, Configuration). Configuration tab renders the three editors stacked vertically. Domain-extensible via `tabs` property merged with built-in tabs. Three consumption tiers. `configure()` method. EventStreamController for real-time updates when `pushUrl` is set. See spec §8.

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Write index.ts, add root tsconfig reference, commit**

```bash
git add components/evolution-workbench/ tsconfig.json
git commit -m "feat(#178): evolution-workbench — dashboard with summary bar, tabs, real-time updates

Closes casehubio/blocks-ui#178
Refs casehubio/engine#1149"
```

---

## Batch 6: Sample page + schema registry + engine issues (#179)

### Task 7: Sample page, schema registration, and engine issue filing

**Files:**
- Create: `components/evolution-workbench/sample/index.html`
- Create: `components/evolution-workbench/src/sample-data.ts`
- Modify: `packages/blocks-ui-schema/src/registry.ts` — add 4 component entries
- Modify: `packages/blocks-ui-schema/package.json` — add devDependencies

**Interfaces:**
- Consumes: `EvolutionWorkbench`, all types from Task 1
- Produces: Sample page at `sample/index.html`, `BlocksComponentRegistry` entries for all 4 new elements

- [ ] **Step 1: Create sample-data.ts**

Export a complete inline data set: `EvolutionStateSnapshot` with component scores, `ImprovementStreamView[]` (3 streams — one active, one blocked, one at gate), `ConductorInboxEntry[]` (4 entries — PENDING, APPROVED, TIMED_OUT, AUTO_APPROVED), `DenyPatternView` (2 structural + 2 dynamic), `WatchPattern[]` (3 patterns), `StageDescriptor[]` (code-evolution 11 stages), `CategoryDescriptor[]` (5 code-evolution categories), `GatePolicy` (mixed modes). Include a sample `TabDefinition` for a mock trading risk panel demonstrating domain extension.

- [ ] **Step 2: Create sample/index.html**

HTML page that imports the workbench element and sample data, instantiates `<blocks-evolution-workbench>` with all inline data properties. Include a `<sample-trading-risk-panel>` custom element showing mock trading metrics.

- [ ] **Step 3: Add schema registry entries**

In `packages/blocks-ui-schema/src/registry.ts`, add import and registry entries:

```typescript
import type { DenyPatternEditorProps, WatchPatternEditorProps, GatePolicyEditorProps } from '@casehubio/blocks-ui-evolution-config';
import type { EvolutionWorkbenchProps } from '@casehubio/blocks-ui-evolution-workbench';
```

Add to `BlocksComponentRegistry`:
```typescript
'blocks-deny-pattern-editor': DenyPatternEditorProps;
'blocks-watch-pattern-editor': WatchPatternEditorProps;
'blocks-gate-policy-editor': GatePolicyEditorProps;
'blocks-evolution-workbench': EvolutionWorkbenchProps;
```

Add `workspace:*` devDependencies to `packages/blocks-ui-schema/package.json`.

- [ ] **Step 4: Regenerate schemas**

Run: `yarn workspace @casehubio/blocks-ui-schema run generate`
Expected: `component-schemas.generated.ts` updated with 4 new schemas.

- [ ] **Step 5: Run full test suite**

Run: `yarn test`
Run: `yarn typecheck`
Expected: All pass.

- [ ] **Step 6: File engine issues for API gaps**

Use `gh issue create` for each:

1. `feat: expose getWatchPatterns via EvolutionMcpAdapter` (casehubio/engine)
2. `feat: expose getStages and getCategories via EvolutionMcpAdapter` (casehubio/engine)
3. `feat: expose getGatePolicy via EvolutionMcpAdapter` (casehubio/engine)

- [ ] **Step 7: File deferred blocks-ui issues**

1. `feat: deny-pattern testing/preview against active streams` (casehubio/blocks-ui)
2. `feat: gate-policy impact preview` (casehubio/blocks-ui)
3. `feat: evolution audit trail tab` (casehubio/blocks-ui)
4. `feat: health detail tab with trust-score-panel` (casehubio/blocks-ui)
5. `feat: evolution workbench responsive layout` (casehubio/blocks-ui)

- [ ] **Step 8: Update CLAUDE.md with new component entries**

Add `evolution-config/` and `evolution-workbench/` to the Key Directories table in CLAUDE.md.

- [ ] **Step 9: Commit**

```bash
git add components/evolution-workbench/sample/ components/evolution-workbench/src/sample-data.ts packages/blocks-ui-schema/ CLAUDE.md
git commit -m "feat(#179): sample page, schema registration, engine issues filed

Closes casehubio/blocks-ui#179
Refs casehubio/engine#1149"
```

---

## References

- `specs/issue-1149-evolution-ui/2026-09-26-evolution-conductor-ui-design.md` — design spec this plan implements
- `specs/issue-1149-evolution-ui/decisions.md` — D1–D5 design decisions
- `components/notification-inbox/` — precedent for multi-element package structure, api.ts, events.ts, types.ts
- `components/notification-inbox/src/mute-list.ts` — CRUD table + inline add form + delete confirmation pattern
- `components/detail-pane/src/detail-pane.ts:176-179` — item guard requiring standalone mode enhancement
- `components/detail-pane/src/types.ts` — TabDefinition interface
- `components/preferences-editor/src/api.ts` — API class with injectable fetchFn
- `packages/blocks-ui-schema/src/registry.ts` — BlocksComponentRegistry pattern
- `EngineEvolutionApi.java` — engine API (19 methods)
- `EvolutionMcpAdapter.java` — REST endpoint exposure
- `ConductorInboxEntry.java` — 14-field inbox entry
- PP-20260713-8ea1af — component customisation protocol
- casehubio/blocks-ui#174 (epic), #175-#179 (issues)
- casehubio/engine#1149 (parent epic)
