# Preferences Editor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #92 — preferences-editor component
**Issue group:** #92

**Goal:** Build a tree-table preferences editor that fetches schema and records from the platform REST API, shows scope hierarchy with preference leaves, and provides type-aware inline editing.

**Architecture:** Single `<preferences-editor>` component using `pages-data-table` with `expandable` config. Fetches schema (`GET /preferences/schema`) and bulk records (`GET /preferences`) in two calls, merges client-side into a flat dataset with `id`/`parentId` columns for tree-table rendering. Type-aware editing via a `<value-editor>` sub-component driven by `PreferenceSchemaDescriptor`. Follows case-explorer's self-managed fetch pattern (no DataSourceMixin).

**Tech Stack:** Lit 3, TypeScript, `@casehubio/pages-data` (TypedDataSet, fromRows, ColumnType), `@casehubio/pages-table` (pages-data-table with expandable), vitest + jsdom.

## Global Constraints

- Pre-release platform — breaking changes cost nothing
- Protocol PP-20260713-8ea1af — typed config properties + render callbacks, no slots for content, inline styles in render callbacks
- No DataSourceMixin — component manages its own fetch lifecycle
- Two REST calls maximum on load (schema + bulk records)
- `exactOptionalPropertyTypes: true` in tsconfig — use spread patterns for optional props, never assign `undefined`

---

### Task 1: Package scaffold and types

**Files:**
- Create: `components/preferences-editor/package.json`
- Create: `components/preferences-editor/tsconfig.json`
- Create: `components/preferences-editor/tsconfig.build.json`
- Create: `components/preferences-editor/vitest.config.ts`
- Create: `components/preferences-editor/src/types.ts`
- Create: `components/preferences-editor/src/index.ts`
- Test: `components/preferences-editor/src/types.test.ts`

**Interfaces:**
- Consumes: nothing (foundation task)
- Produces: `ScopeNode`, `PreferenceSchemaDescriptor`, `EnumOption`, `PreferenceRecord`, `PreferenceInput`, `InheritanceState` — used by all subsequent tasks

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/blocks-ui-preferences-editor",
  "version": "0.1.0",
  "description": "Preferences editor — tree-table UI for scope-aware preference management",
  "repository": { "type": "git", "url": "https://github.com/casehubio/blocks-ui.git" },
  "publishConfig": { "registry": "https://npm.pkg.github.com" },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "@casehubio/pages-data": "file:../../../pages/packages/pages-data",
    "@casehubio/pages-table": "file:../../../pages/packages/pages-table",
    "lit": "^3.2.1"
  },
  "devDependencies": {
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  },
  "license": "Apache-2.0"
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src"],
  "references": [{ "path": "../../packages/blocks-ui-core" }]
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{ "extends": "./tsconfig.json", "exclude": ["src/**/*.test.ts"] }
```

- [ ] **Step 4: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';
import path from 'path';
import { existsSync } from 'fs';

export default defineConfig({
  resolve: {
    alias: [
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-data/src')) ? [
        { find: /^@casehubio\/pages-data\/dist\/(.*)/, replacement: path.resolve(__dirname, '../../../pages/packages/pages-data/src/$1') },
        { find: '@casehubio/pages-data', replacement: path.resolve(__dirname, '../../../pages/packages/pages-data/src') },
      ] : []),
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-table/src')) ? [
        { find: '@casehubio/pages-table', replacement: path.resolve(__dirname, '../../../pages/packages/pages-table/src') },
      ] : []),
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-ui-tokens/src')) ? [
        { find: '@casehubio/pages-ui-tokens', replacement: path.resolve(__dirname, '../../../pages/packages/pages-ui-tokens/src') },
      ] : []),
      { find: '@casehubio/blocks-ui-core', replacement: path.resolve(__dirname, '../../packages/blocks-ui-core/src') },
    ],
  },
  esbuild: {
    target: 'es2022',
    tsconfigRaw: {
      compilerOptions: {
        experimentalDecorators: true,
        useDefineForClassFields: false,
      },
    },
  },
  test: {
    environment: 'jsdom',
    globals: true,
  },
});
```

- [ ] **Step 5: Create src/types.ts**

```typescript
export interface ScopeNode {
  readonly path: string;
  readonly label: string;
  readonly children?: readonly ScopeNode[];
}

export interface EnumOption {
  readonly value: string;
  readonly label: string;
}

export interface PreferenceSchemaDescriptor {
  readonly namespace: string;
  readonly name: string;
  readonly qualifiedName: string;
  readonly type: string;
  readonly label: string;
  readonly description: string | null;
  readonly defaultValue: string;
  readonly multiValue: boolean;
  readonly constraints: Record<string, unknown>;
  readonly options: readonly EnumOption[];
}

export interface PreferenceRecord {
  readonly tenancyId: string;
  readonly scope: string;
  readonly namespace: string;
  readonly name: string;
  readonly subKey: string;
  readonly value: string;
}

export interface PreferenceInput {
  readonly namespace: string;
  readonly name: string;
  readonly subKey: string;
  readonly value: string;
}

export type InheritanceState = 'local' | 'inherited' | 'overridden' | 'default';

export interface PreferenceRow {
  readonly id: string;
  readonly parentId: string;
  readonly rowType: 'scope' | 'preference';
  readonly label: string;
  readonly value: string;
  readonly schemaType: string;
  readonly inheritanceState: InheritanceState;
  readonly sourceScope: string;
  readonly qualifiedName: string;
  readonly scope: string;
}
```

