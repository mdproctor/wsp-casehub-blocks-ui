# Agency Pattern Views — Batch 0: Foundation

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #277 — Epic: Agency Pattern Views — Full YAML-to-UI coverage aligned to agency taxonomy
**Issue group:** #277
**Deferred:** #258 (gallery scaffold) — separate plan per issue, builds on Batch 0 infrastructure

**Goal:** Build the shared infrastructure that all pattern views depend on: PatternViewMixin, shared styles, directory structure, exhaustive dispatch in agent-viewer, base-pattern-view for thin types, playbook-spec-view, and migration of existing voting/debate views.

**Architecture:** Pattern views are LitElement Web Components in `packages/agent-viewer/src/views/<category>/`. Agent-viewer dispatches via an exhaustive switch on `PatternCategory` (closed discriminated union with `default: never` guards). A thin `PatternViewMixin` provides the uniform `spec` property, ARIA setup, and empty state. Six capability types with no category-specific fields share a single `agency-base-pattern-view`. Existing voting-view and debate-view are migrated to the new convention (renamed, moved to subdirectories, adopt mixin, `pattern` → `spec`).

**Tech Stack:** TypeScript, Lit 3, Vitest + jsdom, Yarn workspaces

## Global Constraints

- Custom element tags: `agency-<category>-<type>-view` (D6)
- Property name: `spec` (not `pattern`) for PatternCategory data (R1-07)
- ARIA: `role="region"`, `aria-label`, `aria-busy` on every component (CLAUDE.md)
- Events: `<component-prefix>:<action>` namespace (PP-20260930)
- Inline styles in render callbacks (PP-20260713)
- `default: never` in every inner switch for exhaustiveness (R2-04)
- Pattern views in `packages/agent-viewer/src/views/` are internal — exempt from PP-20260907 registry entries

---

## Batch 1: Shared styles, mixin, and directory scaffold

### Task 1: PatternViewMixin and shared styles

**Files:**
- Create: `packages/agent-viewer/src/shared-styles.ts`
- Create: `packages/agent-viewer/src/shared-styles.test.ts`

**Interfaces:**
- Consumes: `PatternCategory` from `@casehubio/mosaic-core`
- Produces:
  - `patternViewStyles: CSSResult` — shared `:host` CSS block
  - `PatternViewMixin<T extends Constructor<LitElement>>(Base: T)` — mixin adding `spec: PatternCategory`, ARIA setup, `setLoading(loading: boolean)`, `renderEmptyState(message?: string): TemplateResult`

- [ ] **Step 1: Write the failing test for PatternViewMixin**

```typescript
// packages/agent-viewer/src/shared-styles.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { LitElement, html } from 'lit';
import { customElement } from 'lit/decorators.js';
import type { DeliberationPatternCategory } from '@casehubio/mosaic-core';
import { PatternViewMixin, patternViewStyles } from './shared-styles.js';

@customElement('test-pattern-view')
class TestPatternView extends PatternViewMixin(LitElement) {
  static override styles = [patternViewStyles];
  override render() {
    if (!this.spec) return this.renderEmptyState();
    return html`<div class="content">${this.spec.category}/${this.spec.type}</div>`;
  }
}

function fixture(): DeliberationPatternCategory {
  return { category: 'deliberation', type: 'voting', name: 'test-vote' };
}

describe('PatternViewMixin', () => {
  let el: TestPatternView;

  afterEach(() => el?.remove());

  it('sets role="region" on connect', async () => {
    el = document.createElement('test-pattern-view') as TestPatternView;
    el.spec = fixture();
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.getAttribute('role')).toBe('region');
  });

  it('sets aria-label from category', async () => {
    el = document.createElement('test-pattern-view') as TestPatternView;
    el.spec = fixture();
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.getAttribute('aria-label')).toBe('deliberation view');
  });

  it('sets aria-label to "pattern view" when spec is undefined', async () => {
    el = document.createElement('test-pattern-view') as TestPatternView;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.getAttribute('aria-label')).toBe('pattern view');
  });

  it('setLoading updates aria-busy', async () => {
    el = document.createElement('test-pattern-view') as TestPatternView;
    el.spec = fixture();
    document.body.appendChild(el);
    await el.updateComplete;
    (el as any).setLoading(true);
    expect(el.getAttribute('aria-busy')).toBe('true');
    (el as any).setLoading(false);
    expect(el.getAttribute('aria-busy')).toBe('false');
  });

  it('renderEmptyState renders default message', async () => {
    el = document.createElement('test-pattern-view') as TestPatternView;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.empty-state')?.textContent).toBe('No pattern data');
  });

  it('renders content when spec is set', async () => {
    el = document.createElement('test-pattern-view') as TestPatternView;
    el.spec = fixture();
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.content')?.textContent).toBe('deliberation/voting');
  });
});

describe('patternViewStyles', () => {
  it('is a CSSResult', () => {
    expect(patternViewStyles).toBeDefined();
    expect(patternViewStyles.cssText).toContain('display: block');
    expect(patternViewStyles.cssText).toContain('border-radius');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/agent-viewer run test -- --run src/shared-styles.test.ts`
