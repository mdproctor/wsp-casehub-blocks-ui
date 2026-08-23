# Timeline Shared Renderers Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #123 — refactor: blocks-timeline to extend PagesTimeline base from casehub-pages
**Issue group:** #123

**Goal:** Extract timeline render functions (vertical, horizontal, compact, filter bar) from blocks-ui to pages-viz as shared pure functions, then refactor both PagesEventTimeline and BlocksTimeline to consume them.

**Architecture:** Four-layer composition. Layer 1: types in pages-component (already exist). Layer 2: pure render functions in pages-viz (new). Layer 3: domain strategies in blocks-ui (updated). Layer 4: component shells in pages-viz and blocks-ui (refactored). No inheritance — both components import render functions and call them with their own callbacks.

**Tech Stack:** TypeScript, Lit 3.x, Vitest, pages-viz (pages-component sibling), blocks-ui (DataSourceMixin consumer)

## Global Constraints

- Renderers use status-based CSS only (`status-completed`, `status-active`, `status-pending`, `status-failed`, `status-skipped`) — no CaseHub domain categories
- Strategy `renderNode` callbacks use inline styles per protocol PP-20260713-8ea1af
- Render callback return types are `unknown` (not `TemplateResult`) for framework-agnostic type safety
- `computeTemporalWeights` is internal to the compact renderer — not exported
- No backward-compatibility re-exports — all consumers updated in-band
- Event topics use colon-delimited convention per ARC42STORIES.MD §4

---

## Batch 1: Shared render functions in pages-viz

### Task 1: Vertical timeline renderer

**Files:**
- Create: `packages/pages-viz/src/components/event-timeline/renderers/vertical.ts`
- Create: `packages/pages-viz/src/components/event-timeline/renderers/vertical.test.ts`

**Interfaces:**
- Consumes: `EventTimelineNode`, `EventNodeStatus` from `../event-timeline-types.js`
- Consumes: `renderPropertyTree`, `propertyTreeStyles` from `@casehubio/pages-ui-components`
- Produces: `renderVerticalTimeline(nodes: EventTimelineNode[], opts: VerticalTimelineOptions): TemplateResult`
- Produces: `verticalTimelineStyles: CSSResult` (includes `propertyTreeStyles`)
- Produces: `VerticalTimelineOptions` type

- [ ] **Step 1: Write the failing test**

Create `packages/pages-viz/src/components/event-timeline/renderers/vertical.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { render } from 'lit';
import type { EventTimelineNode } from '../../event-timeline-types.js';
import { renderVerticalTimeline, type VerticalTimelineOptions } from './vertical.js';

const makeNodes = (count: number, overrides?: Partial<EventTimelineNode>): EventTimelineNode[] =>
  Array.from({ length: count }, (_, i) => ({
    key: `node-${i}`,
    label: `Node ${i}`,
    status: 'completed' as const,
    timestamp: `2026-01-01T${String(10 + i).padStart(2, '0')}:00:00Z`,
    ...overrides,
  }));

function renderToContainer(template: unknown): HTMLDivElement {
  const container = document.createElement('div');
  render(template, container);
  return container;
}

describe('renderVerticalTimeline', () => {
  const defaultOpts: VerticalTimelineOptions = {
    expandedKeys: new Set(),
    onNodeClick: () => {},
    onToggleExpand: () => {},
    onKeyDown: () => {},
  };

  it('renders a list with role="list"', () => {
    const nodes = makeNodes(3);
    const container = renderToContainer(renderVerticalTimeline(nodes, defaultOpts));
    const list = container.querySelector('[role="list"]');
    expect(list).toBeTruthy();
    expect(list?.getAttribute('aria-label')).toBe('Timeline');
  });

  it('renders each node as role="listitem"', () => {
    const nodes = makeNodes(3);
    const container = renderToContainer(renderVerticalTimeline(nodes, defaultOpts));
    const items = container.querySelectorAll('[role="listitem"]');
    expect(items.length).toBe(3);
  });

  it('uses status-based CSS classes, not category classes', () => {
    const nodes = makeNodes(1, { status: 'active' });
    const container = renderToContainer(renderVerticalTimeline(nodes, defaultOpts));
    const node = container.querySelector('.timeline-node');
    expect(node?.classList.contains('status-active')).toBe(true);
    expect(node?.classList.contains('lifecycle')).toBe(false);
    expect(node?.classList.contains('CASE')).toBe(false);
  });

  it('shows expand button when node has detail', () => {
    const nodes = makeNodes(1, { detail: { foo: 'bar' } });
    const container = renderToContainer(renderVerticalTimeline(nodes, defaultOpts));
    expect(container.querySelector('.expand-button')).toBeTruthy();
  });

  it('shows detail content when key is in expandedKeys', () => {
    const nodes = makeNodes(1, { detail: { foo: 'bar' } });
    const opts = { ...defaultOpts, expandedKeys: new Set(['node-0']) };
    const container = renderToContainer(renderVerticalTimeline(nodes, opts));
    expect(container.querySelector('.payload-detail')).toBeTruthy();
  });

  it('calls renderNode callback when provided', () => {
    const nodes = makeNodes(1);
    let called = false;
    const opts = { ...defaultOpts, renderNode: () => { called = true; return 'custom'; } };
    renderToContainer(renderVerticalTimeline(nodes, opts));
    expect(called).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run packages/pages-viz/src/components/event-timeline/renderers/vertical.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement the vertical renderer**

Create `packages/pages-viz/src/components/event-timeline/renderers/vertical.ts`.

Transfer from blocks-timeline's `renderers/vertical.ts` with these changes:
- Import `EventTimelineNode` from `../../event-timeline-types.js` instead of `TimelineNode`
- Replace all category-specific CSS (`.timeline-node.lifecycle .node-dot`, `.timeline-node.CASE .node-dot`, `.event-type-badge.*`, etc.) with status-based classes: `.timeline-node.status-completed .node-dot`, `.timeline-node.status-active .node-dot`, etc.
- Node div class: `status-${node.status}` instead of `${node.category ?? 'lifecycle'}`
- Default `renderNode` returns `html`<span class="node-label">${node.label}</span>`` (no badge classes)
- Render callback types return `unknown` instead of `TemplateResult`
- Import `renderPropertyTree`, `propertyTreeStyles` from `@casehubio/pages-ui-components`
- Include `${propertyTreeStyles}` in exported `verticalTimelineStyles`
- Rename exports: `renderVertical` → `renderVerticalTimeline`, `verticalStyles` → `verticalTimelineStyles`, `VerticalOptions` → `VerticalTimelineOptions`

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run packages/pages-viz/src/components/event-timeline/renderers/vertical.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-viz/src/components/event-timeline/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-viz): add shared vertical timeline renderer Refs casehubio/blocks-ui#123"
```

