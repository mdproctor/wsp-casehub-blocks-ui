# Evolution Conductor UI — Operational Completeness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #188 — epic: Evolution conductor UI — operational completeness
**Issue group:** #186, #187, #181, #182, #183, #184, #185

**Goal:** Make the evolution workbench operationally complete — real data tables for streams and inbox, audit/health tabs, editor previews, responsive layout.

**Architecture:** Extend `TabDefinition` with an optional `renderContent` callback (PP-20260713-8ea1af mechanism #2). Create two standalone domain components (`blocks-evolution-streams`, `blocks-evolution-inbox`) in the evolution-config package. Refactor the workbench to use render callbacks for all built-in tabs. Add audit/health tabs as inline compositions of existing components. Extend editors with stream-aware previews. Apply CSS container queries for responsiveness.

**Tech Stack:** Lit 3, TypeScript, pages-table, pages-data (fromRows, columnId, ColumnType), vitest, CSS container queries

## Global Constraints

- All components must have ARIA attributes (role, aria-label, aria-busy). Tests must assert ARIA.
- Column renderers use inline styles (PP-20260713-8ea1af rule #4 — shadow DOM boundary).
- New `blocks-*` components export a Props interface and register in `BlocksComponentRegistry` (PP-20260907-fd8ee7).
- Run `yarn workspace @casehubio/blocks-ui-schema run generate` after registry changes.
- Dual data mode: every component works with inline data (property) AND endpoint fetch.
- Use `ide_create_file` for new `.ts` files, `ide_edit_member`/`ide_replace_member`/`ide_insert_member` for structural edits.

---

## Batch 1: Foundation — detail-pane renderContent + Streams component

### Task 1: Extend TabDefinition with renderContent callback

**Files:**
- Modify: `components/detail-pane/src/types.ts`
- Modify: `components/detail-pane/src/detail-pane.ts:142-149` (`_getOrCreateTabElement` and render)
- Test: `components/detail-pane/src/detail-pane.test.ts`

**Interfaces:**
- Produces: `TabDefinition.renderContent?: (item: unknown) => TemplateResult` — all subsequent tasks depend on this

- [ ] **Step 1: Write failing test for renderContent callback**

Add a test to `detail-pane.test.ts` after the existing tests:

```typescript
import { html, type TemplateResult } from 'lit';

it('renders content from renderContent callback when present', async () => {
  el = document.createElement('blocks-detail-pane') as DetailPaneEl;
  el.tabs = [
    {
      id: 'custom', label: 'Custom', tagName: 'div', order: 0,
      renderContent: () => html`<div class="callback-content">Rendered via callback</div>` as TemplateResult,
    },
  ];
  (el as any).standalone = true;
  document.body.appendChild(el);
  await el.updateComplete;

  const panel = el.shadowRoot!.querySelector('[role="tabpanel"]');
  expect(panel!.innerHTML).toContain('callback-content');
  expect(panel!.textContent).toContain('Rendered via callback');
});

it('falls back to tagName when renderContent is absent', async () => {
  el = document.createElement('blocks-detail-pane') as DetailPaneEl;
  el.tabs = [
    { id: 'fallback', label: 'Fallback', tagName: 'test-tab-panel', order: 0 },
  ];
  (el as any).standalone = true;
  document.body.appendChild(el);
  await el.updateComplete;

  const panel = el.shadowRoot!.querySelector('[role="tabpanel"]');
  const child = panel!.querySelector('test-tab-panel');
  expect(child).not.toBeNull();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/blocks-ui-detail-pane test`
Expected: first test FAILS (renderContent not recognized, callback not called)

- [ ] **Step 3: Add renderContent to TabDefinition**

In `components/detail-pane/src/types.ts`, use `ide_edit_member` on `TabDefinition`:

```typescript
export interface TabDefinition {
  id: string;
  label: string;
  tagName: string;
  icon?: string;
  order?: number;
  badge?: (item: unknown) => string | null;
  renderContent?: (item: unknown) => unknown;
}
```

- [ ] **Step 4: Update detail-pane render to use renderContent**

In `components/detail-pane/src/detail-pane.ts`, use `ide_replace_member` on the `render` method. Replace the `activeElement` computation:

```typescript
override render() {
  if (!this._item && !this.standalone) {
    return html`<div class="empty" role="status">${this.emptyMessage}</div>`;
  }

  const sorted = this._sortedTabs;
  const activeTab = this._activeTab;
  const activeContent = activeTab?.renderContent
    ? activeTab.renderContent(this._item)
    : (activeTab ? this._getOrCreateTabElement(activeTab) : null);

  return html`
    <div class="tab-bar" role="tablist" @keydown=${this._handleTabKeyDown}>
      ${sorted.map(tab => {
        const isActive = tab.id === this._activeTabId;
        const badgeValue = tab.badge?.(this._item);
        return html`
          <button class="tab-button"
                  role="tab"
                  aria-selected="${isActive}"
                  aria-controls="panel-${tab.id}"
                  tabindex="${isActive ? 0 : -1}"
                  @click=${() => this._handleTabClick(tab.id)}>
            ${tab.label}
            ${badgeValue ? html`<span class="badge">${badgeValue}</span>` : nothing}
          </button>
        `;
      })}
    </div>
    <div class="tab-panel"
         role="tabpanel"
         id="panel-${this._activeTabId}"
         aria-labelledby="tab-${this._activeTabId}"
         tabindex="0">
      ${activeContent}
    </div>
  `;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/blocks-ui-detail-pane test`
Expected: ALL tests PASS (new + existing)

- [ ] **Step 6: Run full typecheck**

Run: `yarn typecheck`
Expected: no errors

- [ ] **Step 7: Commit**

```bash
git add components/detail-pane/
git commit -m "feat(#188): add renderContent callback to TabDefinition

Extends detail-pane's TabDefinition with an optional renderContent
callback (PP-20260713-8ea1af mechanism #2). When present, detail-pane
renders the callback output instead of createElement(tagName).
Backward-compatible — existing consumers unaffected. Refs #188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Create blocks-evolution-streams component

**Files:**
- Create: `components/evolution-config/src/evolution-streams.ts`
- Create: `components/evolution-config/src/evolution-streams.test.ts`
- Modify: `components/evolution-config/src/index.ts`
- Modify: `components/evolution-config/src/events.ts`

**Interfaces:**
- Consumes: `ImprovementStreamView` from `types.ts`, `EvolutionApi` from `api.ts`
- Produces: `EvolutionStreamsProps { endpoint?: string; caseId?: string; tenancyId?: string; streams?: readonly ImprovementStreamView[]; readonly?: boolean }`, tag `blocks-evolution-streams`, event `evolution:stream-changed`

- [ ] **Step 1: Write failing tests**

Create `components/evolution-config/src/evolution-streams.test.ts`:

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';
import type { ImprovementStreamView } from './types.js';
import './evolution-streams.js';
import type { EvolutionStreams } from './evolution-streams.js';

function createElement(streams?: ImprovementStreamView[]): EvolutionStreams {
  const el = document.createElement('blocks-evolution-streams') as EvolutionStreams;
  if (streams) el.streams = streams;
  document.body.appendChild(el);
  return el;
}

const SAMPLE_STREAMS: ImprovementStreamView[] = [
  {
    improvementCaseId: '660e8400-0001-0000-0000-000000000001',
    category: 'dependency-update',
    target: 'quarkus-core',
    currentStage: 'implement',
    blockedBy: null,
    conflictBlocked: false,
    startedAt: '2026-09-26T06:00:00Z',
  },
  {
    improvementCaseId: '660e8400-0001-0000-0000-000000000002',
    category: 'lint-fix',
    target: 'auth-module',
    currentStage: 'pr-review',
    blockedBy: '660e8400-0001-0000-0000-000000000001',
    conflictBlocked: true,
    startedAt: '2026-09-26T07:00:00Z',
  },
];

describe('blocks-evolution-streams', () => {
  afterEach(() => {
    document.body.querySelectorAll('blocks-evolution-streams').forEach(el => el.remove());
  });

  it('has correct ARIA attributes', async () => {
    const el = createElement(SAMPLE_STREAMS);
    await el.updateComplete;
    expect(el.getAttribute('role')).toBe('region');
    expect(el.getAttribute('aria-label')).toBe('Improvement streams');
  });

  it('renders a pages-table with stream data', async () => {
    const el = createElement(SAMPLE_STREAMS);
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-table');
    expect(table).not.toBeNull();
  });

  it('shows empty state when no streams', async () => {
    const el = createElement([]);
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('No active improvement streams');
  });

  it('hides action buttons in readonly mode', async () => {
    const el = createElement(SAMPLE_STREAMS);
    el.readonly = true;
    await el.updateComplete;
    const buttons = el.shadowRoot!.querySelectorAll('button[aria-label]');
    const actionButtons = Array.from(buttons).filter(b =>
      b.getAttribute('aria-label')?.includes('Block') ||
      b.getAttribute('aria-label')?.includes('Unblock')
    );
    expect(actionButtons.length).toBe(0);
  });

  it('renders blocked status indicator', async () => {
    const el = createElement(SAMPLE_STREAMS);
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-table');
    expect(table).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config test -- --reporter=verbose evolution-streams`
Expected: FAIL (module not found)

- [ ] **Step 3: Implement evolution-streams component**

Create `components/evolution-config/src/evolution-streams.ts` using `ide_create_file`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import '@casehubio/pages-table';
import type { ColumnRenderer } from '@casehubio/pages-table';
import { fromRows } from '@casehubio/pages-data/dist/dataset/conversion.js';
import { columnId, ColumnType } from '@casehubio/pages-data/dist/dataset/types.js';
import type { CellValue, ColumnId, TypedRow } from '@casehubio/pages-data/dist/dataset/types.js';
import type { ImprovementStreamView } from './types.js';
import { EvolutionApi } from './api.js';
import { emitEvolutionEvent } from './events.js';

const ID_COL = columnId('improvementCaseId');
const CATEGORY_COL = columnId('category');
const TARGET_COL = columnId('target');
const STAGE_COL = columnId('currentStage');
const BLOCKED_COL = columnId('blockedBy');
const CONFLICT_COL = columnId('conflictBlocked');
const STARTED_COL = columnId('startedAt');
const ACTIONS_COL = columnId('actions');

const COL_DEFS = [
  { id: ID_COL, type: ColumnType.TEXT, getValue: (s: ImprovementStreamView) => s.improvementCaseId },
  { id: CATEGORY_COL, name: 'Category', type: ColumnType.TEXT, getValue: (s: ImprovementStreamView) => s.category },
  { id: TARGET_COL, name: 'Target', type: ColumnType.TEXT, getValue: (s: ImprovementStreamView) => s.target ?? '' },
  { id: STAGE_COL, name: 'Stage', type: ColumnType.TEXT, getValue: (s: ImprovementStreamView) => s.currentStage },
  { id: BLOCKED_COL, name: 'Blocked', type: ColumnType.TEXT, getValue: (s: ImprovementStreamView) => s.blockedBy ?? '' },
  { id: CONFLICT_COL, name: 'Conflict', type: ColumnType.TEXT, getValue: (s: ImprovementStreamView) => s.conflictBlocked ? 'Yes' : '' },
  { id: STARTED_COL, name: 'Started', type: ColumnType.TEXT, getValue: (s: ImprovementStreamView) => s.startedAt },
  { id: ACTIONS_COL, type: ColumnType.TEXT, getValue: () => '' },
] as const;

const COL_CONFIG = [
  { id: ID_COL, visible: false },
  { id: CATEGORY_COL, sortable: true, width: '120px' },
  { id: TARGET_COL, sortable: true, width: '1fr' },
  { id: STAGE_COL, sortable: true, width: '140px' },
  { id: BLOCKED_COL, sortable: false, width: '100px' },
  { id: CONFLICT_COL, sortable: false, width: '80px' },
  { id: STARTED_COL, sortable: true, width: '140px' },
  { id: ACTIONS_COL, sortable: false, width: '100px' },
];

export interface EvolutionStreamsProps {
  endpoint?: string;
  caseId?: string;
  tenancyId?: string;
  streams?: readonly ImprovementStreamView[];
  readonly?: boolean;
}

@customElement('blocks-evolution-streams')
export class EvolutionStreams extends LitElement {
  @property({ type: String }) endpoint?: string;
  @property({ type: String }) caseId?: string;
  @property({ type: String }) tenancyId?: string;
  @property({ type: Array, attribute: false }) streams?: readonly ImprovementStreamView[];
  @property({ type: Boolean }) readonly = false;

  @state() private _loading = false;
  @state() private _error: string | null = null;
  @state() private _fetched: ImprovementStreamView[] | null = null;

  private _api?: EvolutionApi;

  private get _data(): readonly ImprovementStreamView[] {
    return this.streams ?? this._fetched ?? [];
  }

  private _columnRenderers: ReadonlyMap<ColumnId, ColumnRenderer> = new Map<ColumnId, ColumnRenderer>([
    [CATEGORY_COL, (cell: CellValue) => {
      const val = cell.type === 'NULL' ? '' : (cell as { value: string }).value;
      return html`<span style="display:inline-block;padding:2px 8px;border-radius:10px;font-size:12px;font-weight:500;background:var(--pages-accent-3,#dbeafe);color:var(--pages-accent-11,#1e40af)">${val}</span>`;
    }],
    [TARGET_COL, (cell: CellValue) => {
      const val = cell.type === 'NULL' ? '' : (cell as { value: string }).value;
      if (!val) return html`<span style="color:var(--pages-neutral-7,#737373);font-style:italic">—</span>`;
      return html`<code style="font-family:var(--pages-font-mono,monospace);font-size:13px">${val}</code>`;
    }],
    [BLOCKED_COL, (cell: CellValue) => {
      const val = cell.type === 'NULL' ? '' : (cell as { value: string }).value;
      if (!val) return nothing;
      return html`<span style="display:inline-flex;align-items:center;gap:4px;color:var(--pages-warning-11,#92400e);font-size:12px">⊘ Blocked</span>`;
    }],
    [CONFLICT_COL, (cell: CellValue) => {
      const val = cell.type === 'NULL' ? '' : (cell as { value: string }).value;
      if (!val) return nothing;
      return html`<span style="display:inline-block;padding:2px 6px;border-radius:10px;font-size:11px;background:var(--pages-warning-3,#fef3c7);color:var(--pages-warning-11,#92400e)">Conflict</span>`;
    }],
    [STARTED_COL, (cell: CellValue) => {
      if (cell.type === 'NULL' || !(cell as { value: string }).value) return '';
      const d = new Date((cell as { value: string }).value);
      return html`<span style="font-size:12px;color:var(--pages-neutral-9,#737373)">${d.toLocaleString()}</span>`;
    }],
    [ACTIONS_COL, (_cell: CellValue, row: TypedRow) => {
      if (this.readonly) return nothing;
      const blocked = row.text(BLOCKED_COL);
      const id = row.text(ID_COL);
      if (blocked) {
        return html`<button style="padding:4px 10px;border:none;border-radius:4px;font-size:12px;font-weight:500;cursor:pointer;background:var(--pages-success-3,#dcfce7);color:var(--pages-success-11,#166534)" aria-label="Unblock ${id}" @click=${(e: Event) => { e.stopPropagation(); this._handleUnblock(id); }}>Unblock</button>`;
      }
      return html`<button style="padding:4px 10px;border:none;border-radius:4px;font-size:12px;font-weight:500;cursor:pointer;background:var(--pages-warning-3,#fef3c7);color:var(--pages-warning-11,#92400e)" aria-label="Block ${id}" @click=${(e: Event) => { e.stopPropagation(); this._handleBlock(id); }}>Block</button>`;
    }],
  ]);

  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui); }
    .empty { padding: 24px; text-align: center; color: var(--pages-neutral-9, #737373); }
    .loading { padding: 24px; text-align: center; color: var(--pages-neutral-9, #737373); }
    .error { padding: 16px; background: var(--pages-danger-3, #fee); color: var(--pages-danger-11, #c00); border-radius: 4px; }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this.setAttribute('role', 'region');
    this.setAttribute('aria-label', 'Improvement streams');
    if (this.endpoint && !this._api) {
      this._api = new EvolutionApi(this.endpoint);
    }
    if (this._api && this.caseId && this.tenancyId && !this.streams) {
      this._fetch();
    }
  }

  private async _fetch(): Promise<void> {
    if (!this._api || !this.caseId || !this.tenancyId) return;
    this._loading = true;
    this._error = null;
    try {
      this._fetched = await this._api.getStreamProgress(this.caseId, this.tenancyId);
    } catch (e) {
      this._error = e instanceof Error ? e.message : 'Failed to load streams';
    } finally {
      this._loading = false;
    }
  }

  private async _handleBlock(improvementId: string): Promise<void> {
    if (!this._api || !this.caseId || !this.tenancyId) return;
    const blockedBy = prompt('Enter blocking improvement ID:');
    if (!blockedBy) return;
    try {
      await this._api.blockImprovement(this.caseId, this.tenancyId, improvementId, blockedBy);
      emitEvolutionEvent(this, 'evolution:stream-changed', { action: 'block', improvementId });
      await this._fetch();
    } catch (e) {
      this._error = e instanceof Error ? e.message : 'Failed to block improvement';
    }
  }

  private async _handleUnblock(improvementId: string): Promise<void> {
    if (!this._api || !this.caseId || !this.tenancyId) return;
    try {
      await this._api.unblockImprovement(this.caseId, this.tenancyId, improvementId);
      emitEvolutionEvent(this, 'evolution:stream-changed', { action: 'unblock', improvementId });
      await this._fetch();
    } catch (e) {
      this._error = e instanceof Error ? e.message : 'Failed to unblock improvement';
    }
  }

  override render() {
    this.setAttribute('aria-busy', String(this._loading));
    if (this._loading) return html`<div class="loading">Loading streams...</div>`;
    if (this._error) return html`<div class="error">${this._error}</div>`;
    const data = this._data;
    if (data.length === 0) return html`<div class="empty">No active improvement streams.</div>`;

    return html`
      <pages-table
        .dataSet=${fromRows([...data], COL_DEFS)}
        .columnConfig=${COL_CONFIG}
        .columnRenderers=${this._columnRenderers}
        .getRowKey=${(row: TypedRow) => row.text(ID_COL)}
        mode="scroll"
        selection="none"
      ></pages-table>
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'blocks-evolution-streams': EvolutionStreams;
  }
}
```

- [ ] **Step 4: Add export to index.ts**

In `components/evolution-config/src/index.ts`, add:

```typescript
export { EvolutionStreams } from './evolution-streams.js';
export type { EvolutionStreamsProps } from './evolution-streams.js';
```

- [ ] **Step 5: Add stream-changed event topic to events.ts**

In `components/evolution-config/src/events.ts`, add to `EvolutionEventTopics`:

```typescript
STREAM_CHANGED: 'evolution:stream-changed',
```

- [ ] **Step 6: Run tests**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config test -- --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 7: Run typecheck**

Run: `yarn typecheck`
Expected: no errors

- [ ] **Step 8: Commit**

```bash
git add components/evolution-config/
git commit -m "feat(#186): add blocks-evolution-streams component

pages-table for ImprovementStreamView with block/unblock actions,
category badges, conflict indicators. Dual data mode. Refs #186

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Inbox component + workbench refactor

### Task 3: Create blocks-evolution-inbox component

**Files:**
- Create: `components/evolution-config/src/evolution-inbox.ts`
- Create: `components/evolution-config/src/evolution-inbox.test.ts`
- Modify: `components/evolution-config/src/index.ts`

**Interfaces:**
- Consumes: `ConductorInboxEntry`, `GateOutcome` from `types.ts`, `EvolutionApi.resolveGate()` from `api.ts`
- Produces: `EvolutionInboxProps { endpoint?: string; caseId?: string; tenancyId?: string; inbox?: readonly ConductorInboxEntry[]; readonly?: boolean }`, tag `blocks-evolution-inbox`

- [ ] **Step 1: Write failing tests**

Create `components/evolution-config/src/evolution-inbox.test.ts`:

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';
import type { ConductorInboxEntry } from './types.js';
import './evolution-inbox.js';
import type { EvolutionInbox } from './evolution-inbox.js';

function createElement(inbox?: ConductorInboxEntry[]): EvolutionInbox {
  const el = document.createElement('blocks-evolution-inbox') as EvolutionInbox;
  if (inbox) el.inbox = inbox;
  document.body.appendChild(el);
  return el;
}

const SAMPLE_INBOX: ConductorInboxEntry[] = [
  {
    caseId: '550e8400-e29b-41d4-a716-446655440000',
    id: 'inbox-001', stage: 'pr-review', status: 'PENDING',
    category: 'lint-fix', areaId: 'lint-compliance',
    improvementCaseId: '660e8400-0001-0000-0000-000000000002',
    summary: 'Fix 12 lint violations in auth module',
    escalationTriggers: [{ layer: 'CONFIDENCE_SCORE', reason: 'High confidence (0.92)' }],
    confidence: 0.92, queuedAt: '2026-09-26T09:30:00Z',
    resolvedAt: null, timeoutMinutes: 1440, decision: null,
  },
  {
    caseId: '550e8400-e29b-41d4-a716-446655440000',
    id: 'inbox-002', stage: 'hypothesis-approval', status: 'APPROVED',
    category: 'dependency-update', areaId: 'dependency-freshness',
    improvementCaseId: '660e8400-0001-0000-0000-000000000001',
    summary: 'Upgrade quarkus-core from 3.35 to 3.36',
    escalationTriggers: [],
    confidence: 0.88, queuedAt: '2026-09-26T06:15:00Z',
    resolvedAt: '2026-09-26T06:20:00Z', timeoutMinutes: 720,
    decision: { outcome: 'APPROVED', reason: 'Changelog clean', feedback: null },
  },
];

describe('blocks-evolution-inbox', () => {
  afterEach(() => {
    document.body.querySelectorAll('blocks-evolution-inbox').forEach(el => el.remove());
  });

  it('has correct ARIA attributes', async () => {
    const el = createElement(SAMPLE_INBOX);
    await el.updateComplete;
    expect(el.getAttribute('role')).toBe('region');
    expect(el.getAttribute('aria-label')).toBe('Conductor inbox');
  });

  it('renders pages-table with inbox data', async () => {
    const el = createElement(SAMPLE_INBOX);
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-table');
    expect(table).not.toBeNull();
  });

  it('shows empty state when no entries', async () => {
    const el = createElement([]);
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('No inbox entries');
  });

  it('shows action buttons only for PENDING entries', async () => {
    const el = createElement(SAMPLE_INBOX);
    await el.updateComplete;
    const approveButtons = el.shadowRoot!.querySelectorAll('[aria-label*="Approve"]');
    expect(approveButtons.length).toBe(1);
  });

  it('hides actions in readonly mode', async () => {
    const el = createElement(SAMPLE_INBOX);
    el.readonly = true;
    await el.updateComplete;
    const approveButtons = el.shadowRoot!.querySelectorAll('[aria-label*="Approve"]');
    expect(approveButtons.length).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config test -- --reporter=verbose evolution-inbox`
Expected: FAIL (module not found)

- [ ] **Step 3: Implement evolution-inbox component**

Create `components/evolution-config/src/evolution-inbox.ts` using `ide_create_file`. Follow the same structure as `evolution-streams.ts`:

- Column defs: stage, category (badge), summary, confidence (bar), escalation triggers (badges), status (6-value coloured badge), queued time (relative), timeout remaining
- Status badge colours: PENDING=amber, APPROVED=green, REJECTED=red, REDIRECTED=blue, TIMED_OUT=grey, AUTO_APPROVED=light green
- Actions column: Approve + Reject buttons for PENDING rows only
- Reject opens a `pages-confirm-dialog` with reason (required) and feedback (optional) fields
- Approve calls `EvolutionApi.resolveGate(entryId, 'APPROVED')`
- After resolution, emits `EvolutionEventTopics.GATE_RESOLVED`

Component structure mirrors `evolution-streams.ts`: Props interface, dual data mode, ARIA attributes, same CSS patterns.

- [ ] **Step 4: Add export to index.ts**

```typescript
export { EvolutionInbox } from './evolution-inbox.js';
export type { EvolutionInboxProps } from './evolution-inbox.js';
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config test -- --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add components/evolution-config/
git commit -m "feat(#187): add blocks-evolution-inbox component

pages-table for ConductorInboxEntry with approve/reject actions,
6-value status badges, confidence bar, escalation trigger badges.
Dual data mode. Refs #187

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Refactor workbench to use renderContent callbacks

**Files:**
- Modify: `components/evolution-workbench/src/evolution-workbench.ts`
- Modify: `components/evolution-workbench/src/evolution-workbench.test.ts`
- Modify: `components/evolution-workbench/src/sample-data.ts` (if needed)

**Interfaces:**
- Consumes: `TabDefinition.renderContent` from Task 1, `EvolutionStreams` from Task 2, `EvolutionInbox` from Task 3
- Produces: Updated `EvolutionWorkbenchProps` (unchanged interface, new tab rendering), two new tabs (audit, health)

- [ ] **Step 1: Update workbench tests**

In `components/evolution-workbench/src/evolution-workbench.test.ts`, update the test that checks for detail-pane tabs:

```typescript
it('built-in tabs include all five', async () => {
  const el = createElement();
  el.state = SAMPLE_STATE;
  await el.updateComplete;
  const detailPane = el.shadowRoot!.querySelector('blocks-detail-pane') as any;
  const tabIds = detailPane.tabs.map((t: TabDefinition) => t.id);
  expect(tabIds).toContain('streams');
  expect(tabIds).toContain('inbox');
  expect(tabIds).toContain('audit');
  expect(tabIds).toContain('config');
  expect(tabIds).toContain('health');
});

it('built-in tabs have renderContent callbacks', async () => {
  const el = createElement();
  el.state = SAMPLE_STATE;
  await el.updateComplete;
  const detailPane = el.shadowRoot!.querySelector('blocks-detail-pane') as any;
  const builtInTabs = detailPane.tabs.filter(
    (t: TabDefinition) => ['streams', 'inbox', 'audit', 'config', 'health'].includes(t.id)
  );
  for (const tab of builtInTabs) {
    expect(tab.renderContent).toBeDefined();
    expect(typeof tab.renderContent).toBe('function');
  }
});

it('custom tabs do not have renderContent', async () => {
  const el = createElement();
  el.state = SAMPLE_STATE;
  el.tabs = [{ id: 'trading-risk', label: 'Trading Risk', tagName: 'div', order: 30 }];
  await el.updateComplete;
  const detailPane = el.shadowRoot!.querySelector('blocks-detail-pane') as any;
  const customTab = detailPane.tabs.find((t: TabDefinition) => t.id === 'trading-risk');
  expect(customTab.renderContent).toBeUndefined();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/blocks-ui-evolution-workbench test`
Expected: FAIL (no audit/health tabs, no renderContent on built-in tabs)

- [ ] **Step 3: Refactor _builtInTabs and add render methods**

Use `ide_replace_member` on `_builtInTabs`:

```typescript
private _builtInTabs(): TabDefinition[] {
  return [
    { id: 'streams', label: 'Streams', tagName: 'div', order: 10,
      badge: () => {
        const s = this._snapshot;
        return s && s.activeImprovementCount > 0 ? String(s.activeImprovementCount) : null;
      },
      renderContent: () => this._renderStreams() },
    { id: 'inbox', label: 'Inbox', tagName: 'div', order: 20,
      badge: () => {
        const s = this._snapshot;
        return s && s.pendingInboxCount > 0 ? String(s.pendingInboxCount) : null;
      },
      renderContent: () => this._renderInbox() },
    { id: 'audit', label: 'Audit', tagName: 'div', order: 30,
      renderContent: () => this._renderAudit() },
    { id: 'config', label: 'Configuration', tagName: 'div', order: 40,
      renderContent: () => this._renderConfig() },
    { id: 'health', label: 'Health', tagName: 'div', order: 50,
      renderContent: () => this._renderHealth() },
  ];
}
```

- [ ] **Step 4: Add render methods and remove dead code**

Remove the old `_renderTabContent` method. Add these methods using `ide_insert_member`:

```typescript
private _renderStreams() {
  return html`
    <div class="tab-content">
      <blocks-evolution-streams
        .endpoint=${this.endpoint}
        .caseId=${this.caseId}
        .tenancyId=${this.tenancyId}
        .streams=${this.streams}
      ></blocks-evolution-streams>
    </div>
  `;
}

private _renderInbox() {
  return html`
    <div class="tab-content">
      <blocks-evolution-inbox
        .endpoint=${this.endpoint}
        .caseId=${this.caseId}
        .tenancyId=${this.tenancyId}
        .inbox=${this.inbox}
      ></blocks-evolution-inbox>
    </div>
  `;
}

private _renderAudit() {
  const auditEndpoint = this.endpoint ? `${this.endpoint}/audit` : undefined;
  return html`
    <div class="tab-content">
      <blocks-audit-trail-viewer
        .endpoint=${auditEndpoint}
      ></blocks-audit-trail-viewer>
    </div>
  `;
}

private _renderConfig() {
  return html`
    <div class="tab-content">
      <div class="section-header">Deny Patterns</div>
      <blocks-deny-pattern-editor
        .endpoint=${this.endpoint}
        .caseId=${this.caseId}
        .tenancyId=${this.tenancyId}
        .patterns=${this.denyPatterns}
        .streams=${this.streams}
      ></blocks-deny-pattern-editor>

      <div class="section-header">Watch Patterns</div>
      <blocks-watch-pattern-editor
        .endpoint=${this.endpoint}
        .caseId=${this.caseId}
        .tenancyId=${this.tenancyId}
        .patterns=${this.watchPatterns}
        .categories=${this.categories}
      ></blocks-watch-pattern-editor>

      <div class="section-header">Gate Policy</div>
      <blocks-gate-policy-editor
        .endpoint=${this.endpoint}
        .caseId=${this.caseId}
        .tenancyId=${this.tenancyId}
        .stages=${this.stages}
        .policy=${this.gatePolicy}
        .streams=${this.streams}
      ></blocks-gate-policy-editor>
    </div>
  `;
}

private _renderHealth() {
  const s = this._snapshot;
  if (!s) return html`<div class="tab-content"><div class="empty-tab">No health data available.</div></div>`;
  return html`
    <div class="tab-content">
      <blocks-trust-score-panel
        .score=${s.healthScore}
        mode="full"
      ></blocks-trust-score-panel>
    </div>
  `;
}
```

- [ ] **Step 5: Add import for evolution-streams and evolution-inbox**

Add to the top of `evolution-workbench.ts`:

```typescript
import '@casehubio/blocks-ui-evolution-config/evolution-streams';
import '@casehubio/blocks-ui-evolution-config/evolution-inbox';
import '@casehubio/blocks-ui-audit-trail-viewer';
```

Also add `STREAM_CHANGED` to the `EvolutionEventTopics` import and add a listener in `connectedCallback`:

```typescript
onPagesEvent(this, EvolutionEventTopics.STREAM_CHANGED, () => this._refreshState()),
```

- [ ] **Step 6: Run tests**

Run: `yarn workspace @casehubio/blocks-ui-evolution-workbench test`
Expected: ALL PASS

- [ ] **Step 7: Run full typecheck and test suite**

Run: `yarn typecheck && yarn test`
Expected: no errors, all tests pass

- [ ] **Step 8: Commit**

```bash
git add components/evolution-workbench/ components/evolution-config/
git commit -m "feat(#188): refactor workbench to use renderContent callbacks

Replace placeholder tabs with real content via renderContent callbacks.
Add Audit and Health tabs. Wire streams/inbox components. Remove dead
_renderTabContent method. Refs #186 #187 #183 #184

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Schema registry + editor previews

### Task 5: Register new components in BlocksComponentRegistry

**Files:**
- Modify: `packages/blocks-ui-schema/src/registry.ts`

**Interfaces:**
- Consumes: `EvolutionStreamsProps` from Task 2, `EvolutionInboxProps` from Task 3

- [ ] **Step 1: Add imports and registry entries**

In `packages/blocks-ui-schema/src/registry.ts`, add the import:

```typescript
import type { EvolutionStreamsProps, EvolutionInboxProps } from '@casehubio/blocks-ui-evolution-config';
```

Add to `BlocksComponentRegistry`:

```typescript
'blocks-evolution-streams': EvolutionStreamsProps;
'blocks-evolution-inbox': EvolutionInboxProps;
```

- [ ] **Step 2: Generate schemas**

Run: `yarn workspace @casehubio/blocks-ui-schema run generate`
Expected: schemas regenerated without errors

- [ ] **Step 3: Run schema tests**

Run: `yarn workspace @casehubio/blocks-ui-schema test`
Expected: ALL PASS (completeness test passes with new entries)

- [ ] **Step 4: Commit**

```bash
git add packages/blocks-ui-schema/
git commit -m "feat(#188): register evolution-streams and evolution-inbox in schema registry

Adds BlocksComponentRegistry entries and regenerates Zod schemas.
Refs #188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 6: Deny-pattern preview against active streams (#181)

**Files:**
- Modify: `components/evolution-config/src/deny-pattern-editor.ts`
- Modify: `components/evolution-config/src/deny-pattern-editor.test.ts`

**Interfaces:**
- Consumes: `ImprovementStreamView` from `types.ts`
- Produces: `DenyPatternEditorProps.streams?: readonly ImprovementStreamView[]`

- [ ] **Step 1: Write failing test**

Add to `deny-pattern-editor.test.ts`:

```typescript
import type { ImprovementStreamView } from './types.js';

const SAMPLE_STREAMS: ImprovementStreamView[] = [
  { improvementCaseId: '001', category: 'lint-fix', target: 'AuthController', currentStage: 'implement', blockedBy: null, conflictBlocked: false, startedAt: '2026-09-26T06:00:00Z' },
  { improvementCaseId: '002', category: 'coverage-gap', target: 'PaymentService', currentStage: 'pr-review', blockedBy: null, conflictBlocked: false, startedAt: '2026-09-26T07:00:00Z' },
];

it('shows preview of matching streams when typing a deny pattern', async () => {
  const el = createElement(SAMPLE_VIEW);
  el.streams = SAMPLE_STREAMS;
  await el.updateComplete;

  // Open add form
  const addBtn = el.shadowRoot!.querySelector('.btn-add') as HTMLButtonElement;
  addBtn.click();
  await el.updateComplete;

  // Type a pattern that matches
  const input = el.shadowRoot!.querySelector('.add-form input') as HTMLInputElement;
  input.value = 'Auth';
  input.dispatchEvent(new Event('input'));
  await el.updateComplete;

  const preview = el.shadowRoot!.querySelector('.preview-section');
  expect(preview).not.toBeNull();
  expect(preview!.textContent).toContain('1 active stream');
  expect(preview!.textContent).toContain('AuthController');
});

it('hides preview when no streams prop provided', async () => {
  const el = createElement(SAMPLE_VIEW);
  await el.updateComplete;

  const addBtn = el.shadowRoot!.querySelector('.btn-add') as HTMLButtonElement;
  addBtn.click();
  await el.updateComplete;

  const input = el.shadowRoot!.querySelector('.add-form input') as HTMLInputElement;
  input.value = 'Auth';
  input.dispatchEvent(new Event('input'));
  await el.updateComplete;

  const preview = el.shadowRoot!.querySelector('.preview-section');
  expect(preview).toBeNull();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config test -- --reporter=verbose deny-pattern`
Expected: FAIL (no streams property, no preview-section)

- [ ] **Step 3: Add streams property and preview rendering**

Add `streams` property to `DenyPatternEditor`:

```typescript
@property({ type: Array, attribute: false }) streams?: readonly ImprovementStreamView[];
```

Add preview rendering in the add-form section, after the input:

```typescript
private _renderPreview() {
  if (!this.streams || !this._addValue.trim()) return nothing;
  const pattern = this._addValue.trim().toLowerCase();
  const matches = this.streams.filter(s =>
    s.target && s.target.toLowerCase().includes(pattern)
  );
  if (matches.length === 0) return nothing;
  return html`
    <div class="preview-section" style="margin-top:12px;padding:8px;background:var(--pages-warning-3,#fef3c7);border-radius:4px;font-size:13px">
      <strong>${matches.length} active stream${matches.length > 1 ? 's' : ''} would be denied:</strong>
      <ul style="margin:4px 0 0;padding-left:20px">
        ${matches.map(s => html`<li style="margin:2px 0"><code style="font-family:var(--pages-font-mono,monospace)">${s.target}</code> (${s.category}, ${s.currentStage})</li>`)}
      </ul>
    </div>
  `;
}
```

Add `this._renderPreview()` after the input in the add-form template.

Update `DenyPatternEditorProps` to include `streams`.

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config test -- --reporter=verbose deny-pattern`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add components/evolution-config/
git commit -m "feat(#181): deny-pattern testing preview against active streams

Shows matching active improvement streams when typing a new deny
pattern. Case-insensitive substring match on target. Refs #181

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 7: Gate-policy impact preview (#182)

**Files:**
- Modify: `components/evolution-config/src/gate-policy-editor.ts`
- Modify: `components/evolution-config/src/gate-policy-editor.test.ts`

**Interfaces:**
- Consumes: `ImprovementStreamView` from `types.ts`
- Produces: `GatePolicyEditorProps.streams?: readonly ImprovementStreamView[]`

- [ ] **Step 1: Write failing test**

Add to `gate-policy-editor.test.ts`:

```typescript
import type { ImprovementStreamView } from './types.js';

const SAMPLE_STREAMS: ImprovementStreamView[] = [
  { improvementCaseId: '001', category: 'lint-fix', target: 'auth-module', currentStage: 'pr-review', blockedBy: null, conflictBlocked: false, startedAt: '2026-09-26T06:00:00Z' },
  { improvementCaseId: '002', category: 'dep-update', target: 'quarkus', currentStage: 'pr-review', blockedBy: null, conflictBlocked: false, startedAt: '2026-09-26T07:00:00Z' },
  { improvementCaseId: '003', category: 'coverage', target: 'payment', currentStage: 'implement', blockedBy: null, conflictBlocked: false, startedAt: '2026-09-26T08:00:00Z' },
];

it('shows impact preview when streams provided', async () => {
  const el = createElement();
  el.streams = SAMPLE_STREAMS;
  el.stages = SAMPLE_STAGES;
  el.policy = SAMPLE_POLICY;
  await el.updateComplete;

  const preview = el.shadowRoot!.querySelector('.impact-preview');
  expect(preview).not.toBeNull();
  expect(preview!.textContent).toContain('pr-review');
  expect(preview!.textContent).toContain('2');
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config test -- --reporter=verbose gate-policy`
Expected: FAIL

- [ ] **Step 3: Add streams property and impact preview**

Add `streams` property to `GatePolicyEditor`. Add an impact preview section below the policy table that shows, for each gated stage, how many active streams are at that stage.

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/blocks-ui-evolution-config test -- --reporter=verbose gate-policy`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add components/evolution-config/
git commit -m "feat(#182): gate-policy impact preview

Shows how many active improvements at each stage would be affected
by the current gate mode. Refs #182

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Responsive layout + CLAUDE.md

### Task 8: Responsive layout with CSS container queries (#185)

**Files:**
- Modify: `components/evolution-workbench/src/evolution-workbench.ts` (styles only)
- Modify: `components/evolution-workbench/src/evolution-workbench.test.ts`

**Interfaces:**
- No new interfaces — CSS-only changes

- [ ] **Step 1: Write test for container-type**

Add to `evolution-workbench.test.ts`:

```typescript
it('has container-type for responsive queries', async () => {
  const el = createElement();
  el.state = SAMPLE_STATE;
  await el.updateComplete;
  const styles = getComputedStyle(el);
  expect(styles.containerType).toBe('inline-size');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/blocks-ui-evolution-workbench test`
Expected: FAIL (no container-type set)

- [ ] **Step 3: Add container queries to styles**

Use `ide_edit_member` on `styles` in `evolution-workbench.ts`. Add to the `:host` block:

```css
container-type: inline-size;
container-name: evolution-workbench;
```

Add container query rules after the existing styles:

```css
@container evolution-workbench (max-width: 600px) {
  .metric-grid { gap: 8px; padding: 8px; }
  .metric-card { min-width: 60px; }
  .metric-value { font-size: 18px; }
  .metric-label { font-size: 10px; }
}

@container evolution-workbench (max-width: 400px) {
  .metric-grid { flex-direction: column; align-items: stretch; }
  .metric-card { flex-direction: row; justify-content: space-between; min-width: unset; }
}
```

Add to `.tab-bar` equivalent (inside detail-pane, so this applies to the `.tabs` wrapper):

```css
.tabs { overflow: hidden; }
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/blocks-ui-evolution-workbench test`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add components/evolution-workbench/
git commit -m "feat(#185): evolution workbench responsive layout

CSS container queries on workbench host. Metric grid collapses at
narrow widths. Refs #185

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 9: Update CLAUDE.md and docs

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- None

- [ ] **Step 1: Update CLAUDE.md key directories**

Update the `evolution-config` entry in the Key Directories table to include the new components:

Add `EvolutionStreams (improvement stream table with block/unblock), EvolutionInbox (conductor inbox table with approve/reject)` to the evolution-config description.

- [ ] **Step 2: Run full test suite**

Run: `yarn test && yarn typecheck && yarn aria-check`
Expected: ALL PASS

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#188): update CLAUDE.md with new evolution components

Adds evolution-streams and evolution-inbox to key directories.
Refs #188

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-09-28-evolution-operational-design.md] — design spec this plan implements
- [components/evolution-workbench/src/evolution-workbench.ts] — existing workbench shell
- [components/evolution-config/src/types.ts] — domain types
- [components/evolution-config/src/api.ts] — EvolutionApi
- [components/evolution-config/src/deny-pattern-editor.ts] — editor pattern reference
- [components/detail-pane/src/detail-pane.ts:142-149] — render path to modify
- [components/detail-pane/src/types.ts] — TabDefinition to extend
- [packages/blocks-ui-schema/src/registry.ts] — component registry
- [PP-20260713-8ea1af] — component-customisation-pattern (render callbacks)
- [PP-20260907-fd8ee7] — component-registry-props (Props + BlocksComponentRegistry)
- [GitHub #188] — parent epic
- [GitHub #186] — streams tab
- [GitHub #187] — inbox tab
- [GitHub #181] — deny-pattern preview
- [GitHub #182] — gate-policy preview
- [GitHub #183] — audit trail tab
- [GitHub #184] — health detail tab
- [GitHub #185] — responsive layout