Expected: FAIL — module `./shared-styles.js` not found

- [ ] **Step 3: Implement PatternViewMixin and patternViewStyles**

```typescript
// packages/agent-viewer/src/shared-styles.ts
import { css, html, type CSSResult, type TemplateResult } from 'lit';
import { property } from 'lit/decorators.js';
import type { LitElement } from 'lit';
import type { PatternCategory } from '@casehubio/mosaic-core';

type Constructor<T = {}> = new (...args: any[]) => T;

export const patternViewStyles: CSSResult = css`
  :host {
    display: block;
    background: rgba(255, 255, 255, 0.02);
    border: 1px solid rgba(255, 255, 255, 0.08);
    border-radius: 8px;
    overflow: hidden;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
    color: #cdd6f4;
  }

  .empty-state {
    padding: 32px;
    text-align: center;
    color: #7f849c;
    font-size: 13px;
  }
`;

export const PatternViewMixin = <T extends Constructor<LitElement>>(Base: T) => {
  class PatternViewHost extends Base {
    @property({ attribute: false }) spec!: PatternCategory;

    override connectedCallback() {
      super.connectedCallback();
      this.setAttribute('role', 'region');
      this.setAttribute('aria-label', `${this.spec?.category ?? 'pattern'} view`);
    }

    protected setLoading(loading: boolean) {
      this.setAttribute('aria-busy', String(loading));
    }

    protected renderEmptyState(message = 'No pattern data'): TemplateResult {
      return html`<div class="empty-state">${message}</div>`;
    }
  }
  return PatternViewHost as Constructor<PatternViewHost> & T;
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/agent-viewer run test -- --run src/shared-styles.test.ts`
Expected: PASS — all 6 tests green

- [ ] **Step 5: Create directory structure**

```bash
mkdir -p /Users/mdproctor/claude/casehub/casehub-mosaic/packages/agent-viewer/src/views/orchestration
mkdir -p /Users/mdproctor/claude/casehub/casehub-mosaic/packages/agent-viewer/src/views/deliberation
mkdir -p /Users/mdproctor/claude/casehub/casehub-mosaic/packages/agent-viewer/src/views/perception
mkdir -p /Users/mdproctor/claude/casehub/casehub-mosaic/packages/agent-viewer/src/views/governance
mkdir -p /Users/mdproctor/claude/casehub/casehub-mosaic/packages/agent-viewer/src/views/optimisation
mkdir -p /Users/mdproctor/claude/casehub/casehub-mosaic/packages/agent-viewer/src/views/social
```

- [ ] **Step 6: Commit**

```bash
git add packages/agent-viewer/src/shared-styles.ts packages/agent-viewer/src/shared-styles.test.ts packages/agent-viewer/src/views/
git commit -m "feat(agent-viewer): add PatternViewMixin, shared styles, and views directory structure

Refs #277"
```

---

## Batch 2: Migrate voting-view and debate-view

### Task 2: Migrate voting-view to new convention

**Files:**
- Move: `packages/agent-viewer/src/voting-view.ts` → `packages/agent-viewer/src/views/deliberation/voting-view.ts` (use `ide_move_file`)
- Move: `packages/agent-viewer/src/voting-view.test.ts` → `packages/agent-viewer/src/views/deliberation/voting-view.test.ts` (use `ide_move_file`)
- Modify: the moved `voting-view.ts` — adopt PatternViewMixin, rename `pattern` → `spec`, rename element to `agency-deliberation-voting-view`
- Modify: the moved `voting-view.test.ts` — update element name and property name
- Modify: `packages/agent-viewer/src/index.ts` — update import paths