### Task 2: Horizontal timeline renderer

**Files:**
- Create: `packages/pages-viz/src/components/event-timeline/renderers/horizontal.ts`
- Create: `packages/pages-viz/src/components/event-timeline/renderers/horizontal.test.ts`

**Interfaces:**
- Consumes: `EventTimelineNode` from `../../event-timeline-types.js`
- Produces: `renderHorizontalTimeline(nodes: EventTimelineNode[], opts: HorizontalTimelineOptions): TemplateResult`
- Produces: `horizontalTimelineStyles: CSSResult`
- Produces: `HorizontalTimelineOptions` type

- [ ] **Step 1: Write the failing test**

Create `packages/pages-viz/src/components/event-timeline/renderers/horizontal.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { render } from 'lit';
import type { EventTimelineNode } from '../../event-timeline-types.js';
import { renderHorizontalTimeline, type HorizontalTimelineOptions } from './horizontal.js';

const makeNodes = (count: number, overrides?: Partial<EventTimelineNode>): EventTimelineNode[] =>
  Array.from({ length: count }, (_, i) => ({
    key: `stage-${i}`,
    label: `Stage ${i}`,
    status: 'completed' as const,
    ...overrides,
  }));

function renderToContainer(template: unknown): HTMLDivElement {
  const container = document.createElement('div');
  render(template, container);
  return container;
}

describe('renderHorizontalTimeline', () => {
  const defaultOpts: HorizontalTimelineOptions = {
    onNodeClick: () => {},
    onKeyDown: () => {},
  };

  it('renders a list with aria-orientation="horizontal"', () => {
    const nodes = makeNodes(3);
    const container = renderToContainer(renderHorizontalTimeline(nodes, defaultOpts));
    const list = container.querySelector('[role="list"]');
    expect(list).toBeTruthy();
    expect(list?.getAttribute('aria-orientation')).toBe('horizontal');
  });

  it('renders connectors between nodes', () => {
    const nodes = makeNodes(3);
    const container = renderToContainer(renderHorizontalTimeline(nodes, defaultOpts));
    const connectors = container.querySelectorAll('.connector');
    expect(connectors.length).toBe(2);
  });

  it('uses status-based CSS classes on stage nodes', () => {
    const nodes = makeNodes(1, { status: 'active' });
    const container = renderToContainer(renderHorizontalTimeline(nodes, defaultOpts));
    const stageNode = container.querySelector('.stage-node');
    expect(stageNode?.classList.contains('stage-node--active')).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run packages/pages-viz/src/components/event-timeline/renderers/horizontal.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement the horizontal renderer**

Create `packages/pages-viz/src/components/event-timeline/renderers/horizontal.ts`.

Transfer from blocks-timeline's `renderers/horizontal.ts` with these changes:
- Import `EventTimelineNode` instead of `TimelineNode`
- Rename: `renderHorizontal` → `renderHorizontalTimeline`, `horizontalStyles` → `horizontalTimelineStyles`, `HorizontalOptions` → `HorizontalTimelineOptions`
- Render callback return types: `unknown` instead of `TemplateResult`
- No category-specific CSS exists in horizontal renderer (already status-based) — no changes needed

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run packages/pages-viz/src/components/event-timeline/renderers/horizontal.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-viz/src/components/event-timeline/renderers/horizontal.*
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-viz): add shared horizontal timeline renderer Refs casehubio/blocks-ui#123"
```