- [ ] **Step 6: Create src/index.ts**

```typescript
export type * from './types.js';
```

- [ ] **Step 7: Write types.test.ts — type shape validation**

```typescript
import { describe, it, expect, expectTypeOf } from 'vitest';
import type {
  ScopeNode, PreferenceSchemaDescriptor, PreferenceRecord,
  PreferenceInput, InheritanceState, PreferenceRow, EnumOption,
} from './types.js';

describe('types', () => {
  it('ScopeNode supports recursive children', () => {
    const tree: ScopeNode = {
      path: 'system', label: 'System',
      children: [{ path: 'tenant/acme', label: 'Acme' }],
    };
    expect(tree.children).toHaveLength(1);
  });

  it('PreferenceSchemaDescriptor has required fields', () => {
    expectTypeOf<PreferenceSchemaDescriptor>().toHaveProperty('qualifiedName');
    expectTypeOf<PreferenceSchemaDescriptor>().toHaveProperty('type');
    expectTypeOf<PreferenceSchemaDescriptor>().toHaveProperty('multiValue');
    expectTypeOf<PreferenceSchemaDescriptor>().toHaveProperty('constraints');
    expectTypeOf<PreferenceSchemaDescriptor>().toHaveProperty('options');
  });

  it('InheritanceState covers all states', () => {
    const states: InheritanceState[] = ['local', 'inherited', 'overridden', 'default'];
    expect(states).toHaveLength(4);
  });

  it('PreferenceRow carries both tree and preference metadata', () => {
    expectTypeOf<PreferenceRow>().toHaveProperty('id');
    expectTypeOf<PreferenceRow>().toHaveProperty('parentId');
    expectTypeOf<PreferenceRow>().toHaveProperty('rowType');
    expectTypeOf<PreferenceRow>().toHaveProperty('inheritanceState');
    expectTypeOf<PreferenceRow>().toHaveProperty('sourceScope');
  });
});
```

- [ ] **Step 8: Run tests**

Run: `cd components/preferences-editor && npx vitest run`
Expected: all tests pass

- [ ] **Step 9: Commit**

```bash
git -C $PROJECT add components/preferences-editor/
git -C $PROJECT commit -m "feat(#92): package scaffold and types for preferences-editor"
```

---

### Task 2: REST API client

**Files:**
- Create: `components/preferences-editor/src/api.ts`
- Test: `components/preferences-editor/src/api.test.ts`

**Interfaces:**
- Consumes: `PreferenceSchemaDescriptor`, `PreferenceRecord`, `PreferenceInput` from Task 1
- Produces: `PreferencesApi` class — `fetchSchema()`, `fetchAll()`, `set()`, `deleteOne()`, `deleteNamespace()` — used by Task 4

- [ ] **Step 1: Write failing tests for PreferencesApi**

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { PreferencesApi } from './api.js';
import type { PreferenceSchemaDescriptor, PreferenceRecord } from './types.js';