**Interfaces:**
- Consumes: `PatternViewMixin`, `patternViewStyles` from `../shared-styles.js`
- Produces: `VotingView` class with `spec: PatternCategory` property, registered as `agency-deliberation-voting-view`

- [ ] **Step 1: Move voting-view files to deliberation subdirectory**

Use `ide_move_file` for both:
- `packages/agent-viewer/src/voting-view.ts` → `packages/agent-viewer/src/views/deliberation/voting-view.ts`
- `packages/agent-viewer/src/voting-view.test.ts` → `packages/agent-viewer/src/views/deliberation/voting-view.test.ts`

- [ ] **Step 2: Update voting-view to adopt PatternViewMixin**

In the moved `packages/agent-viewer/src/views/deliberation/voting-view.ts`:

1. Change import: add `import { PatternViewMixin, patternViewStyles } from '../../shared-styles.js';`
2. Change class declaration: `export class VotingView extends PatternViewMixin(LitElement) {`
3. Remove the `@property({ attribute: false }) pattern!: DeliberationPatternCategory;` line (mixin provides `spec`)
4. Change `@customElement('voting-view')` → `@customElement('agency-deliberation-voting-view')`
5. In `static override styles`, prepend `patternViewStyles` to the array: `static override styles = [patternViewStyles, css\`...\`];` — remove the `:host` block from the component's own CSS since `patternViewStyles` provides it
6. Replace all `this.pattern` references with `this.spec` throughout the file
7. Cast `this.spec` to `DeliberationPatternCategory` where type-specific fields are accessed: `const p = this.spec as DeliberationPatternCategory;`

- [ ] **Step 3: Update voting-view test**

In `packages/agent-viewer/src/views/deliberation/voting-view.test.ts`:

1. Update import paths: `import '../../shared-styles.js';` (ensure mixin is loaded)
2. Change element creation: `document.createElement('agency-deliberation-voting-view')`
3. Change property assignment: `el.spec = fixture();` (was `el.pattern`)
4. Update import path for the view: `import './voting-view.js';` and `import type { VotingView } from './voting-view.js';`
5. Add ARIA test:

```typescript
it('has role="region" and aria-label', () => {
  expect(el.getAttribute('role')).toBe('region');
  expect(el.getAttribute('aria-label')).toBe('deliberation view');
});
```

- [ ] **Step 4: Run voting-view tests to verify migration**

Run: `yarn workspace @casehubio/agent-viewer run test -- --run src/views/deliberation/voting-view.test.ts`
Expected: PASS — all existing tests pass with new element name and property name, plus new ARIA test

- [ ] **Step 5: Commit**

```bash
git add packages/agent-viewer/src/
git commit -m "refactor(agent-viewer): migrate voting-view to agency-deliberation-voting-view

- Move to src/views/deliberation/
- Adopt PatternViewMixin (pattern → spec)
- Add ARIA assertions

Refs #277"
```

### Task 3: Migrate debate-view to new convention

**Files:**
- Move: `packages/agent-viewer/src/debate-view.ts` → `packages/agent-viewer/src/views/deliberation/debate-view.ts` (use `ide_move_file`)
- Move: `packages/agent-viewer/src/debate-view.test.ts` → `packages/agent-viewer/src/views/deliberation/debate-view.test.ts` (use `ide_move_file`)
- Modify: the moved `debate-view.ts` — same changes as voting-view migration
- Modify: the moved `debate-view.test.ts` — same changes as voting-view test migration
- Modify: `packages/agent-viewer/src/index.ts` — update import paths

**Interfaces:**
- Consumes: `PatternViewMixin`, `patternViewStyles` from `../../shared-styles.js`
- Produces: `DebateView` class with `spec: PatternCategory` property, registered as `agency-deliberation-debate-view`

- [ ] **Step 1: Move debate-view files to deliberation subdirectory**

Use `ide_move_file` for both:
- `packages/agent-viewer/src/debate-view.ts` → `packages/agent-viewer/src/views/deliberation/debate-view.ts`
- `packages/agent-viewer/src/debate-view.test.ts` → `packages/agent-viewer/src/views/deliberation/debate-view.test.ts`

- [ ] **Step 2: Update debate-view to adopt PatternViewMixin**