### Task 3: Compact timeline renderer + filter bar

**Files:**
- Create: `packages/pages-viz/src/components/event-timeline/renderers/compact.ts`
- Create: `packages/pages-viz/src/components/event-timeline/renderers/compact.test.ts`
- Create: `packages/pages-viz/src/components/event-timeline/renderers/filter-bar.ts`
- Create: `packages/pages-viz/src/components/event-timeline/renderers/filter-bar.test.ts`

**Interfaces:**
- Consumes: `EventTimelineNode` from `../../event-timeline-types.js`
- Produces: `renderCompactTimeline(nodes: EventTimelineNode[], opts: CompactTimelineOptions): TemplateResult`
- Produces: `compactTimelineStyles: CSSResult`
- Produces: `CompactTimelineOptions` type
- Produces: `renderFilterBar(categories: string[], activeFilters: Set<string>, onToggle: (cat: string) => void): TemplateResult`
- Produces: `filterBarStyles: CSSResult`

- [ ] **Step 1: Write the failing tests**

Create `compact.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { render } from 'lit';
import type { EventTimelineNode } from '../../event-timeline-types.js';
import { renderCompactTimeline, type CompactTimelineOptions } from './compact.js';

const makeNodes = (count: number, overrides?: Partial<EventTimelineNode>): EventTimelineNode[] =>
  Array.from({ length: count }, (_, i) => ({
    key: `node-${i}`,
    label: `Node ${i}`,
    status: 'completed' as const,
    timestamp: `2026-01-01T${String(10 + i).padStart(2, '0')}:00:00Z`,
    ...overrides,
  }));

function renderToContainer(template: unknown): HTMLDivElement {
  const container = document.createElement('div');
  render(template, container);
  return container;
}

describe('renderCompactTimeline', () => {
  const defaultOpts: CompactTimelineOptions = {
    onExpandRequested: () => {},
    onKeyDown: () => {},
  };

  it('renders with role="img" and summary aria-label', () => {
    const nodes = makeNodes(3);
    const container = renderToContainer(renderCompactTimeline(nodes, defaultOpts));
    const strip = container.querySelector('[role="img"]');
    expect(strip).toBeTruthy();
    expect(strip?.getAttribute('aria-label')).toContain('3 events');
  });

  it('truncates at 7+ nodes showing first 3 + last 2 with ellipsis', () => {
    const nodes = makeNodes(10);
    const container = renderToContainer(renderCompactTimeline(nodes, defaultOpts));
    const dots = container.querySelectorAll('.event-dot');
    expect(dots.length).toBe(5);
    const ellipsis = container.querySelector('.ellipsis');
    expect(ellipsis?.textContent).toContain('+5');
  });

  it('uses status-based CSS classes on dots', () => {
    const nodes = makeNodes(1, { status: 'failed' });
    const container = renderToContainer(renderCompactTimeline(nodes, defaultOpts));
    const dot = container.querySelector('.event-dot');
    expect(dot?.classList.contains('status-failed')).toBe(true);
    expect(dot?.classList.contains('lifecycle')).toBe(false);
  });
});
```

