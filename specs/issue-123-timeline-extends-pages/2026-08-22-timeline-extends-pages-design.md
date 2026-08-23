# blocks-timeline refactoring — shared renderers via composition

**Issue:** #123
**Date:** 2026-08-22

## Summary

Refactor `blocks-timeline` to share rendering code with `PagesEventTimeline` in casehub-pages via composition — pure render functions imported by both components. BlocksTimeline keeps its DataSourceMixin-based data lifecycle; PagesEventTimeline keeps its PagesElement-based host-pushed lifecycle. Rendering duplication moves to pages-viz as standalone pure functions.

## Why composition, not inheritance

Issue #123 originally proposed extending `PagesTimeline` from pages. Analysis during design revealed two blockers:

1. **PagesElement render gate:** `PagesElement.render()` gates content display on `!!this.props && !this.controller.loading && !!this.controller.dataSet`. BlocksTimeline consumers set `endpoint` and `strategy` — they never set `props` or push `dataSet` via the host infrastructure. Inheriting PagesEventTimeline means overriding `render()` entirely (~125 lines of bridge/override code), yielding negative-value inheritance.

2. **Different data acquisition models:** PagesElement uses host-pushed data via `pages-data-request` events and `DataSourceController`. DataSourceMixin uses self-fetch via `endpoint` and `fetchSource`. These are correctly independent — they serve fundamentally different hosting contexts (dashboard-embedded vs standalone/panel-hosted).

Composition shares the code that IS generic (rendering) while keeping the code that ISN'T (data lifecycle) correctly independent.

## Architecture

Four layers, each depending only downward:

```
┌──────────────────────────────────────────────────┐
│ Layer 4: Component shells                         │
│ PagesEventTimeline (pages-viz, host-pushed data)  │
│ BlocksTimeline (blocks-ui, self-fetch + panels)   │
│ Each owns: data lifecycle, events, configure()    │
├──────────────────────────────────────────────────┤
│ Layer 3: Domain strategies (blocks-ui)            │
│ eventChronology, stateProgression,                │
│ commitmentLifecycle, orchestrationEvents           │
├──────────────────────────────────────────────────┤
│ Layer 2: Render functions (pages-viz)             │
│ renderVerticalTimeline + verticalTimelineStyles   │
│ renderHorizontalTimeline + horizontalTimelineStyles│
│ renderCompactTimeline + compactTimelineStyles     │
│ renderFilterBar (shared filter chip UI)           │
│ Pure: EventTimelineNode[] + callbacks → HTML      │
├──────────────────────────────────────────────────┤
│ Layer 1: Types (pages-component, already there)   │
│ EventTimelineNode, EventTimelineStrategy,         │
│ EventTimelineLayout, EventNodeStatus              │
└──────────────────────────────────────────────────┘
```

No cross-layer coupling:
- Strategies don't reference renderer CSS classes (inline styles per protocol PP-20260713-8ea1af)
- Renderers use status-based styling only — no domain categories (CASE, WORKER, TIMER, etc.)
- Renderers don't know about data sources
- Types are pure data

## Pages changes (PR 1 — casehub-pages)

### New renderer functions

Pure render functions in pages-viz, transferred from blocks-timeline's `renderers/` directory:

```
packages/pages-viz/src/components/event-timeline/
  renderers/
    vertical.ts        # renderVerticalTimeline + verticalTimelineStyles
    horizontal.ts      # renderHorizontalTimeline + horizontalTimelineStyles
    compact.ts         # renderCompactTimeline + compactTimelineStyles
    filter-bar.ts      # renderFilterBar + filterBarStyles
```

Each renderer is a pure function with an option bag:

```typescript
function renderVerticalTimeline(
  nodes: EventTimelineNode[],
  opts: VerticalTimelineOptions,
): TemplateResult;

interface VerticalTimelineOptions {
  expandedKeys: Set<string>;
  onNodeClick: (node: EventTimelineNode, index: number) => void;
  onToggleExpand: (key: string) => void;
  onKeyDown: (e: KeyboardEvent, index: number) => void;
  renderNode?: (node: EventTimelineNode) => unknown;
  renderDetail?: (node: EventTimelineNode) => unknown;
}

function renderHorizontalTimeline(
  nodes: EventTimelineNode[],
  opts: HorizontalTimelineOptions,
): TemplateResult;

interface HorizontalTimelineOptions {
  onNodeClick: (node: EventTimelineNode, index: number) => void;
  onKeyDown: (e: KeyboardEvent, index: number) => void;
  renderNode?: (node: EventTimelineNode) => unknown;
}

function renderCompactTimeline(
  nodes: EventTimelineNode[],
  opts: CompactTimelineOptions,
): TemplateResult;

interface CompactTimelineOptions {
  onExpandRequested: () => void;
  onKeyDown: (e: KeyboardEvent) => void;
}

function renderFilterBar(
  categories: string[],
  activeFilters: Set<string>,
  onToggle: (category: string) => void,
): TemplateResult;
```