Same changes as Task 2 Step 2 but for debate-view:

1. Add import: `import { PatternViewMixin, patternViewStyles } from '../../shared-styles.js';`
2. Change class: `export class DebateView extends PatternViewMixin(LitElement) {`
3. Remove the `@property({ attribute: false }) pattern!: DeliberationPatternCategory;` line
4. Change element name: `@customElement('agency-deliberation-debate-view')`
5. Prepend `patternViewStyles` to styles array, remove duplicate `:host` block
6. Replace `this.pattern` → `this.spec` throughout
7. Cast where needed: `const p = this.spec as DeliberationPatternCategory;`

- [ ] **Step 3: Update debate-view test**

Same changes as Task 2 Step 3 but for debate-view:

1. Change element creation: `document.createElement('agency-deliberation-debate-view')`
2. Change property: `el.spec = fixture();`
3. Update import paths
4. Add ARIA test

- [ ] **Step 4: Run debate-view tests**

Run: `yarn workspace @casehubio/agent-viewer run test -- --run src/views/deliberation/debate-view.test.ts`
Expected: PASS

- [ ] **Step 5: Update index.ts exports**

In `packages/agent-viewer/src/index.ts`:
- Change `export { VotingView } from './voting-view.js';` → `export { VotingView } from './views/deliberation/voting-view.js';`
- Change `export { DebateView } from './debate-view.js';` → `export { DebateView } from './views/deliberation/debate-view.js';`

- [ ] **Step 6: Run all agent-viewer tests to verify nothing broke**

Run: `yarn workspace @casehubio/agent-viewer run test -- --run`
Expected: PASS — all tests across the package pass

- [ ] **Step 7: Commit**

```bash
git add packages/agent-viewer/src/
git commit -m "refactor(agent-viewer): migrate debate-view to agency-deliberation-debate-view

- Move to src/views/deliberation/
- Adopt PatternViewMixin (pattern → spec)
- Update index.ts exports for both migrated views
- Add ARIA assertions

Refs #277"
```

---

## Batch 3: Base pattern view, playbook spec view, and exhaustive dispatch

### Task 4: Create agency-base-pattern-view

**Files:**
- Create: `packages/agent-viewer/src/views/base-pattern-view.ts`
- Create: `packages/agent-viewer/src/views/base-pattern-view.test.ts`

**Interfaces:**
- Consumes: `PatternViewMixin`, `patternViewStyles` from `../shared-styles.js`; `AgentCard` from `../agent-card.js`
- Produces: `BasePatternView` class registered as `agency-base-pattern-view`, renders agent card grid for types with no category-specific fields

- [ ] **Step 1: Write the failing test**

```typescript
// packages/agent-viewer/src/views/base-pattern-view.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import type { GovernancePatternCategory } from '@casehubio/mosaic-core';
import './base-pattern-view.js';

function fixture(): GovernancePatternCategory {
  return {
    category: 'governance',
    type: 'oversight',
    name: 'deployment-gate',
    description: 'Reviews deployment readiness',
    agents: [
      { type: 'worker', name: 'compliance-checker', description: 'Verifies compliance rules', capabilities: ['compliance'] },
      { type: 'human', name: 'release-manager', description: 'Final approval' },
    ],
  };
}

describe('agency-base-pattern-view', () => {
  let el: HTMLElement;

  afterEach(() => el?.remove());

  async function create(spec: any) {
    el = document.createElement('agency-base-pattern-view');
    (el as any).spec = spec;
    document.body.appendChild(el);
    await (el as any).updateComplete;
  }

  it('has role="region" and aria-label', async () => {
    await create(fixture());
    expect(el.getAttribute('role')).toBe('region');
    expect(el.getAttribute('aria-label')).toBe('governance view');
  });

  it('renders type badge from spec.type', async () => {
    await create(fixture());
    expect(el.shadowRoot!.querySelector('.type-badge')?.textContent).toBe('oversight');
  });

  it('renders category badge', async () => {
    expect(el.shadowRoot!.querySelector('.category-badge')?.textContent).toBe('governance');
  });

  it('renders name', async () => {
    await create(fixture());
    expect(el.shadowRoot!.querySelector('.header-name')?.textContent).toBe('deployment-gate');
  });

  it('renders description', async () => {
    await create(fixture());
    expect(el.shadowRoot!.textContent).toContain('Reviews deployment readiness');
  });

  it('renders agent cards', async () => {
    await create(fixture());
    expect(el.shadowRoot!.querySelectorAll('agent-card').length).toBe(2);
  });

  it('renders empty state when no spec', async () => {
    await create(undefined);
    expect(el.shadowRoot!.querySelector('.empty-state')).toBeTruthy();
  });

  it('works for communication types', async () => {
    await create({ category: 'communication', type: 'channel-execution', name: 'support-channel' });
    expect(el.shadowRoot!.querySelector('.type-badge')?.textContent).toBe('channel-execution');
    expect(el.shadowRoot!.querySelector('.category-badge')?.textContent).toBe('communication');
  });

  it('works for social/reflection type', async () => {
    await create({ category: 'social', type: 'reflection', name: 'daily-reflection' });
    expect(el.shadowRoot!.querySelector('.type-badge')?.textContent).toBe('reflection');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/agent-viewer run test -- --run src/views/base-pattern-view.test.ts`