Create `filter-bar.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render } from 'lit';
import { renderFilterBar } from './filter-bar.js';

function renderToContainer(template: unknown): HTMLDivElement {
  const container = document.createElement('div');
  render(template, container);
  return container;
}

describe('renderFilterBar', () => {
  it('renders filter chips with role="checkbox"', () => {
    const categories = ['CASE', 'WORKER', 'TIMER'];
    const active = new Set(categories);
    const container = renderToContainer(renderFilterBar(categories, active, () => {}));
    const chips = container.querySelectorAll('[role="checkbox"]');
    expect(chips.length).toBe(3);
  });

  it('marks active filters as aria-checked="true"', () => {
    const active = new Set(['CASE']);
    const container = renderToContainer(renderFilterBar(['CASE', 'WORKER'], active, () => {}));
    const chips = container.querySelectorAll('[role="checkbox"]');
    expect(chips[0]?.getAttribute('aria-checked')).toBe('true');
    expect(chips[1]?.getAttribute('aria-checked')).toBe('false');
  });

  it('calls onToggle with the category when clicked', () => {
    const onToggle = vi.fn();
    const container = renderToContainer(renderFilterBar(['CASE'], new Set(['CASE']), onToggle));
    const chip = container.querySelector('[role="checkbox"]') as HTMLElement;
    chip.click();
    expect(onToggle).toHaveBeenCalledWith('CASE');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run packages/pages-viz/src/components/event-timeline/renderers/compact.test.ts packages/pages-viz/src/components/event-timeline/renderers/filter-bar.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement compact renderer and filter bar**

Create `compact.ts` — transfer from blocks-timeline's `renderers/compact.ts` with:
- Import `EventTimelineNode` instead of `TimelineNode`
- Replace category CSS (`.event-dot.lifecycle`, `.event-dot.CASE`, `.event-dot.WORKER`, etc.) with status-based: `.event-dot.status-completed`, `.event-dot.status-active`, `.event-dot.status-pending`, `.event-dot.status-failed`, `.event-dot.status-skipped`
- Dot div class: `event-dot status-${node.status}` instead of `event-dot ${node.category ?? 'lifecycle'}`
- Rename: `renderCompact` → `renderCompactTimeline`, `compactStyles` → `compactTimelineStyles`, `CompactOptions` → `CompactTimelineOptions`
- `computeTemporalWeights` stays internal (not exported from the module)

Create `filter-bar.ts`:

```typescript
import { html, css } from 'lit';
import type { TemplateResult } from 'lit';

export function renderFilterBar(
  categories: string[],
  activeFilters: Set<string>,
  onToggle: (category: string) => void,
): TemplateResult {
  return html`
    <div class="filter-bar" role="group" aria-label="Filter">
      ${categories.map(cat => html`
        <button
          class="filter-chip"
          role="checkbox"
          aria-checked="${activeFilters.has(cat)}"
          @click=${() => onToggle(cat)}
        >${cat}</button>
      `)}
    </div>
  `;
}

export const filterBarStyles = css`
  .filter-bar {
    display: flex;
    gap: 8px;
    margin-bottom: 24px;
    flex-wrap: wrap;
  }
  .filter-chip {
    padding: 6px 12px;
    border-radius: 16px;
    border: 1px solid var(--pages-neutral-6, #d1d5db);
    background: var(--pages-neutral-1, #fff);
    cursor: pointer;
    font-size: 13px;
    font-weight: 500;
    transition: all 0.2s;
  }
  .filter-chip[aria-checked="true"] {
    background: var(--pages-accent-9, #2563eb);
    color: white;
    border-color: var(--pages-accent-9, #2563eb);
  }
  .filter-chip:hover { border-color: var(--pages-accent-7, #3b82f6); }
`;
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run packages/pages-viz/src/components/event-timeline/renderers/compact.test.ts packages/pages-viz/src/components/event-timeline/renderers/filter-bar.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-viz/src/components/event-timeline/renderers/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-viz): add shared compact renderer and filter bar Refs casehubio/blocks-ui#123"
```

### Task 4: Refactor PagesEventTimeline + export from pages-viz index

**Files:**
- Modify: `packages/pages-viz/src/components/PagesEventTimeline.ts`
- Modify: `packages/pages-viz/src/components/PagesEventTimeline.test.ts`
- Modify: `packages/pages-viz/src/index.ts`

**Interfaces:**
- Consumes: all four render functions from `./event-timeline/renderers/`
- Produces: updated PagesEventTimeline with layout support (vertical, horizontal, compact)
- Produces: pages-viz index exports for render functions + option bag types

- [ ] **Step 1: Write failing tests for horizontal/compact layout**

Add to `PagesEventTimeline.test.ts`:

```typescript
it('renders horizontal layout when props.layout is "horizontal"', async () => {
  // Set props with layout: 'horizontal', provide data with test nodes
  // Assert: container has .pipeline element with role="list" aria-orientation="horizontal"
});

it('renders compact layout when props.layout is "compact"', async () => {
  // Set props with layout: 'compact', provide data
  // Assert: container has .compact-strip element with role="img"
});

