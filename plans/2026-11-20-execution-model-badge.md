# Execution Model Badge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #295 — Execution model badge on case overview diagram

**Goal:** Show a colour-coded document-type badge in the case diagram toolbar, extracted from the `name.type.yaml` filename convention with content-based fallback.

**Architecture:** Two detection functions in mosaic-core (`detectDiagramTypeFromSrc` for filename, extended `detectDiagramType` for content). The toolbar renders a styled pill. `casehub-diagram` resolves the type and passes it down. `DIAGRAM_TAGS` and `diagram-workbench` updated to handle the `case` → `choreography` rename.

**Tech Stack:** TypeScript, Lit, Vitest, `@casehubio/mosaic-core`, `@casehubio/pages-ui-components`

## Global Constraints

- ARIA required on every element: `role` + `aria-label` minimum
- Use `--pages-*` CSS custom properties — no hardcoded dark fallbacks
- Return type of detection functions is `string`, not a union literal (open type set)
- Keep `case` in `DIAGRAM_TAGS` for backward compat alongside new `choreography` alias

---

## Batch 1: Detection functions and tag map

### Task 1: Add `detectDiagramTypeFromSrc` and extend `detectDiagramType`

**Files:**
- Modify: `packages/mosaic-core/src/diagram-type.ts`
- Modify: `packages/mosaic-core/src/diagram-type.test.ts`
- Modify: `packages/mosaic-core/src/index.ts`

**Interfaces:**
- Produces: `detectDiagramTypeFromSrc(src: string): string | undefined` — extracts type from `name.type.yaml` filename pattern
- Produces: `detectDiagramType(yaml: string): string` — now returns `choreography` instead of `case`, adds `goap` detection

- [ ] **Step 1: Write failing tests for `detectDiagramTypeFromSrc`**

Add to `packages/mosaic-core/src/diagram-type.test.ts`:

```typescript
import { detectDiagramType, detectDiagramTypeFromSrc } from './diagram-type.js';

describe('detectDiagramTypeFromSrc', () => {
  it('extracts type from name.type.yaml pattern', () => {
    expect(detectDiagramTypeFromSrc('mycase.choreography.yaml')).toBe('choreography');
    expect(detectDiagramTypeFromSrc('plan.htn.yaml')).toBe('htn');
    expect(detectDiagramTypeFromSrc('bot.agent-conf.yaml')).toBe('agent-conf');
    expect(detectDiagramTypeFromSrc('flow.swf.yml')).toBe('swf');
  });

  it('returns undefined for single-segment filenames', () => {
    expect(detectDiagramTypeFromSrc('legacy.yaml')).toBeUndefined();
    expect(detectDiagramTypeFromSrc('file.yml')).toBeUndefined();
  });

  it('returns undefined for empty or missing src', () => {
    expect(detectDiagramTypeFromSrc('')).toBeUndefined();
  });

  it('extracts from URL paths', () => {
    expect(detectDiagramTypeFromSrc('/api/definitions/mycase.choreography.yaml')).toBe('choreography');
    expect(detectDiagramTypeFromSrc('https://example.com/defs/plan.htn.yaml')).toBe('htn');
  });

  it('ignores query strings and fragments', () => {
    expect(detectDiagramTypeFromSrc('mycase.goap.yaml?v=2')).toBe('goap');
    expect(detectDiagramTypeFromSrc('mycase.llm.yaml#section')).toBe('llm');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/mosaic-core/src/diagram-type.test.ts`
Expected: FAIL — `detectDiagramTypeFromSrc` is not exported

- [ ] **Step 3: Implement `detectDiagramTypeFromSrc`**

Add to `packages/mosaic-core/src/diagram-type.ts`:

```typescript
export function detectDiagramTypeFromSrc(src: string): string | undefined {
  if (!src) return undefined;
  const filename = src.split('/').pop()!.split('?')[0]!.split('#')[0]!;
  const segments = filename.split('.');
  if (segments.length < 3) return undefined;
  const ext = segments[segments.length - 1];
  if (ext !== 'yaml' && ext !== 'yml') return undefined;
  return segments[segments.length - 2];
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/mosaic-core/src/diagram-type.test.ts`
Expected: `detectDiagramTypeFromSrc` tests PASS, existing `detectDiagramType` tests still pass