Expected: FAIL — module `./base-pattern-view.js` not found

- [ ] **Step 3: Implement base-pattern-view**

```typescript
// packages/agent-viewer/src/views/base-pattern-view.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement } from 'lit/decorators.js';
import { PatternViewMixin, patternViewStyles } from '../shared-styles.js';
import '../agent-card.js';

@customElement('agency-base-pattern-view')
export class BasePatternView extends PatternViewMixin(LitElement) {
  static override styles = [patternViewStyles, css`
    .header {
      padding: 10px 16px;
      border-bottom: 1px solid rgba(255, 255, 255, 0.08);
      display: flex;
      align-items: center;
      gap: 8px;
    }
    .category-badge {
      font-size: 10px;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 0.5px;
      padding: 2px 8px;
      border-radius: 4px;
      background: rgba(137, 180, 250, 0.15);
      color: #89b4fa;
    }
    .type-badge {
      font-size: 10px;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 0.5px;
      padding: 2px 8px;
      border-radius: 4px;
      background: rgba(166, 227, 161, 0.15);
      color: #a6e3a1;
    }
    .header-name {
      font-size: 14px;
      font-weight: 600;
      color: #f0f4ff;
      margin-left: auto;
    }
    .description {
      padding: 12px 16px;
      font-size: 13px;
      color: #a6adc8;
      border-bottom: 1px solid rgba(255, 255, 255, 0.06);
    }
    .agents-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
      gap: 8px;
      padding: 12px 16px;
    }
  `];

  override render() {
    if (!this.spec) return this.renderEmptyState();
    return html`
      <div class="header">
        <span class="category-badge">${this.spec.category}</span>
        <span class="type-badge">${this.spec.type}</span>
        ${this.spec.name ? html`<span class="header-name">${this.spec.name}</span>` : nothing}
      </div>
      ${this.spec.description ? html`<div class="description">${this.spec.description}</div>` : nothing}
      ${this.spec.agents?.length ? html`
        <div class="agents-grid">
          ${this.spec.agents.map(a => html`<agent-card .agent=${a}></agent-card>`)}
        </div>
      ` : nothing}
    `;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/agent-viewer run test -- --run src/views/base-pattern-view.test.ts`
Expected: PASS — all 9 tests green

- [ ] **Step 5: Commit**

```bash
git add packages/agent-viewer/src/views/base-pattern-view.ts packages/agent-viewer/src/views/base-pattern-view.test.ts
git commit -m "feat(agent-viewer): add agency-base-pattern-view for thin capability types

Shared view for 6 types with no category-specific fields:
governance/oversight, governance/attestation, governance/trust,
communication/channel-execution, communication/channel-summary,
social/reflection.

Refs #277"
```

### Task 5: Create agency-playbook-spec-view

**Files:**
- Create: `packages/agent-viewer/src/views/playbook-spec-view.ts`
- Create: `packages/agent-viewer/src/views/playbook-spec-view.test.ts`

**Interfaces:**
- Consumes: `PatternViewMixin`, `patternViewStyles` from `../shared-styles.js`; `PlaybookPatternCategory` from `@casehubio/mosaic-core`
- Produces: `PlaybookSpecView` class registered as `agency-playbook-spec-view`, renders thin spec descriptor (type label, name, description)

- [ ] **Step 1: Write the failing test**