it('renders filter bar when strategy has filterCategories', async () => {
  // Provide a strategy with filterCategories: ['A', 'B']
  // Assert: filter chips rendered with role="checkbox"
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run packages/pages-viz/src/components/PagesEventTimeline.test.ts`
Expected: FAIL — only vertical rendering exists

- [ ] **Step 3: Refactor PagesEventTimeline**

Replace `_renderVertical()` and `_renderFilterBar()` with calls to shared render functions. Add layout selection in `renderContent()`:

```typescript
import { renderVerticalTimeline, verticalTimelineStyles } from './event-timeline/renderers/vertical.js';
import { renderHorizontalTimeline, horizontalTimelineStyles } from './event-timeline/renderers/horizontal.js';
import { renderCompactTimeline, compactTimelineStyles } from './event-timeline/renderers/compact.js';
import { renderFilterBar, filterBarStyles } from './event-timeline/renderers/filter-bar.js';
```

In `renderContent()`:
```typescript
const layout = props.layout ?? strategy?.defaultLayout ?? "vertical";
```

Then dispatch to the appropriate renderer. Remove the inline `_renderVertical` and `_renderFilterBar` methods. Update `static styles` to import from shared style exports.

- [ ] **Step 4: Update pages-viz index.ts**

Add exports for render functions and option bag types:

```typescript
export { renderVerticalTimeline, verticalTimelineStyles } from "./components/event-timeline/renderers/vertical.js";
export type { VerticalTimelineOptions } from "./components/event-timeline/renderers/vertical.js";
export { renderHorizontalTimeline, horizontalTimelineStyles } from "./components/event-timeline/renderers/horizontal.js";
export type { HorizontalTimelineOptions } from "./components/event-timeline/renderers/horizontal.js";
export { renderCompactTimeline, compactTimelineStyles } from "./components/event-timeline/renderers/compact.js";
export type { CompactTimelineOptions } from "./components/event-timeline/renderers/compact.js";
export { renderFilterBar, filterBarStyles } from "./components/event-timeline/renderers/filter-bar.js";
export type { EventTimelineLayout } from "@casehubio/pages-component";
```

Note: `EventTimelineLayout` is defined in `@casehubio/pages-component` (not pages-viz). Re-exporting it from pages-viz ensures blocks-ui has a single import source for all event-timeline types.

- [ ] **Step 5: Run all PagesEventTimeline tests**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run packages/pages-viz/src/components/PagesEventTimeline.test.ts`
Expected: PASS

- [ ] **Step 6: Run full pages-viz test suite**

Run: `cd /Users/mdproctor/claude/casehub/pages && npx vitest run packages/pages-viz/`
Expected: PASS

- [ ] **Step 7: Build and publish SNAPSHOT**

Run: `cd /Users/mdproctor/claude/casehub/pages && yarn build && mvn install`
Expected: BUILD SUCCESS — SNAPSHOT published to ~/.m2

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-viz/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-viz): refactor PagesEventTimeline to use shared renderers, add layout support Refs casehubio/blocks-ui#123"
```

---

## Batch 2: Refactor blocks-timeline types + strategies

### Task 5: Refactor types.ts — adopt pages types

**Files:**
- Modify: `components/blocks-timeline/src/types.ts`
- Modify: `components/blocks-timeline/src/types.test.ts`
- Modify: `components/blocks-timeline/package.json`

**Interfaces:**
- Consumes: `EventTimelineNode`, `EventTimelineStrategy`, `EventNodeStatus`, `EventTimelineLayout` from `@casehubio/pages-viz`
- Produces: `BlocksTimelineStrategy<T>` extending `EventTimelineStrategy<T>` with pagination fields
- Produces: `PaginationMeta`, `StageConfig` (unchanged)

- [ ] **Step 1: Add pages-viz dependency to package.json**

Add `"@casehubio/pages-viz": "*"` to `dependencies` in `components/blocks-timeline/package.json`.

- [ ] **Step 2: Update types.ts**

Replace `types.ts` contents:

```typescript
import type { EventTimelineStrategy } from '@casehubio/pages-viz';

export type { EventTimelineNode, EventNodeStatus, EventTimelineLayout } from '@casehubio/pages-viz';

export interface BlocksTimelineStrategy<T = unknown> extends EventTimelineStrategy<T> {
  supportsPagination?: boolean;
  extractPaginationMeta?: (raw: unknown) => PaginationMeta | undefined;
}

export interface PaginationMeta {
  page: number;
  totalPages: number;
  totalElements: number;
}

export interface StageConfig {
  key: string;
  label: string;
  icon?: string;
  terminal?: 'success' | 'failure' | 'transfer';
}
```

- [ ] **Step 3: Update types.test.ts**

Update imports to use `EventTimelineNode` instead of `TimelineNode`, `EventNodeStatus` instead of `NodeStatus`, etc.

- [ ] **Step 4: Run types tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && npx vitest run components/blocks-timeline/src/types.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/blocks-timeline/src/types.ts components/blocks-timeline/src/types.test.ts components/blocks-timeline/package.json
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor: adopt pages types, add BlocksTimelineStrategy Refs #123"
```

### Task 6: Update strategies — pages types + inline styles + orchestration pagination fix

**Files:**
- Modify: `components/blocks-timeline/src/strategies/event-chronology.ts`
- Modify: `components/blocks-timeline/src/strategies/event-chronology.test.ts`
- Modify: `components/blocks-timeline/src/strategies/state-progression.ts`
- Modify: `components/blocks-timeline/src/strategies/state-progression.test.ts`
- Modify: `components/blocks-timeline/src/strategies/commitment-lifecycle.ts`
- Modify: `components/blocks-timeline/src/strategies/commitment-lifecycle.test.ts`
- Modify: `components/blocks-timeline/src/strategies/orchestration-events.ts`
- Modify: `components/blocks-timeline/src/strategies/orchestration-events.test.ts`

**Interfaces:**
- Consumes: `EventTimelineNode`, `EventNodeStatus` from `../types.js` (re-exported from pages)
- Consumes: `BlocksTimelineStrategy` from `../types.js`
- Produces: updated strategies using `EventTimelineNode`, inline-style render callbacks

- [ ] **Step 1: Write failing test for eventChronologyStrategy inline styles**

Add to `event-chronology.test.ts`:

```typescript
it('renderNode uses inline styles, not CSS classes', () => {
  const strategy = eventChronologyStrategy();
  const node = strategy.toNodes(events)[0]!;
  const result = strategy.renderNode!(node);
  // renderNode should return HTML with style= attribute, not class="event-type-badge"
  const container = document.createElement('div');
  render(result, container);
  const span = container.querySelector('span');
  expect(span?.getAttribute('style')).toBeTruthy();
  expect(span?.classList.contains('event-type-badge')).toBe(false);
});
```

- [ ] **Step 2: Write failing test for orchestrationEventsStrategy transformData**

Add to `orchestration-events.test.ts`:

```typescript
it('has transformData that extracts .content from paginated responses', () => {
  const strategy = orchestrationEventsStrategy;
  expect(strategy.transformData).toBeDefined();
  const pagedResponse = {
    content: [{ id: 'e1', eventType: 'EXECUTION_STARTED', timestamp: '2026-01-01T10:00:00Z', payload: { type: 'EXECUTION_STARTED', model: { pattern: 'SCATTER_GATHER', failurePolicy: { routingFailureAction: 'FAIL', aggregationFailureAction: 'FAIL' } } } }],
    page: 0,
    totalPages: 1,
    totalElements: 1,
    size: 20,
  };
  const result = strategy.transformData!(pagedResponse);
  expect(Array.isArray(result)).toBe(true);
  expect(result).toHaveLength(1);
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && npx vitest run components/blocks-timeline/src/strategies/`
Expected: FAIL on the two new tests

- [ ] **Step 4: Update all four strategies**

**event-chronology.ts:**
- Change `import type { TimelineNode, TimelineStrategy, PaginationMeta }` → `import type { EventTimelineNode, BlocksTimelineStrategy, PaginationMeta }`
- Change `import { renderPropertyTree }` → keep as-is (blocks-ui-core re-export still works; will be cleaned up in #122)
- Update `renderNode` to use inline styles instead of `class="event-type-badge ${eventCategory}"`:

```typescript
const CATEGORY_STYLES: Record<string, string> = {
  lifecycle: 'background: var(--pages-success-3, #dcfce7); color: var(--pages-success-11, #166534);',
  task: 'background: var(--pages-warning-3, #fef3c7); color: var(--pages-warning-11, #92400e);',
  agent: 'background: var(--pages-purple-3, #f3e8ff); color: var(--pages-purple-11, #581c87);',
  milestone: 'background: var(--pages-accent-3, #dbeafe); color: var(--pages-accent-11, #1e3a5f);',
  'action-gate': 'background: var(--pages-error-3, #fee2e2); color: var(--pages-error-11, #991b1b);',
  orchestration: 'background: var(--pages-neutral-3, #f3f4f6); color: var(--pages-neutral-11, #374151);',
  timer: 'background: var(--pages-warning-3, #fef3c7); color: var(--pages-warning-11, #92400e);',
};

renderNode(node: EventTimelineNode) {
  const event = node.detail as CaseEvent | undefined;
  const eventCategory = event ? cat(event.eventType) : 'lifecycle';
  const badgeStyle = CATEGORY_STYLES[eventCategory] ?? '';
  return html`<span style="display:inline-block; padding:4px 8px; border-radius:4px; font-size:12px; font-weight:600; text-transform:uppercase; letter-spacing:0.5px; ${badgeStyle}">${node.label}</span>`;
},
```

- Update return type annotation: `TimelineStrategy<CaseEvent[]>` → `BlocksTimelineStrategy<CaseEvent[]>`
- Update `toNodes` return type: `TimelineNode[]` → `EventTimelineNode[]`

**state-progression.ts:**
- Change imports: `TimelineNode` → `EventTimelineNode`, `NodeStatus` → `EventNodeStatus`, `TimelineStrategy` → use `EventTimelineStrategy` from pages-viz
- Update function signatures and return types

**commitment-lifecycle.ts:**
- Change imports: `TimelineStrategy` → `EventTimelineStrategy`, `StageConfig` stays local
- Update return type

**orchestration-events.ts:**
- Change imports: `TimelineNode` → `EventTimelineNode`, `TimelineStrategy` → `BlocksTimelineStrategy`, `NodeStatus` → `EventNodeStatus`
- Add `transformData` to handle paginated responses:

```typescript
transformData(raw: unknown): OrchestrationAuditEvent[] {
  if (raw != null && typeof raw === 'object' && 'content' in raw && Array.isArray((raw as { content: unknown[] }).content)) {
    return (raw as { content: OrchestrationAuditEvent[] }).content;
  }
  return raw as OrchestrationAuditEvent[];
},
```

- [ ] **Step 5: Run all strategy tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && npx vitest run components/blocks-timeline/src/strategies/`
Expected: PASS

- [ ] **Step 6: Commit orchestration bug fix separately**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/blocks-timeline/src/strategies/orchestration-events.ts components/blocks-timeline/src/strategies/orchestration-events.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "fix: add transformData to orchestrationEventsStrategy for paginated responses Refs #123"
```

- [ ] **Step 7: Commit refactoring changes**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/blocks-timeline/src/strategies/ components/blocks-timeline/src/types.*
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor: update strategies to pages types + inline styles Refs #123"
```

---

## Batch 3: Refactor BlocksTimeline component + consumers

### Task 7: Refactor BlocksTimeline to use shared renderers

**Files:**
- Modify: `components/blocks-timeline/src/blocks-timeline.ts`
- Delete: `components/blocks-timeline/src/renderers/vertical.ts`
- Delete: `components/blocks-timeline/src/renderers/horizontal.ts`
- Delete: `components/blocks-timeline/src/renderers/compact.ts`
- Delete: `components/blocks-timeline/src/renderers/renderers.test.ts`
- Modify: `components/blocks-timeline/src/blocks-timeline.test.ts`

**Interfaces:**
- Consumes: `renderVerticalTimeline`, `renderHorizontalTimeline`, `renderCompactTimeline`, `renderFilterBar` + styles from `@casehubio/pages-viz`
- Consumes: `EventTimelineNode`, `EventTimelineLayout` from `./types.js`
- Consumes: `BlocksTimelineStrategy` from `./types.js`
- Produces: `BlocksTimeline` component with same external API

- [ ] **Step 1: Update blocks-timeline.test.ts imports**

Replace `TimelineNode` → `EventTimelineNode`, `TimelineStrategy` → `BlocksTimelineStrategy` in test imports. Replace event topic assertions from `timeline.node-selected` to `timeline:node:selected`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && npx vitest run components/blocks-timeline/src/blocks-timeline.test.ts`
Expected: FAIL — old types and event topics

- [ ] **Step 3: Refactor blocks-timeline.ts**

Replace renderer imports:
```typescript
// Remove these
import { renderVertical, verticalStyles } from './renderers/vertical.js';
import { renderHorizontal, horizontalStyles } from './renderers/horizontal.js';
import { renderCompact, compactStyles } from './renderers/compact.js';

// Add these
import {
  renderVerticalTimeline, verticalTimelineStyles,
  renderHorizontalTimeline, horizontalTimelineStyles,
  renderCompactTimeline, compactTimelineStyles,
  renderFilterBar, filterBarStyles,
} from '@casehubio/pages-viz';
```

Update type imports:
```typescript
import type { EventTimelineNode, EventTimelineLayout, BlocksTimelineStrategy, PaginationMeta } from './types.js';
```

Update `strategy` property type: `TimelineStrategy` → `BlocksTimelineStrategy`

Update render callbacks in `render()` to use renamed functions (`renderVerticalTimeline`, etc.).

Replace `_renderFilterBar()` method with call to imported `renderFilterBar()`.

Update event topics:
```typescript
emitPagesEvent(this, 'timeline:node:selected', { node, index });
emitPagesEvent(this, 'timeline:expand:requested', {});
```

Update `static styles`:
```typescript
static override styles = css`
  :host { ... }
  .timeline-container { ... }
  .error-message { ... }
  .pagination-footer { ... }
  .pagination-progress { ... }
  .load-more-button { ... }
  ${verticalTimelineStyles}
  ${horizontalTimelineStyles}
  ${compactTimelineStyles}
  ${filterBarStyles}
`;
```

- [ ] **Step 4: Delete local renderer files**

Use `ide_refactor_safe_delete` for:
- `components/blocks-timeline/src/renderers/vertical.ts`
- `components/blocks-timeline/src/renderers/horizontal.ts`
- `components/blocks-timeline/src/renderers/compact.ts`
- `components/blocks-timeline/src/renderers/renderers.test.ts`

- [ ] **Step 5: Run tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && npx vitest run components/blocks-timeline/`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/blocks-timeline/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor: BlocksTimeline uses shared renderers from pages-viz Refs #123"
```

### Task 8: Update index.ts exports + consumers

**Files:**
- Modify: `components/blocks-timeline/src/index.ts`
- Modify: `components/orchestration-workbench/src/orchestration-workbench.ts`
- Modify: `examples/src/pages/timeline-events-page.ts`
- Modify: `examples/src/pages/timeline-commitment-page.ts`
- Modify: `examples/src/pages/commitment-lifecycle-page.ts`
- Modify: `examples/src/pages/timeline-custom-page.ts`

**Interfaces:**
- Consumes: all updated types and strategies from blocks-timeline
- Produces: clean public API without backward-compat aliases

- [ ] **Step 1: Update index.ts**

```typescript
export { BlocksTimeline } from './blocks-timeline.js';
export type { EventTimelineNode, EventNodeStatus, EventTimelineLayout, BlocksTimelineStrategy, StageConfig, PaginationMeta } from './types.js';
export { stateProgressionStrategy, linearResolveStatus, QHORUS_STAGES } from './strategies/state-progression.js';
export {
  eventChronologyStrategy,
  categorizeEvent,
  isCompactModeEvent,
  type CaseEvent,
  type CaseHubEventType,
  type EventStreamType,
  type NodeCategory,
  type EventLogEntryResponse,
  type PagedResponse,
} from './strategies/event-chronology.js';
export {
  commitmentLifecycleStrategy,
  type CommitmentLifecycleData,
} from './strategies/commitment-lifecycle.js';
export {
  orchestrationEventsStrategy,
  orchestrationFilterCategory,
} from './strategies/orchestration-events.js';
```

- [ ] **Step 2: Update orchestration-workbench**

In `orchestration-workbench.ts`, no import changes needed — it imports `orchestrationEventsStrategy` from the package, which still works. Check if it references `TimelineNode` anywhere and update if so. Update event listener topic strings from `timeline.node-selected` to `timeline:node:selected` if present.

- [ ] **Step 3: Update example pages**

In each example page, update:
- `TimelineNode` → `EventTimelineNode`
- `TimelineStrategy` → `BlocksTimelineStrategy`
- `Layout` → `EventTimelineLayout`
- `NodeStatus` → `EventNodeStatus`

These are mechanical find-and-replace operations within each file.

- [ ] **Step 4: Run full build + typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`
Expected: no errors

- [ ] **Step 5: Run full test suite**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn test`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/blocks-timeline/src/index.ts components/orchestration-workbench/ examples/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor: update exports and consumers to pages types Refs #123"
```

---

## References

- [2026-08-22-timeline-extends-pages-design.md] — design spec this plan implements
- [components/blocks-timeline/src/blocks-timeline.ts] — current BlocksTimeline implementation
- [components/blocks-timeline/src/renderers/] — renderer functions to move
- [components/blocks-timeline/src/types.ts] — types to align
- [pages-viz/src/components/PagesEventTimeline.ts] — pages base component
- [pages-viz/src/components/event-timeline-types.ts] — pages types
- [docs/protocols/blocks-ui/component-customisation-pattern.md] — PP-20260713-8ea1af inline styles protocol
- [GitHub #123] — focal issue