Render callback return types are `unknown` (not `TemplateResult`) — framework-agnostic at the type level. Lit's `TemplateResult` is assignable to `unknown`, so consumers pass typed callbacks without issue.

Each renderer exports its CSS as a tagged template literal (`verticalTimelineStyles`, etc.) for component shells to include in their `static styles`.

### Domain-agnostic styling

Renderers use **status-based** styling only (`status-completed`, `status-active`, `status-pending`, `status-failed`, `status-skipped`) — matching `EventNodeStatus`. The current category-specific CSS (`.timeline-node.CASE .node-dot`, `.event-dot.WORKER`, `.event-type-badge.lifecycle`, etc.) is CaseHub domain knowledge that must not move to pages-viz.

Category-specific visual differentiation (colored dots per event stream type, badge styles per category) is achieved through the strategy's `renderNode` callback using inline styles per protocol PP-20260713-8ea1af. The renderers provide a visually neutral default based on node status; strategies that want richer category rendering pass a `renderNode` callback.

### Dependency changes

The vertical renderer's default `renderDetail` uses `renderPropertyTree` and `propertyTreeStyles` from `@casehubio/pages-ui-components` — a sibling package within pages. `verticalTimelineStyles` must include `${propertyTreeStyles}` to ensure default detail rendering is styled. This is a new import in pages-viz from pages-ui-components (sibling dependency, no layering violation).

### PagesEventTimeline refactored

PagesEventTimeline's `_renderVertical()` method and `_renderFilterBar()` method are replaced by calls to the shared render functions. It gains horizontal and compact layout support via its existing `props.layout` property:

```typescript
protected override renderContent(
  props: EventTimelineProps,
  dataset: TypedDataSet,
): TemplateResult {
  const strategy = this._resolveStrategy();
  // ... resolve nodes ...
  const layout = props.layout ?? strategy?.defaultLayout ?? "vertical";
  const filtered = this._filteredNodes;

  return html`
    <div class="timeline-container">
      ${strategy?.filterCategories
        ? renderFilterBar(strategy.filterCategories, activeFilters, onToggle)
        : nothing}
      ${layout === 'vertical'
        ? renderVerticalTimeline(filtered, { ... })
        : layout === 'horizontal'
        ? renderHorizontalTimeline(filtered, { ... })
        : renderCompactTimeline(filtered, { ... })}
    </div>
  `;
}
```

`_dataSetToNodes` is unchanged. Layout-specific data concerns (ordering for pipelines, timestamp format for temporal weighting, node count for compact truncation) are the strategy's responsibility via `transformData` and `toNodes`.

### Pages-viz exports

```typescript
// Existing
export { PagesEventTimeline } from "./components/PagesEventTimeline.js";
export type { EventTimelineNode, EventTimelineStrategy, EventNodeStatus } from "./components/event-timeline-types.js";

// New — render functions for external composition
export { renderVerticalTimeline, verticalTimelineStyles } from "./components/event-timeline/renderers/vertical.js";
export type { VerticalTimelineOptions } from "./components/event-timeline/renderers/vertical.js";
export { renderHorizontalTimeline, horizontalTimelineStyles } from "./components/event-timeline/renderers/horizontal.js";
export type { HorizontalTimelineOptions } from "./components/event-timeline/renderers/horizontal.js";
export { renderCompactTimeline, compactTimelineStyles } from "./components/event-timeline/renderers/compact.js";
export type { CompactTimelineOptions } from "./components/event-timeline/renderers/compact.js";
export { renderFilterBar, filterBarStyles } from "./components/event-timeline/renderers/filter-bar.js";
```

`computeTemporalWeights` is not exported — it is an internal implementation detail of the compact renderer. Tests verify temporal weighting through the compact renderer's output behavior.

## Blocks-ui changes (PR 2 — stacked on pages PR)

### BlocksTimeline refactored

- Drops local `renderers/` directory entirely
- Imports render functions + styles from `@casehubio/pages-viz`
- Keeps `DataSourceMixin(LiveRegionMixin(LitElement))` as base — no lifecycle change
- Keeps `configure()`, `createSourceFactory()`, pagination, event handlers
- `render()` calls imported render functions (including `renderFilterBar`) with callbacks
- `static styles` imports renderer styles from pages-viz

### Types refactored (`types.ts`)

Drops local `TimelineNode`, `NodeStatus`, `Layout`. Uses `EventTimelineNode`, `EventNodeStatus`, `EventTimelineLayout` from pages. Adds `BlocksTimelineStrategy` extending `EventTimelineStrategy` with pagination fields:

```typescript
import type { EventTimelineStrategy } from '@casehubio/pages-viz';

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

### Strategy updates

All four strategies updated to:
- Import `EventTimelineNode` instead of `TimelineNode`
- Use inline styles for `renderNode` callback output per protocol PP-20260713-8ea1af (fixes existing violation where `eventChronologyStrategy` produces `<span class="event-type-badge lifecycle">` referencing renderer CSS classes across the shadow boundary)
- Fix pre-existing bug: `orchestrationEventsStrategy` declares `supportsPagination: true` but has no `transformData` — add `transformData` that extracts `.content` from paginated responses, matching `eventChronologyStrategy`'s pattern

### Event topic alignment

BlocksTimeline aligns to the colon-delimited convention documented in ARC42STORIES.MD §4:
- `timeline.node-selected` → `timeline:node-selected`
- `timeline.expand-requested` → `timeline:expand-requested`

PagesEventTimeline emits `event-timeline:node-selected` — a different prefix matching its element name convention (`pages-event-timeline` → `event-timeline:*`). The divergence is intentional: each component's topic prefix derives from its element name minus the vendor prefix. Consumers bind to one component or the other, never both, so topic collisions are not a concern.

### No backward-compatibility re-exports

All consumers of the old type names (`TimelineNode`, `NodeStatus`, `Layout`) are in the same repo:
- `orchestration-workbench`
- Four strategies in `blocks-timeline/src/strategies/`
- Example showcase pages

All are updated in-band in PR 2. No temporary re-exports — they calcify into permanent debt when nothing forces removal.

### Consumer impact

- `orchestration-workbench`: import paths change, event topic strings change, type names change. All mechanical.
- Example showcase pages: same mechanical updates.
- External API unchanged: `endpoint`, `strategy`, `headers`, `configure()`, `layout`, `data` all work identically.

### Size impact

| Package | Before | After |
|---------|--------|-------|
| blocks-timeline total | ~1395 lines | ~700 lines |
| pages-viz event-timeline renderers | ~125 lines (vertical only) | ~585 lines (+filter bar) |
| pages-viz PagesEventTimeline | ~290 lines | ~200 lines (uses shared renderers) |
| **Net across both repos** | **~1810 lines** | **~1485 lines (-18%)** |

The primary win is deduplication: PagesEventTimeline's `_renderVertical` (~125 lines) and `_renderFilterBar` (~20 lines) are replaced by shared function calls. BlocksTimeline's three renderers (~585 lines) move to pages and are consumed by both components.

## Strategy binding

Two binding patterns coexist — they serve different contexts and are orthogonal to the shared rendering layer:
- **PagesEventTimeline**: strategy registry (`registerStrategy` / `strategyKey`) for YAML-driven dashboards
- **BlocksTimeline**: property injection (`@property strategy`) for programmatic composition

## Testing

### Pages-viz (PR 1)
- Unit tests for each renderer function: nodes + options → snapshot of HTML output
- Tests for temporal weighting through compact renderer output (closely-timed events produce visually closer dots)
- Tests for truncation logic (compact renderer's 7-node threshold)
- ARIA assertions: `role="list"` / `role="listitem"` for vertical + horizontal, `role="img"` for compact
- PagesEventTimeline integration: verify layout prop selects correct renderer
- Filter bar: verify chip rendering, toggle callback, ARIA `role="checkbox"` + `aria-checked`

### Blocks-ui (PR 2)
- Existing `blocks-timeline.test.ts` updated: same behavioral assertions, updated imports
- Existing strategy tests unchanged (strategies are the same, using `EventTimelineNode` type)
- Existing `renderers.test.ts` deleted — covered by pages-viz renderer tests
- New: verify event topics use colon-delimited format
- New: verify `BlocksTimelineStrategy` pagination fields work with imported renderers
- New: verify `orchestrationEventsStrategy.transformData` extracts `.content` from paginated responses
- Verify `orchestration-workbench` compiles and renders

## Migration checklist

1. **pages PR**: add renderer functions + styles + filter bar to pages-viz, refactor PagesEventTimeline, add tests
2. **pages SNAPSHOT**: `yarn build && mvn install` in casehub-pages to publish SNAPSHOT
3. **blocks-ui PR** (stacked): drop local renderers, import from pages-viz, update types/strategies/events, update all consumers in-band, add/update tests

## References

- `components/blocks-timeline/src/blocks-timeline.ts` — current implementation
- `components/blocks-timeline/src/renderers/` — renderer functions to move
- `components/blocks-timeline/src/types.ts` — types to align
- `pages-viz/src/components/PagesEventTimeline.ts` — pages base component
- `pages-viz/src/components/event-timeline-types.ts` — pages types (EventTimelineNode, EventTimelineStrategy)
- `pages-viz/src/base/PagesElement.ts` — PagesElement render gate (lines 93-98)
- `docs/specs/2026-07-13-blocks-timeline-design.md` — original blocks-timeline design spec
- `docs/protocols/blocks-ui/component-customisation-pattern.md` (PP-20260713-8ea1af) — inline styles requirement
- Decision review: R1-02 (PagesElement lifecycle incompatibility), R1-03 (composition superiority), R1-06 (pagination model)
- Spec review: R1-02 (category CSS domain coupling), R1-03 (propertyTreeStyles), R1-04 (drop re-exports), R1-06 (filter bar duplication)
- Issue #123 body — original scope and dependencies