```typescript
// packages/agent-viewer/src/views/playbook-spec-view.test.ts
import { describe, it, expect, afterEach } from 'vitest';
import './playbook-spec-view.js';

describe('agency-playbook-spec-view', () => {
  let el: HTMLElement;

  afterEach(() => el?.remove());

  async function create(spec: any) {
    el = document.createElement('agency-playbook-spec-view');
    (el as any).spec = spec;
    document.body.appendChild(el);
    await (el as any).updateComplete;
  }

  it('has role="region" and aria-label', async () => {
    await create({ category: 'playbook', type: 'sequence', name: 'deploy-pipeline' });
    expect(el.getAttribute('role')).toBe('region');
    expect(el.getAttribute('aria-label')).toBe('playbook view');
  });

  it('renders playbook type badge', async () => {
    await create({ category: 'playbook', type: 'parallel', name: 'fan-out' });
    expect(el.shadowRoot!.querySelector('.type-badge')?.textContent).toBe('parallel');
  });

  it('renders name', async () => {
    await create({ category: 'playbook', type: 'loop', name: 'retry-loop', description: 'Retries until success' });
    expect(el.shadowRoot!.querySelector('.header-name')?.textContent).toBe('retry-loop');
  });

  it('renders description', async () => {
    await create({ category: 'playbook', type: 'loop', name: 'retry-loop', description: 'Retries until success' });
    expect(el.shadowRoot!.textContent).toContain('Retries until success');
  });

  it('renders empty state when no spec', async () => {
    await create(undefined);
    expect(el.shadowRoot!.querySelector('.empty-state')).toBeTruthy();
  });

  it('renders all playbook types', async () => {
    for (const type of ['sequence', 'parallel', 'conditional', 'match', 'loop', 'try-catch']) {
      await create({ category: 'playbook', type, name: `test-${type}` });
      expect(el.shadowRoot!.querySelector('.type-badge')?.textContent).toBe(type);
      el.remove();
    }
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/agent-viewer run test -- --run src/views/playbook-spec-view.test.ts`
Expected: FAIL — module `./playbook-spec-view.js` not found

- [ ] **Step 3: Implement playbook-spec-view**

```typescript
// packages/agent-viewer/src/views/playbook-spec-view.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement } from 'lit/decorators.js';
import { PatternViewMixin, patternViewStyles } from '../shared-styles.js';

@customElement('agency-playbook-spec-view')
export class PlaybookSpecView extends PatternViewMixin(LitElement) {
  static override styles = [patternViewStyles, css`
    .header {
      padding: 10px 16px;
      border-bottom: 1px solid rgba(255, 255, 255, 0.08);
      display: flex;
      align-items: center;
      gap: 8px;
    }
    .category-badge {
      font-size: 10px;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 0.5px;
      padding: 2px 8px;
      border-radius: 4px;
      background: rgba(203, 166, 247, 0.15);
      color: #cba6f7;
    }
    .type-badge {
      font-size: 10px;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 0.5px;
      padding: 2px 8px;
      border-radius: 4px;
      background: rgba(166, 227, 161, 0.15);
      color: #a6e3a1;
    }
    .header-name {
      font-size: 14px;
      font-weight: 600;
      color: #f0f4ff;
      margin-left: auto;
    }
    .description {
      padding: 12px 16px;
      font-size: 13px;
      color: #a6adc8;
    }
    .note {
      padding: 12px 16px;
      font-size: 12px;
      color: #7f849c;
      font-style: italic;
    }
  `];

  override render() {
    if (!this.spec) return this.renderEmptyState();
    return html`
      <div class="header">
        <span class="category-badge">playbook</span>
        <span class="type-badge">${this.spec.type}</span>
        ${this.spec.name ? html`<span class="header-name">${this.spec.name}</span>` : nothing}
      </div>
      ${this.spec.description ? html`<div class="description">${this.spec.description}</div>` : nothing}
      <div class="note">Playbook step orchestration — see Steps tab for runtime pipeline view</div>
    `;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/agent-viewer run test -- --run src/views/playbook-spec-view.test.ts`
Expected: PASS — all 6 tests green

- [ ] **Step 5: Commit**