- [ ] **Step 5: Write failing tests for extended `detectDiagramType`**

Update the existing test in `packages/mosaic-core/src/diagram-type.test.ts`:

```typescript
  it('detects case definition as choreography', () => {
    expect(detectDiagramType(CASE_YAML)).toBe('choreography');
  });

  it('detects GOAP definition', () => {
    const GOAP_YAML = `dsl: "1.0.0"
namespace: test
name: my-case
spec:
  bindings:
    - name: b1
      capability: cap1
  workers:
    - name: w1
      capabilities: [cap1]
  goals:
    - name: resolve-case
      expression:
        all: [b1]
`;
    expect(detectDiagramType(GOAP_YAML)).toBe('goap');
  });
```

Replace the existing `'detects case definition'` test with the `'detects case definition as choreography'` test.

- [ ] **Step 6: Run tests to verify they fail**

Run: `yarn vitest run packages/mosaic-core/src/diagram-type.test.ts`
Expected: FAIL — `'case'` !== `'choreography'`, and `goap` test fails

- [ ] **Step 7: Update `detectDiagramType` implementation**

Replace the function body in `packages/mosaic-core/src/diagram-type.ts`:

```typescript
export function detectDiagramType(yaml: string): string {
  try {
    const parsed = parseYaml(yaml) as Record<string, unknown> | null;
    if (!parsed || typeof parsed !== 'object') return 'unknown';

    const spec = parsed['spec'] as Record<string, unknown> | undefined;
    if (spec?.['decomposition']) return 'htn';
    if (spec?.['goals']) return 'goap';
    if (spec && (spec['bindings'] || spec['workers'])) return 'choreography';

    if (parsed['document'] && parsed['do']) return 'swf';

    return 'unknown';
  } catch {
    return 'unknown';
  }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `yarn vitest run packages/mosaic-core/src/diagram-type.test.ts`
Expected: ALL PASS

- [ ] **Step 9: Export `detectDiagramTypeFromSrc` from index**

Modify `packages/mosaic-core/src/index.ts`:

```typescript
export { detectDiagramType, detectDiagramTypeFromSrc } from './diagram-type.js';
```

- [ ] **Step 10: Commit**

```bash
git add packages/mosaic-core/src/diagram-type.ts packages/mosaic-core/src/diagram-type.test.ts packages/mosaic-core/src/index.ts
git commit -m "feat(#295): extend diagram type detection — add filename extraction, goap, choreography

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Update `DIAGRAM_TAGS` and its test

**Files:**
- Modify: `packages/mosaic-core/src/diagram-tags.ts`
- Modify: `packages/mosaic-core/src/diagram-tags.test.ts`

**Interfaces:**
- Consumes: new type strings `choreography`, `goap` from Task 1
- Produces: `DIAGRAM_TAGS` with `choreography` and `goap` entries pointing to `'casehub-diagram'`

- [ ] **Step 1: Write failing test**

Update `packages/mosaic-core/src/diagram-tags.test.ts`:

```typescript
  it('maps all domain formats including aliases', () => {
    expect(DIAGRAM_TAGS).toEqual({
      case: 'casehub-diagram',
      choreography: 'casehub-diagram',
      goap: 'casehub-diagram',
      swf: 'swf-diagram',
      htn: 'htn-diagram',
      org: 'agency-org-diagram',
    });
  });
```