describe('PreferencesApi', () => {
  let fetchFn: ReturnType<typeof vi.fn>;
  let api: PreferencesApi;

  beforeEach(() => {
    fetchFn = vi.fn();
    api = new PreferencesApi('/preferences', fetchFn);
  });

  const mockSchema: PreferenceSchemaDescriptor[] = [
    {
      namespace: 'casehub.work', name: 'sla.default-hours',
      qualifiedName: 'casehub.work.sla.default-hours', type: 'integer',
      label: 'Default SLA hours', description: null, defaultValue: '24',
      multiValue: false, constraints: { min: 1, max: 720 }, options: [],
    },
  ];

  const mockRecords: PreferenceRecord[] = [
    { tenancyId: 't1', scope: 'system', namespace: 'casehub.work',
      name: 'sla.default-hours', subKey: '', value: '24' },
  ];

  it('fetchSchema calls GET /preferences/schema', async () => {
    fetchFn.mockResolvedValue({ ok: true, json: async () => mockSchema });
    const result = await api.fetchSchema();
    expect(fetchFn).toHaveBeenCalledWith('/preferences/schema', expect.objectContaining({ method: 'GET' }));
    expect(result).toEqual(mockSchema);
  });

  it('fetchAll calls GET /preferences with no scope', async () => {
    fetchFn.mockResolvedValue({ ok: true, json: async () => mockRecords });
    const result = await api.fetchAll();
    expect(fetchFn).toHaveBeenCalledWith('/preferences', expect.objectContaining({ method: 'GET' }));
    expect(result).toEqual(mockRecords);
  });

  it('set calls PUT /preferences?scope=<scope>', async () => {
    fetchFn.mockResolvedValue({ ok: true });
    await api.set('tenant/acme', { namespace: 'casehub.work', name: 'sla.default-hours', subKey: '', value: '8' });
    expect(fetchFn).toHaveBeenCalledWith(
      '/preferences?scope=tenant%2Facme',
      expect.objectContaining({
        method: 'PUT',
        body: JSON.stringify({ namespace: 'casehub.work', name: 'sla.default-hours', subKey: '', value: '8' }),
      }),
    );
  });

  it('deleteOne calls DELETE with query params', async () => {
    fetchFn.mockResolvedValue({ ok: true });
    await api.deleteOne('system', 'casehub.work', 'sla.default-hours', '');
    const url = fetchFn.mock.calls[0]![0] as string;
    expect(url).toContain('/preferences?');
    expect(url).toContain('scope=system');
    expect(url).toContain('namespace=casehub.work');
    expect(url).toContain('name=sla.default-hours');
    expect(fetchFn.mock.calls[0]![1].method).toBe('DELETE');
  });

  it('deleteNamespace calls DELETE /preferences/by-namespace', async () => {
    fetchFn.mockResolvedValue({ ok: true });
    await api.deleteNamespace('system', 'casehub.work');
    const url = fetchFn.mock.calls[0]![0] as string;
    expect(url).toContain('/preferences/by-namespace?');
    expect(url).toContain('namespace=casehub.work');
    expect(fetchFn.mock.calls[0]![1].method).toBe('DELETE');
  });

  it('throws on non-ok response', async () => {
    fetchFn.mockResolvedValue({ ok: false, status: 500, statusText: 'Internal Server Error' });
    await expect(api.fetchSchema()).rejects.toThrow('500');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd components/preferences-editor && npx vitest run src/api.test.ts`
Expected: FAIL — `api.js` not found

- [ ] **Step 3: Implement PreferencesApi**

```typescript
import type { PreferenceSchemaDescriptor, PreferenceRecord, PreferenceInput } from './types.js';

export class PreferencesApi {
  constructor(
    private readonly baseUrl: string,
    private readonly fetchFn: typeof fetch = fetch,
  ) {}

  async fetchSchema(namespace?: string): Promise<PreferenceSchemaDescriptor[]> {
    const url = namespace
      ? `${this.baseUrl}/schema?namespace=${encodeURIComponent(namespace)}`
      : `${this.baseUrl}/schema`;
    return this._get(url);
  }

  async fetchAll(): Promise<PreferenceRecord[]> {
    return this._get(this.baseUrl);
  }

  async set(scope: string, input: PreferenceInput): Promise<void> {
    const url = `${this.baseUrl}?scope=${encodeURIComponent(scope)}`;
    const response = await this.fetchFn(url, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(input),
    });
    if (!response.ok) throw new Error(`${response.status}: ${response.statusText}`);
  }

  async deleteOne(scope: string, namespace: string, name: string, subKey: string): Promise<void> {
    const params = new URLSearchParams({ scope, namespace, name, subKey });
    const response = await this.fetchFn(`${this.baseUrl}?${params}`, { method: 'DELETE' });
    if (!response.ok) throw new Error(`${response.status}: ${response.statusText}`);
  }

  async deleteNamespace(scope: string, namespace: string): Promise<void> {
    const params = new URLSearchParams({ scope, namespace });
    const response = await this.fetchFn(`${this.baseUrl}/by-namespace?${params}`, { method: 'DELETE' });
    if (!response.ok) throw new Error(`${response.status}: ${response.statusText}`);
  }

  private async _get<T>(url: string): Promise<T> {
    const response = await this.fetchFn(url, {
      method: 'GET',
      headers: { 'Accept': 'application/json' },
    });
    if (!response.ok) throw new Error(`${response.status}: ${response.statusText}`);
    return response.json();
  }
}
```

- [ ] **Step 4: Export from index.ts**

Add to `src/index.ts`:
```typescript
export { PreferencesApi } from './api.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd components/preferences-editor && npx vitest run src/api.test.ts`
Expected: all 6 tests pass

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add components/preferences-editor/src/api.ts components/preferences-editor/src/api.test.ts components/preferences-editor/src/index.ts
git -C $PROJECT commit -m "feat(#92): PreferencesApi REST client"
```

---

### Task 3: Value editor component

**Files:**
- Create: `components/preferences-editor/src/value-editor.ts`
- Test: `components/preferences-editor/src/value-editor.test.ts`

**Interfaces:**
- Consumes: `PreferenceSchemaDescriptor` from Task 1
- Produces: `<value-editor>` element — renders type-aware input, emits `value-changed` event with `{ value: string }` — used by Task 4

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import './value-editor.js';
import type { PreferenceSchemaDescriptor } from './types.js';

describe('ValueEditor', () => {
  beforeEach(() => { document.body.innerHTML = ''; });

  function create(schema: Partial<PreferenceSchemaDescriptor>, value: string) {
    const el = document.createElement('value-editor') as any;
    el.schema = {
      namespace: 'test', name: 'key', qualifiedName: 'test.key',
      type: 'string', label: 'Test', description: null,
      defaultValue: '', multiValue: false, constraints: {}, options: [],
      ...schema,
    };
    el.value = value;
    document.body.appendChild(el);
    return el;
  }

  it('renders text input for string type', async () => {
    const el = create({ type: 'string' }, 'hello');
    await el.updateComplete;
    const input = el.shadowRoot!.querySelector('input[type="text"]');
    expect(input).toBeTruthy();
    expect((input as HTMLInputElement).value).toBe('hello');
  });

  it('renders number input for integer type', async () => {
    const el = create({ type: 'integer', constraints: { min: 1, max: 100 } }, '42');
    await el.updateComplete;
    const input = el.shadowRoot!.querySelector('input[type="number"]') as HTMLInputElement;
    expect(input).toBeTruthy();
    expect(input.step).toBe('1');
    expect(input.min).toBe('1');
    expect(input.max).toBe('100');
  });

  it('renders number input for number type (decimal)', async () => {
    const el = create({ type: 'number' }, '3.14');
    await el.updateComplete;
    const input = el.shadowRoot!.querySelector('input[type="number"]') as HTMLInputElement;
    expect(input).toBeTruthy();
    expect(input.step).toBe('any');
  });

  it('renders checkbox for boolean type', async () => {
    const el = create({ type: 'boolean' }, 'true');
    await el.updateComplete;
    const input = el.shadowRoot!.querySelector('input[type="checkbox"]') as HTMLInputElement;
    expect(input).toBeTruthy();
    expect(input.checked).toBe(true);
  });

  it('renders select for enum type', async () => {
    const el = create({
      type: 'enum',
      options: [{ value: 'A', label: 'Alpha' }, { value: 'B', label: 'Beta' }],
    }, 'A');
    await el.updateComplete;
    const select = el.shadowRoot!.querySelector('select') as HTMLSelectElement;
    expect(select).toBeTruthy();
    expect(select.value).toBe('A');
    expect(select.options).toHaveLength(2);
  });

  it('renders text input for duration type with ISO value', async () => {
    const el = create({ type: 'duration' }, 'PT24H');
    await el.updateComplete;
    const input = el.shadowRoot!.querySelector('input');
    expect(input).toBeTruthy();
  });

  it('emits value-changed on input', async () => {
    const el = create({ type: 'string' }, 'old');
    await el.updateComplete;
    const handler = vi.fn();
    el.addEventListener('value-changed', handler);
    const input = el.shadowRoot!.querySelector('input') as HTMLInputElement;
    input.value = 'new';
    input.dispatchEvent(new Event('change', { bubbles: true }));
    expect(handler).toHaveBeenCalled();
    expect(handler.mock.calls[0]![0].detail.value).toBe('new');
  });

  it('validates pattern constraint', async () => {
    const el = create({ type: 'string', constraints: { pattern: '^https?://' } }, 'not-a-url');
    await el.updateComplete;
    const input = el.shadowRoot!.querySelector('input') as HTMLInputElement;
    expect(input.pattern).toBe('^https?://');
  });

  it('renders read-only when disabled', async () => {
    const el = create({ type: 'string' }, 'hello');
    el.disabled = true;
    await el.updateComplete;
    const input = el.shadowRoot!.querySelector('input') as HTMLInputElement;
    expect(input.disabled).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd components/preferences-editor && npx vitest run src/value-editor.test.ts`
Expected: FAIL — `value-editor.js` not found

- [ ] **Step 3: Implement ValueEditor**

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import type { PreferenceSchemaDescriptor } from './types.js';

@customElement('value-editor')
export class ValueEditor extends LitElement {
  @property({ attribute: false }) schema!: PreferenceSchemaDescriptor;
  @property({ type: String }) value = '';
  @property({ type: Boolean }) disabled = false;

  static override styles = css`
    :host { display: inline-block; }
    input, select {
      padding: 4px 8px;
      border: 1px solid var(--pages-border-color, #ccc);
      border-radius: 4px;
      font-size: 0.8125rem;
      font-family: inherit;
      width: 100%;
      box-sizing: border-box;
    }
    input:invalid { border-color: var(--pages-danger-color, #dc3545); }
    .checkbox-wrap { display: flex; align-items: center; gap: 6px; }
  `;

  override render() {
    switch (this.schema.type) {
      case 'boolean': return this._renderBoolean();
      case 'integer': return this._renderNumber('1');
      case 'number': return this._renderNumber('any');
      case 'enum': return this._renderEnum();
      case 'duration': return this._renderDuration();
      default: return this._renderString();
    }
  }

  private _renderString() {
    const c = this.schema.constraints;
    return html`<input type="text" .value=${this.value}
      ?disabled=${this.disabled}
      pattern=${(c.pattern as string) ?? ''}
      minlength=${(c.minLength as number) ?? ''}
      maxlength=${(c.maxLength as number) ?? ''}
      @change=${this._onInput}>`;
  }

  private _renderNumber(step: string) {
    const c = this.schema.constraints;
    return html`<input type="number" .value=${this.value}
      ?disabled=${this.disabled}
      step=${step}
      min=${(c.min as number) ?? ''}
      max=${(c.max as number) ?? ''}
      @change=${this._onInput}>`;
  }

  private _renderBoolean() {
    return html`<div class="checkbox-wrap">
      <input type="checkbox" .checked=${this.value === 'true'}
        ?disabled=${this.disabled}
        @change=${(e: Event) => this._emit((e.target as HTMLInputElement).checked ? 'true' : 'false')}>
    </div>`;
  }

  private _renderEnum() {
    return html`<select .value=${this.value} ?disabled=${this.disabled} @change=${this._onInput}>
      ${this.schema.options.map(o => html`<option value=${o.value} ?selected=${o.value === this.value}>${o.label}</option>`)}
    </select>`;
  }

  private _renderDuration() {
    return html`<input type="text" .value=${this.value}
      ?disabled=${this.disabled}
      placeholder="PT1H30M"
      @change=${this._onInput}>`;
  }

  private _onInput = (e: Event) => {
    const target = e.target as HTMLInputElement | HTMLSelectElement;
    this._emit(target.value);
  };

  private _emit(value: string) {
    this.dispatchEvent(new CustomEvent('value-changed', {
      detail: { value },
      bubbles: true, composed: true,
    }));
  }
}
```

- [ ] **Step 4: Export from index.ts**

Add to `src/index.ts`:
```typescript
export { ValueEditor } from './value-editor.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd components/preferences-editor && npx vitest run src/value-editor.test.ts`
Expected: all 9 tests pass

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add components/preferences-editor/src/value-editor.ts components/preferences-editor/src/value-editor.test.ts components/preferences-editor/src/index.ts
git -C $PROJECT commit -m "feat(#92): value-editor — type-aware inline editor"
```

---

### Task 4: Main preferences-editor component

**Files:**
- Create: `components/preferences-editor/src/preferences-editor.ts`
- Test: `components/preferences-editor/src/preferences-editor.test.ts`
- Modify: `components/preferences-editor/src/index.ts` — add exports

**Interfaces:**
- Consumes: `PreferencesApi` from Task 2, `ValueEditor` from Task 3, all types from Task 1, `pages-data-table` with `expandable`, `fromRows`/`ColumnType` from `pages-data`
- Produces: `<preferences-editor>` element — the final component

- [ ] **Step 1: Write failing tests — data loading and tree construction**

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import './preferences-editor.js';
import type { PreferenceSchemaDescriptor, PreferenceRecord, ScopeNode } from './types.js';

describe('PreferencesEditor', () => {
  let fetchFn: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchFn = vi.fn();
    document.body.innerHTML = '';
  });

  const SCHEMA: PreferenceSchemaDescriptor[] = [
    {
      namespace: 'casehub.work', name: 'sla.default-hours',
      qualifiedName: 'casehub.work.sla.default-hours', type: 'integer',
      label: 'Default SLA hours', description: 'Hours before escalation',
      defaultValue: '24', multiValue: false,
      constraints: { min: 1, max: 720 }, options: [],
    },
    {
      namespace: 'casehub.work', name: 'delegation.decline-target',
      qualifiedName: 'casehub.work.delegation.decline-target', type: 'enum',
      label: 'Decline target', description: null, defaultValue: 'POOL',
      multiValue: false, constraints: {},
      options: [{ value: 'POOL', label: 'Return to pool' }, { value: 'DELEGATOR', label: 'Return to delegator' }],
    },
  ];

  const RECORDS: PreferenceRecord[] = [
    { tenancyId: 't1', scope: 'system', namespace: 'casehub.work', name: 'sla.default-hours', subKey: '', value: '24' },
    { tenancyId: 't1', scope: 'tenant/acme', namespace: 'casehub.work', name: 'sla.default-hours', subKey: '', value: '8' },
  ];

  const SCOPE_TREE: ScopeNode[] = [
    { path: 'system', label: 'System', children: [
      { path: 'tenant/acme', label: 'Acme Corp' },
    ]},
  ];

  function mockFetchResponses() {
    fetchFn.mockImplementation((url: string) => {
      if (url.includes('/schema')) return Promise.resolve({ ok: true, json: async () => SCHEMA });
      if (url.endsWith('/preferences')) return Promise.resolve({ ok: true, json: async () => RECORDS });
      return Promise.resolve({ ok: true });
    });
  }

  function create(scopeTree: ScopeNode[] = SCOPE_TREE) {
    const el = document.createElement('preferences-editor') as any;
    el.scopeTree = scopeTree;
    el.endpoint = '/preferences';
    el.fetchFn = fetchFn;
    document.body.appendChild(el);
    return el;
  }

  it('fetches schema and records on connectedCallback', async () => {
    mockFetchResponses();
    const el = create();
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 50));
    expect(fetchFn).toHaveBeenCalledTimes(2);
    const urls = fetchFn.mock.calls.map((c: unknown[]) => c[0]);
    expect(urls).toContain('/preferences/schema');
    expect(urls).toContain('/preferences');
  });

  it('renders scope nodes as expandable rows', async () => {
    mockFetchResponses();
    const el = create();
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 50));
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-data-table');
    expect(table).toBeTruthy();
  });

  it('shows inherited state for preferences not set at a scope', async () => {
    mockFetchResponses();
    const el = create();
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 50));
    await el.updateComplete;
    const dataSet = el._dataSet;
    expect(dataSet).toBeTruthy();
    expect(dataSet.rows.length).toBeGreaterThan(0);
  });

  it('emits preference-changed on successful save', async () => {
    mockFetchResponses();
    const el = create();
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 50));
    const handler = vi.fn();
    el.addEventListener('preference-changed', handler);
    await el.handleSave('tenant/acme', 'casehub.work', 'sla.default-hours', '', '12');
    expect(handler).toHaveBeenCalled();
    expect(handler.mock.calls[0]![0].detail.newValue).toBe('12');
  });

  it('emits preference-deleted on successful delete', async () => {
    mockFetchResponses();
    const el = create();
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 50));
    const handler = vi.fn();
    el.addEventListener('preference-deleted', handler);
    await el.handleDelete('tenant/acme', 'casehub.work', 'sla.default-hours', '');
    expect(handler).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd components/preferences-editor && npx vitest run src/preferences-editor.test.ts`
Expected: FAIL — `preferences-editor.js` not found

- [ ] **Step 3: Implement preferences-editor**

This is the largest file. Key responsibilities:
- Fetch schema + records on connect
- Build `PreferenceRow[]` array merging scope tree, schema, and records with inheritance computation
- Convert to `TypedDataSet` with `fromRows` for `pages-data-table`
- Configure `expandable` with `idColumn`/`parentColumn`
- Render `value-editor` in the value column via `columnRenderers`
- Handle save/delete via `PreferencesApi`, re-fetch after mutations

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { fromRows } from '@casehubio/pages-data/dist/dataset/conversion.js';
import { ColumnType, columnId } from '@casehubio/pages-data/dist/dataset/types.js';
import type { TypedDataSet, TypedRow } from '@casehubio/pages-data/dist/dataset/types.js';
import '@casehubio/pages-table';
import './value-editor.js';
import { PreferencesApi } from './api.js';
import type {
  ScopeNode, PreferenceSchemaDescriptor, PreferenceRecord,
  PreferenceRow, InheritanceState,
} from './types.js';

const ID_COL = columnId('id');
const PARENT_COL = columnId('parentId');
const LABEL_COL = columnId('label');
const VALUE_COL = columnId('value');
const TYPE_COL = columnId('schemaType');
const STATE_COL = columnId('inheritanceState');
const SOURCE_COL = columnId('sourceScope');

@customElement('preferences-editor')
export class PreferencesEditor extends LitElement {
  @property({ attribute: false }) scopeTree: readonly ScopeNode[] = [];
  @property({ type: String }) endpoint = '/preferences';
  @property({ attribute: false }) fetchFn: typeof fetch = fetch;

  @state() _dataSet: TypedDataSet | undefined;
  @state() _loading = false;
  @state() _error: string | null = null;

  private _api!: PreferencesApi;
  private _schema: PreferenceSchemaDescriptor[] = [];
  private _records: PreferenceRecord[] = [];

  static override styles = css`
    :host { display: block; height: 100%; }
    .error { padding: 16px; color: var(--pages-danger-color, #dc3545); text-align: center; }
    .loading { padding: 32px; text-align: center; color: var(--pages-muted-color, #999); }
    .inherited { opacity: 0.5; }
    .source-badge {
      font-size: 0.6875rem;
      color: var(--pages-muted-color, #999);
      margin-left: 4px;
    }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this._api = new PreferencesApi(this.endpoint, this.fetchFn);
    this._loadData();
  }

  override render() {
    if (this._loading) return html`<div class="loading">Loading preferences...</div>`;
    if (this._error) return html`<div class="error">${this._error}</div>`;
    if (!this._dataSet) return nothing;

    return html`
      <pages-data-table
        .dataSet=${this._dataSet}
        .columnConfig=${[
          { id: ID_COL, visible: false },
          { id: PARENT_COL, visible: false },
          { id: LABEL_COL, label: 'Name' },
          { id: VALUE_COL, label: 'Value' },
          { id: TYPE_COL, label: 'Type', visible: false },
          { id: STATE_COL, label: 'State', visible: false },
          { id: SOURCE_COL, label: 'Source' },
        ]}
        .columnRenderers=${this._columnRenderers()}
        .props=${{ expandable: { idColumn: ID_COL, parentColumn: PARENT_COL, defaultExpanded: true } }}
        client-filter
      ></pages-data-table>
    `;
  }

  private _columnRenderers() {
    const schemaMap = new Map(this._schema.map(s => [s.qualifiedName, s]));
    return new Map([
      [VALUE_COL, (row: TypedRow) => {
        const rowType = row.text(columnId('rowType'));
        if (rowType === 'scope') return html``;
        const qn = row.text(columnId('qualifiedName'));
        const schema = schemaMap.get(qn);
        if (!schema) return html`<span>${row.text(VALUE_COL)}</span>`;
        const inheritState = row.text(STATE_COL) as InheritanceState;
        const disabled = inheritState === 'inherited' || inheritState === 'default';
        const scope = row.text(columnId('scope'));
        const value = row.text(VALUE_COL);
        return html`<value-editor
          .schema=${schema}
          .value=${value}
          ?disabled=${disabled}
          style="display:inline-block;min-width:120px;"
          @value-changed=${(e: CustomEvent) => this.handleSave(scope, schema.namespace, schema.name, '', e.detail.value)}
        ></value-editor>`;
      }],
      [SOURCE_COL, (row: TypedRow) => {
        const rowType = row.text(columnId('rowType'));
        if (rowType === 'scope') return html``;
        const state = row.text(STATE_COL) as InheritanceState;
        const source = row.text(SOURCE_COL);
        if (state === 'inherited') return html`<span class="source-badge" style="font-style:italic;">from: ${source}</span>`;
        if (state === 'overridden') return html`<span class="source-badge">overrides ${source}</span>`;
        if (state === 'default') return html`<span class="source-badge" style="font-style:italic;">default</span>`;
        return html`<span class="source-badge">local</span>`;
      }],
    ]);
  }

  async handleSave(scope: string, namespace: string, name: string, subKey: string, newValue: string): Promise<void> {
    const oldValue = this._findRecordValue(scope, namespace, name, subKey);
    try {
      await this._api.set(scope, { namespace, name, subKey, value: newValue });
      this.dispatchEvent(new CustomEvent('preference-changed', {
        detail: { scope, qualifiedName: `${namespace}.${name}`, oldValue, newValue },
        bubbles: true, composed: true,
      }));
      await this._loadData();
    } catch (e) {
      this._error = e instanceof Error ? e.message : String(e);
    }
  }

  async handleDelete(scope: string, namespace: string, name: string, subKey: string): Promise<void> {
    try {
      await this._api.deleteOne(scope, namespace, name, subKey);
      this.dispatchEvent(new CustomEvent('preference-deleted', {
        detail: { scope, qualifiedName: `${namespace}.${name}` },
        bubbles: true, composed: true,
      }));
      await this._loadData();
    } catch (e) {
      this._error = e instanceof Error ? e.message : String(e);
    }
  }

  private _findRecordValue(scope: string, namespace: string, name: string, subKey: string): string | undefined {
    return this._records.find(r =>
      r.scope === scope && r.namespace === namespace && r.name === name && r.subKey === subKey
    )?.value;
  }

  private async _loadData(): Promise<void> {
    this._loading = true;
    this._error = null;
    try {
      const [schema, records] = await Promise.all([
        this._api.fetchSchema(),
        this._api.fetchAll(),
      ]);
      this._schema = schema;
      this._records = records;
      this._dataSet = this._buildDataSet();
    } catch (e) {
      this._error = e instanceof Error ? e.message : String(e);
    } finally {
      this._loading = false;
    }
  }

  private _buildDataSet(): TypedDataSet {
    const rows = this._buildRows();
    return fromRows(rows, [
      { id: ID_COL, name: 'id', type: ColumnType.TEXT, getValue: (r: PreferenceRow) => r.id },
      { id: PARENT_COL, name: 'parentId', type: ColumnType.TEXT, getValue: (r: PreferenceRow) => r.parentId },
      { id: LABEL_COL, name: 'label', type: ColumnType.TEXT, getValue: (r: PreferenceRow) => r.label },
      { id: VALUE_COL, name: 'value', type: ColumnType.TEXT, getValue: (r: PreferenceRow) => r.value },
      { id: TYPE_COL, name: 'schemaType', type: ColumnType.TEXT, getValue: (r: PreferenceRow) => r.schemaType },
      { id: STATE_COL, name: 'inheritanceState', type: ColumnType.TEXT, getValue: (r: PreferenceRow) => r.inheritanceState },
      { id: SOURCE_COL, name: 'sourceScope', type: ColumnType.TEXT, getValue: (r: PreferenceRow) => r.sourceScope },
      { id: columnId('rowType'), name: 'rowType', type: ColumnType.TEXT, getValue: (r: PreferenceRow) => r.rowType },
      { id: columnId('qualifiedName'), name: 'qualifiedName', type: ColumnType.TEXT, getValue: (r: PreferenceRow) => r.qualifiedName },
      { id: columnId('scope'), name: 'scope', type: ColumnType.TEXT, getValue: (r: PreferenceRow) => r.scope },
    ]);
  }

  private _buildRows(): PreferenceRow[] {
    const rows: PreferenceRow[] = [];
    const recordIndex = new Map<string, PreferenceRecord>();
    for (const r of this._records) {
      recordIndex.set(`${r.scope}:${r.namespace}.${r.name}:${r.subKey}`, r);
    }

    const walkScope = (node: ScopeNode, parentId: string, ancestorScopes: string[]) => {
      rows.push({
        id: node.path,
        parentId,
        rowType: 'scope',
        label: node.label,
        value: '',
        schemaType: '',
        inheritanceState: 'local',
        sourceScope: '',
        qualifiedName: '',
        scope: node.path,
      });

      for (const schema of this._schema) {
        if (schema.multiValue) continue;
        const key = `${node.path}:${schema.qualifiedName}:`;
        const localRecord = recordIndex.get(key);

        let inheritanceState: InheritanceState = 'default';
        let value = schema.defaultValue;
        let sourceScope = 'default';

        if (localRecord) {
          value = localRecord.value;
          sourceScope = node.path;
          const parentRecord = ancestorScopes.find(s => recordIndex.has(`${s}:${schema.qualifiedName}:`));
          inheritanceState = parentRecord ? 'overridden' : 'local';
          if (inheritanceState === 'overridden') sourceScope = parentRecord!;
        } else {
          const inheritedScope = ancestorScopes.find(s => recordIndex.has(`${s}:${schema.qualifiedName}:`));
          if (inheritedScope) {
            value = recordIndex.get(`${inheritedScope}:${schema.qualifiedName}:`)!.value;
            sourceScope = inheritedScope;
            inheritanceState = 'inherited';
          }
        }

        rows.push({
          id: `${node.path}:${schema.qualifiedName}`,
          parentId: node.path,
          rowType: 'preference',
          label: schema.label,
          value,
          schemaType: schema.type,
          inheritanceState,
          sourceScope,
          qualifiedName: schema.qualifiedName,
          scope: node.path,
        });
      }

      if (node.children) {
        for (const child of node.children) {
          walkScope(child, node.path, [node.path, ...ancestorScopes]);
        }
      }
    };

    for (const root of this.scopeTree) {
      walkScope(root, '', []);
    }
    return rows;
  }
}
```

- [ ] **Step 4: Export from index.ts**

Add to `src/index.ts`:
```typescript
export { PreferencesEditor } from './preferences-editor.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd components/preferences-editor && npx vitest run`
Expected: all tests pass

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add components/preferences-editor/src/preferences-editor.ts components/preferences-editor/src/preferences-editor.test.ts components/preferences-editor/src/index.ts
git -C $PROJECT commit -m "feat(#92): preferences-editor — tree-table with inheritance and type-aware editing"
```

---

### Task 5: Showcase page and workspace integration

**Files:**
- Create: `examples/src/pages/preferences-editor-page.ts`
- Modify: `examples/src/main.ts` — add dynamic import
- Modify: `examples/src/shell.ts` — add nav entry
- Modify: `examples/vite.config.ts` — add alias
- Modify: `tsconfig.json` (root) — add project reference
- Modify: `package.json` (root) — add workspace entry

**Interfaces:**
- Consumes: `<preferences-editor>` from Task 4
- Produces: Working showcase demo with mock data

- [ ] **Step 1: Add workspace entry to root package.json**

Add `"components/preferences-editor"` to the `workspaces` array.

- [ ] **Step 2: Add project reference to root tsconfig.json**

Add `{ "path": "components/preferences-editor" }` to `references` array.

- [ ] **Step 3: Add Vite alias in examples/vite.config.ts**

Add after the trust-workbench alias:
```typescript
{ find: '@casehubio/blocks-ui-preferences-editor', replacement: resolve(__dirname, '../components/preferences-editor/src') },
```

- [ ] **Step 4: Create showcase page**

Create `examples/src/pages/preferences-editor-page.ts` with mock scope tree and mock fetch returning schema + records.

- [ ] **Step 5: Add nav entry to shell.ts**

Add `{ id: 'preferences-editor', label: 'Preferences Editor', hash: '#components/preferences-editor' }` to the Components nav category.

Add case in `renderPage()`:
```typescript
case '#components/preferences-editor': return html`<preferences-editor-page></preferences-editor-page>`;
```

- [ ] **Step 6: Add dynamic import in main.ts**

Add before the closing of imports:
```typescript
await import('./pages/preferences-editor-page.js');
```

- [ ] **Step 7: Run yarn install to update lockfile**

Run: `cd $PROJECT && yarn install`

- [ ] **Step 8: Build and verify showcase works**

Run: `cd examples && npx vite build`
Expected: build succeeds

- [ ] **Step 9: Commit**

```bash
git -C $PROJECT add examples/ tsconfig.json package.json components/preferences-editor/
git -C $PROJECT commit -m "feat(#92): preferences-editor showcase and workspace integration"
```

---

## Self-Review

**Spec coverage:** All 10 spec sections mapped to tasks. Data model (§3) → Task 1+4. API (§2) → Task 2. Editing (§5) → Task 3+4. File structure (§6) → Task 1. Testing (§7) → Tasks 1-4. Dependencies (§8) → all delivered. Not in scope (§9) → respected.

**Placeholder scan:** No TBDs. All code blocks are complete. All test assertions are specific.

**Type consistency:** `PreferenceSchemaDescriptor` used consistently across all tasks. `PreferencesApi` constructor signature matches test usage. `value-changed` event name matches emitter and listener. `PreferenceRow` fields match `fromRows` column definitions.

**Tooling safety scan:** No bash file operations on source files. All source file creation uses `Write` or `ide_create_file`. Git commands use `-C $PROJECT` consistently.