```bash
git add packages/agent-viewer/src/views/playbook-spec-view.ts packages/agent-viewer/src/views/playbook-spec-view.test.ts
git commit -m "feat(agent-viewer): add agency-playbook-spec-view for PlaybookPatternCategory

Thin descriptor renderer for playbook types (sequence, parallel,
conditional, match, loop, try-catch). Rich pipeline visualization
remains in coordination-pipeline-view via the Steps tab.

Refs #277"
```

### Task 6: Update agent-viewer dispatch to exhaustive switch

**Files:**
- Modify: `packages/agent-viewer/src/agent-viewer.ts` — replace `renderPattern`, `renderOrchestrationPattern`, `renderDeliberationPattern` with exhaustive switch; update `AgentViewerData.pattern` → `AgentViewerData.spec`; import new view components
- Modify: `packages/agent-viewer/src/agent-viewer.test.ts` — update property name `pattern` → `spec` in data fixture; update element name references

**Interfaces:**
- Consumes: `PatternCategory` from `@casehubio/mosaic-core`; all view components from `./views/`
- Produces: `AgentViewerData` with `spec?: PatternCategory` (was `pattern`); exhaustive dispatch

- [ ] **Step 1: Update agent-viewer.ts — imports and type**

In `packages/agent-viewer/src/agent-viewer.ts`:

1. Update import: remove `OrchestrationPatternCategory` from the mosaic-core import (no longer needed for separate method)
2. Add imports for new views:
```typescript
import './views/deliberation/voting-view.js';
import './views/deliberation/debate-view.js';
import './views/base-pattern-view.js';
import './views/playbook-spec-view.js';
```
3. Remove old view imports: `import './voting-view.js';` and `import './debate-view.js';`
4. In `AgentViewerData` interface: rename `pattern?: PatternCategory` → `spec?: PatternCategory`
5. Update all references to `this.data.pattern` → `this.data.spec` (in `renderTabBar`, `renderActiveTab`, etc.)

- [ ] **Step 2: Replace renderPattern with exhaustive switch**

Remove `renderOrchestrationPattern` and `renderDeliberationPattern` methods. Replace `renderPattern` with:

```typescript
private renderPattern(): TemplateResult {
  const spec = this.data.spec;
  if (!spec) return html`<div class="empty-state">No pattern data available</div>`;
  switch (spec.category) {
    case 'orchestration':
      switch (spec.type) {
        case 'supervisor':
        case 'choreography':
        case 'saga':
        case 'decomposition':
          return html`<div class="empty-state">${spec.category}/${spec.type} — coming in Wave 1</div>`;
        default: { const _o: never = spec.type; return _o; }
      }
    case 'deliberation':
      switch (spec.type) {
        case 'voting':
          return html`<agency-deliberation-voting-view .spec=${spec}></agency-deliberation-voting-view>`;
        case 'debate':
          return html`<agency-deliberation-debate-view .spec=${spec}></agency-deliberation-debate-view>`;
        case 'negotiation':
        case 'coalition':
        case 'normative':
          return html`<div class="empty-state">${spec.category}/${spec.type} — coming in Wave 2</div>`;
        default: { const _d: never = spec.type; return _d; }
      }
    case 'perception':
      switch (spec.type) {
        case 'summarisation':
        case 'narrative':
        case 'observation':
          return html`<div class="empty-state">${spec.category}/${spec.type} — coming in Wave 3</div>`;
        default: { const _p: never = spec.type; return _p; }
      }
    case 'communication':
      switch (spec.type) {
        case 'channel-execution':
        case 'channel-summary':
          return html`<agency-base-pattern-view .spec=${spec}></agency-base-pattern-view>`;
        default: { const _c: never = spec.type; return _c; }
      }
    case 'governance':
      switch (spec.type) {
        case 'judgment':
          return html`<div class="empty-state">${spec.category}/${spec.type} — coming in Wave 1</div>`;
        case 'oversight':
        case 'attestation':
        case 'trust':
          return html`<agency-base-pattern-view .spec=${spec}></agency-base-pattern-view>`;
        default: { const _g: never = spec.type; return _g; }
      }
    case 'optimisation':
      switch (spec.type) {
        case 'prompt-optimisation':
          return html`<div class="empty-state">${spec.category}/${spec.type} — coming in Wave 3</div>`;
        default: { const _x: never = spec.type; return _x; }
      }
    case 'social':
      switch (spec.type) {
        case 'memory-hygiene':
        case 'cognition':
          return html`<div class="empty-state">${spec.category}/${spec.type} — coming in Wave 3</div>`;
        case 'reflection':
          return html`<agency-base-pattern-view .spec=${spec}></agency-base-pattern-view>`;
        default: { const _s: never = spec.type; return _s; }
      }
    case 'playbook':
      return html`<agency-playbook-spec-view .spec=${spec}></agency-playbook-spec-view>`;
    default: { const _: never = spec; return _; }
  }
}
```