Replace the existing `'maps all four domain formats'` test.

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/mosaic-core/src/diagram-tags.test.ts`
Expected: FAIL — missing `choreography` and `goap` entries

- [ ] **Step 3: Update `DIAGRAM_TAGS`**

Replace contents of `packages/mosaic-core/src/diagram-tags.ts`:

```typescript
export const DIAGRAM_TAGS: Record<string, string> = {
  case: 'casehub-diagram',
  choreography: 'casehub-diagram',
  goap: 'casehub-diagram',
  swf: 'swf-diagram',
  htn: 'htn-diagram',
  org: 'agency-org-diagram',
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn vitest run packages/mosaic-core/src/diagram-tags.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/mosaic-core/src/diagram-tags.ts packages/mosaic-core/src/diagram-tags.test.ts
git commit -m "feat(#295): add choreography and goap aliases to DIAGRAM_TAGS

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Toolbar badge and wiring

### Task 3: Add `diagramType` badge to toolbar

**Files:**
- Modify: `components/casehub-diagram/src/casehub-diagram-toolbar.ts`
- Modify: `components/casehub-diagram/src/casehub-diagram-toolbar.test.ts`

**Interfaces:**
- Consumes: `diagramType` string property set by parent `casehub-diagram`
- Produces: rendered badge pill in toolbar shadow DOM, class `.diagram-type-badge`

- [ ] **Step 1: Write failing tests**

Add to `components/casehub-diagram/src/casehub-diagram-toolbar.test.ts`:

```typescript
describe('diagram type badge', () => {
  let el: ToolbarEl;

  beforeEach(async () => {
    el = document.createElement('casehub-diagram-toolbar') as ToolbarEl;
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => {
    el.remove();
  });

  it('renders badge with correct text for known type', async () => {
    (el as any).diagramType = 'choreography';
    await el.updateComplete;
    const badge = el.shadowRoot!.querySelector('.diagram-type-badge');
    expect(badge).not.toBeNull();
    expect(badge!.textContent!.trim()).toBe('choreography');
  });

  it('hides badge when diagramType is empty', async () => {
    (el as any).diagramType = '';
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.diagram-type-badge')).toBeNull();
  });

  it('hides badge when diagramType is unknown', async () => {
    (el as any).diagramType = 'unknown';
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.diagram-type-badge')).toBeNull();
  });

  it('renders badge for any string type (open set)', async () => {
    (el as any).diagramType = 'avatar-conf';
    await el.updateComplete;
    const badge = el.shadowRoot!.querySelector('.diagram-type-badge');
    expect(badge).not.toBeNull();
    expect(badge!.textContent!.trim()).toBe('avatar-conf');
  });

  it('has correct ARIA attributes', async () => {
    (el as any).diagramType = 'htn';
    await el.updateComplete;
    const badge = el.shadowRoot!.querySelector('.diagram-type-badge')!;
    expect(badge.getAttribute('role')).toBe('status');
    expect(badge.getAttribute('aria-label')).toBe('Document type: htn');
  });
});
```

Update the `ToolbarEl` type to include `diagramType`:

```typescript
type ToolbarEl = HTMLElement & {
  dirty: boolean;
  saving: boolean;
  hasBackend: boolean;
  runtimeAvailable: boolean;
  mode: 'design' | 'runtime';
  staleSeconds: number;
  diagramType: string;
  updateComplete: Promise<boolean>;
};
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run components/casehub-diagram/src/casehub-diagram-toolbar.test.ts`
Expected: FAIL — no `.diagram-type-badge` in DOM

- [ ] **Step 3: Implement badge in toolbar**

Add to `components/casehub-diagram/src/casehub-diagram-toolbar.ts`:

Property:

```typescript
@property({ type: String }) diagramType = '';
```

Colour map (add as a module-level const):

```typescript
const DIAGRAM_TYPE_COLORS: Record<string, string> = {
  choreography: '#3b82f6',
  htn: '#8b5cf6',
  goap: '#d97706',
  llm: '#0d9488',
  swf: '#ea580c',
  playbook: '#16a34a',
  'agent-conf': '#64748b',
  'avatar-conf': '#64748b',
  agency: '#64748b',
};
const DEFAULT_BADGE_COLOR = '#6b7280';
```

Add CSS class in the `static styles`:

```css
.diagram-type-badge {
  font-size: 11px;
  padding: 1px 8px;
  border-radius: 10px;
  color: #fff;
  text-transform: lowercase;
  letter-spacing: 0.3px;
  white-space: nowrap;
}
```

Add badge rendering in `render()`, after `exportSection` and before `modeSection`:

```typescript
const typeBadge = this.diagramType && this.diagramType !== 'unknown' ? html`
  <span class="diagram-type-badge"
    role="status"
    aria-label="Document type: ${this.diagramType}"
    style="background: ${DIAGRAM_TYPE_COLORS[this.diagramType] ?? DEFAULT_BADGE_COLOR}">
    ${this.diagramType}
  </span>
` : nothing;
```

Update the return to include it:

```typescript
return html`${saveSection}${exportSection}${typeBadge}${modeSection}`;
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run components/casehub-diagram/src/casehub-diagram-toolbar.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add components/casehub-diagram/src/casehub-diagram-toolbar.ts components/casehub-diagram/src/casehub-diagram-toolbar.test.ts
git commit -m "feat(#295): add diagram type badge to toolbar

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 4: Wire type resolution through `casehub-diagram` and update downstream consumers

**Files:**
- Modify: `components/casehub-diagram/src/casehub-diagram.ts`
- Modify: `components/diagram-workbench/src/diagram-workbench.ts`
- Modify: `examples/src/pages/diagram-export-page.ts`

**Interfaces:**
- Consumes: `detectDiagramTypeFromSrc` and `detectDiagramType` from mosaic-core (Task 1)
- Consumes: `diagramType` property on toolbar (Task 3)

- [ ] **Step 1: Wire type resolution in `casehub-diagram`**

In `components/casehub-diagram/src/casehub-diagram.ts`, add the import:

```typescript
import { detectDiagramType, detectDiagramTypeFromSrc } from '@casehubio/mosaic-core';
```

(Replace the existing `import { detectDiagramType } from '@casehubio/mosaic-core';` line.)

In the `render()` method, add type resolution before the toolbar element and pass it:

```typescript
const resolvedType = detectDiagramTypeFromSrc(this.src) ?? detectDiagramType(this._currentYaml);
```

Add `.diagramType=${resolvedType}` to the `<casehub-diagram-toolbar>` element:

```html
<casehub-diagram-toolbar
  .diagramType=${resolvedType}
  ?hasBackend=${this.backend != null}
  ...
></casehub-diagram-toolbar>
```

- [ ] **Step 2: Update `diagram-workbench` root level**

In `components/diagram-workbench/src/diagram-workbench.ts`, change line 114 from:

```typescript
diagramType: 'case',
```

to:

```typescript
diagramType: 'choreography',
```

- [ ] **Step 3: Update `diagram-export-page` to accept `choreography`**

In `examples/src/pages/diagram-export-page.ts`, change line 48:

```typescript
@state() private _diagramType: 'case' | 'swf' | 'choreography' = 'choreography';
```

Change line 55:

```typescript
if (type === 'case' || type === 'swf' || type === 'choreography') this._diagramType = type;
```

Change line 138 — update the active class check:

```typescript
<button class="type-toggle ${this._diagramType === 'choreography' || this._diagramType === 'case' ? 'active' : ''}"
```

Change line 154 — update the conditional render:

```typescript
${this._diagramType === 'choreography' || this._diagramType === 'case'
```

- [ ] **Step 4: Run full test suite**

Run: `yarn test`
Expected: ALL PASS

- [ ] **Step 5: Run typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git add components/casehub-diagram/src/casehub-diagram.ts components/diagram-workbench/src/diagram-workbench.ts examples/src/pages/diagram-export-page.ts
git commit -m "feat(#295): wire diagram type badge through casehub-diagram and update consumers

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-11-20-execution-model-badge-design.md] — design spec this plan implements
- `packages/mosaic-core/src/diagram-type.ts` — existing detection function
- `packages/mosaic-core/src/diagram-type.test.ts` — existing test file
- `packages/mosaic-core/src/diagram-tags.ts` — `DIAGRAM_TAGS` routing map
- `packages/mosaic-core/src/diagram-tags.test.ts` — existing tag test
- `packages/mosaic-core/src/index.ts` — package exports
- `components/casehub-diagram/src/casehub-diagram.ts:39` — existing `detectDiagramType` import
- `components/casehub-diagram/src/casehub-diagram-toolbar.ts` — toolbar component
- `components/casehub-diagram/src/casehub-diagram-toolbar.test.ts` — toolbar tests
- `components/diagram-workbench/src/diagram-workbench.ts:114` — hardcoded `'case'` root type
- `examples/src/pages/diagram-export-page.ts:48-55` — `'case'` type checks
- GitHub #295 — focal issue