- [ ] **Step 3: Update agent-viewer test**

In `packages/agent-viewer/src/agent-viewer.test.ts`:

1. In the `fixture()` function: rename `pattern:` → `spec:`
2. Update any tests that reference `data.pattern` → `data.spec`
3. Add exhaustiveness test:

```typescript
it('dispatches deliberation/voting to agency-deliberation-voting-view', async () => {
  el.data = { ...fixture(), spec: { category: 'deliberation', type: 'voting', name: 'test' } };
  el.shadowRoot!.querySelector<HTMLElement>('[data-tab="pattern"]')?.click();
  await el.updateComplete;
  expect(el.shadowRoot!.querySelector('agency-deliberation-voting-view')).toBeTruthy();
});

it('dispatches governance/oversight to agency-base-pattern-view', async () => {
  el.data = { ...fixture(), spec: { category: 'governance', type: 'oversight', name: 'test' } };
  el.shadowRoot!.querySelector<HTMLElement>('[data-tab="pattern"]')?.click();
  await el.updateComplete;
  expect(el.shadowRoot!.querySelector('agency-base-pattern-view')).toBeTruthy();
});

it('dispatches playbook to agency-playbook-spec-view', async () => {
  el.data = { ...fixture(), spec: { category: 'playbook', type: 'sequence', name: 'test' } };
  el.shadowRoot!.querySelector<HTMLElement>('[data-tab="pattern"]')?.click();
  await el.updateComplete;
  expect(el.shadowRoot!.querySelector('agency-playbook-spec-view')).toBeTruthy();
});
```

- [ ] **Step 4: Update index.ts exports**

In `packages/agent-viewer/src/index.ts`:

1. Add export for `AgentViewerData` — ensure the `spec` property name is in the exported type
2. Add exports for new views:
```typescript
export { BasePatternView } from './views/base-pattern-view.js';
export { PlaybookSpecView } from './views/playbook-spec-view.js';
```

- [ ] **Step 5: Run all agent-viewer tests**

Run: `yarn workspace @casehubio/agent-viewer run test -- --run`
Expected: PASS — all tests across the package pass

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: PASS — the `default: never` guards compile cleanly with `strict: true`

- [ ] **Step 7: Run full build**

Run: `yarn build`
Expected: PASS — topological build succeeds

- [ ] **Step 8: Commit**

```bash
git add packages/agent-viewer/src/
git commit -m "feat(agent-viewer): exhaustive PatternCategory dispatch with default:never guards

- Switch dispatches all 22 agency types + playbook
- Existing views: voting → agency-deliberation-voting-view, debate → agency-deliberation-debate-view
- 6 base-field types → agency-base-pattern-view
- Playbook specs → agency-playbook-spec-view
- Remaining types show wave-number placeholder
- AgentViewerData.pattern renamed to .spec

Refs #277"
```

---

## References

- [2026-10-26-agency-pattern-views-design.md] — design spec this plan implements
- `packages/agent-viewer/src/agent-viewer.ts:262-278` — current dispatch logic (pre-migration)
- `packages/agent-viewer/src/voting-view.ts` — existing voting view (migration source)
- `packages/agent-viewer/src/debate-view.ts` — existing debate view (migration source)
- `packages/mosaic-core/src/types/patterns/categories.ts` — PatternCategoryName union
- `packages/mosaic-core/src/types/patterns/base.ts` — BasePatternCategory, 5 category interfaces
- `packages/mosaic-core/src/types/patterns/orchestration.ts` — OrchestrationPatternCategory
- `packages/mosaic-core/src/types/patterns/deliberation.ts` — DeliberationPatternCategory
- PP-20260713 — component customisation pattern
- PP-20260930 — custom event namespace prefix
- PP-20260907 — component registry props (pattern views exempt)
- GitHub #277 — epic issue
- GitHub #258 — gallery demo data (scaffold in Batch 0)
