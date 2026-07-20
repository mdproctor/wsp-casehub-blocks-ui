# Composable Case Explorer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #87 — composable case explorer
**Issue group:** #87

**Goal:** Build a registration-based entity browser for casehub — cases, workers, gates, channels, definitions — with composable components, dynamic commands, and domain customisation.

**Architecture:** Single package `components/case-explorer/` in the blocks-ui monorepo. Generic components (`entity-list`, `entity-detail`, `entity-tree`, `entity-command-bar`, `case-explorer`) consume `EntityTypeRegistration` declarations. Data flow via DataSourceMixin/cursor-aware fetch per component, coordination via `emitPagesEvent`. NavigationController (Lit ReactiveController) manages breadcrumbs, entity type switching, and list/tree mode. Convenience wrappers provide drop-in presets.

**Tech Stack:** Lit 3.x, TypeScript 5.6+, Vitest, `@casehubio/pages-*` packages, `@casehubio/blocks-ui-core`

## Global Constraints

- All events use `emitPagesEvent()` from `@casehubio/blocks-ui-core`
- Domain customisation per protocol PP-20260713-8ea1af: typed config + render callbacks + factory overrides, no slots for content
- `LiveRegionMixin` from `@casehubio/pages-primitives` for accessibility announcements
- `experimentalDecorators: true`, `useDefineForClassFields: false` in tsconfig
- Package name: `@casehubio/blocks-ui-case-explorer`
- `entity-list` does NOT extend DataSourceMixin — it owns cursor-aware fetch directly and passes `TypedDataSet` to `list-pane` via data-property mode
- `entity-detail` does NOT wrap `detail-pane` — it owns its own fetch and rendering because it needs the full `EntityInstance` (commands, state), not the tabular `TypedRow` from selection events
- Three-tier detail renderer resolution: sub-type-specific (`detailRendererMap`) → entity-type-specific (`detailRenderer`) → default state table
- Compound entity types for SPI routing: `"worker:flow"`, `"worker:agent"`, etc.
- Single WebSocket connection via `EventStreamController` with topic-based multiplexing
- Tree lazy loading: first two levels inline, deeper levels via `childrenEndpoint`

---

### Task 1: Package scaffold and types

**Files:**
- Create: `components/case-explorer/package.json`
- Create: `components/case-explorer/tsconfig.json`
- Create: `components/case-explorer/tsconfig.build.json`
- Create: `components/case-explorer/vitest.config.ts`
- Create: `components/case-explorer/src/types.ts`
- Create: `components/case-explorer/src/index.ts`
- Test: `components/case-explorer/src/types.test.ts`

**Interfaces:**
- Consumes: nothing (foundation task)
- Produces: All TypeScript interfaces — `EntityInstance`, `CommandDescriptor`, `ParameterDescriptor`, `SelectOption`, `EntityTypeRegistration`, `RelationshipDeclaration`, `EntityListResponse`, `FilterDescriptor`, `NavigationState`, `BreadcrumbEntry`, `EntitySelection`, `EntityTreeNode`, `GroupInfo`, `EntityEvent`, `ColumnConfig` (re-export from pages-table's `TableColumnConfig`), `ColumnRenderer` (re-export), `DetailRenderer` type alias

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/blocks-ui-case-explorer",
  "version": "0.1.0",
  "description": "Composable case explorer — universal entity browser with registration, state/command SPI, and management actions",
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
    "@casehubio/pages-component": "^0.2.2",
    "@casehubio/pages-data": "^0.2.2",
    "@casehubio/pages-primitives": "^0.2.2",
    "@casehubio/pages-table": "^0.2.2",
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

- [ ] **Step 2: Create tsconfig.json and tsconfig.build.json**

tsconfig.json:
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

tsconfig.build.json:
```json
{ "extends": "./tsconfig.json", "exclude": ["src/**/*.test.ts"] }
```

- [ ] **Step 3: Create vitest.config.ts**

Follow the exact alias pattern from routing-rationale (existsSync guards for pages packages, blocks-ui-core alias, esbuild target ES2022, jsdom environment).

- [ ] **Step 4: Write types.ts with all interfaces**

All interfaces from spec §1, §2, §4, §6. Use `readonly` on all fields. Import `TableColumnConfig` and `ColumnRenderer` from `@casehubio/pages-table` and re-export as `ColumnConfig` and `ColumnRenderer`.

```typescript
import type { TableColumnConfig, ColumnRenderer } from '@casehubio/pages-table';
import type { TemplateResult } from 'lit';

export type ColumnConfig = TableColumnConfig;
export type { ColumnRenderer };
export type DetailRenderer = (entity: EntityInstance) => TemplateResult;

export interface EntityInstance {
  readonly id: string;
  readonly type: string;
  readonly status: string;
  readonly summary: string;
  readonly state: Record<string, unknown>;
  readonly availableCommands: readonly CommandDescriptor[];
  readonly createdAt: string;
  readonly updatedAt?: string;
}

export interface CommandDescriptor {
  readonly name: string;
  readonly label: string;
  readonly description?: string;
  readonly parameters?: readonly ParameterDescriptor[];
  readonly confirmation?: boolean;
  readonly confirmMessage?: string;
  readonly severity?: 'normal' | 'destructive';
  readonly endpoint: string;
  readonly method?: string;
}

export interface ParameterDescriptor {
  readonly name: string;
  readonly label: string;
  readonly type: 'string' | 'number' | 'boolean' | 'select';
  readonly required?: boolean;
  readonly options?: readonly SelectOption[];
  readonly defaultValue?: unknown;
}

export interface SelectOption {
  readonly value: string;
  readonly label: string;
}

export interface EntityTypeRegistration {
  readonly type: string;
  readonly label: string;
  readonly icon?: string;
  readonly listEndpoint: string;
  readonly detailEndpoint: (id: string) => string;
  readonly columnConfig: readonly ColumnConfig[];
  readonly columnRenderers?: Record<string, ColumnRenderer>;
  readonly detailRenderer?: DetailRenderer;
  readonly detailRendererMap?: Record<string, string | DetailRenderer>;
  readonly relationships?: readonly RelationshipDeclaration[];
  readonly filters?: readonly FilterDescriptor[];
  readonly subTypes?: readonly string[];
  readonly treeEndpoint?: (rootId: string) => string;
  readonly eventTopics?: readonly string[];
}

export interface RelationshipDeclaration {
  readonly childType: string;
  readonly label: string;
  readonly endpointTemplate: string;
}

export interface EntityListResponse {
  readonly entities: readonly EntityInstance[];
  readonly nextCursor?: string;
  readonly totalCount?: number;
}

export interface FilterDescriptor {
  readonly field: string;
  readonly label: string;
  readonly type: 'text' | 'select' | 'date-range' | 'status';
  readonly options?: readonly SelectOption[];
}

export interface NavigationState {
  currentEntityType: string;
  selectedEntityId: string | null;
  viewMode: 'list' | 'tree';
  breadcrumbs: readonly BreadcrumbEntry[];
  availableEntityTypes: readonly EntityTypeRegistration[];
}

export interface BreadcrumbEntry {
  readonly entityType: string;
  readonly entityId: string;
  readonly label: string;
  readonly listEndpoint: string;
}

export interface EntitySelection {
  readonly id: string;
  readonly type: string;
}

export interface EntityTreeNode {
  readonly id: string;
  readonly type: string;
  readonly label: string;
  readonly status: string;
  readonly icon?: string;
  readonly children?: readonly EntityTreeNode[];
  readonly childrenEndpoint?: string;
  readonly childCount?: number;
  readonly groupInfo?: GroupInfo;
}

export interface GroupInfo {
  readonly groupId: string;
  readonly totalInGroup: number;
  readonly requiredCount: number;
  readonly completedCount: number;
}

export interface EntityEvent {
  readonly entityType: string;
  readonly entityId: string;
  readonly eventType: string;
  readonly data?: Record<string, unknown>;
  readonly timestamp: string;
}
```

- [ ] **Step 5: Write type-level tests**

```typescript
// types.test.ts — verify interfaces are structurally sound
import { describe, it, expectTypeOf } from 'vitest';
import type {
  EntityInstance, CommandDescriptor, EntityTypeRegistration,
  EntitySelection, EntityTreeNode, EntityListResponse, EntityEvent,
} from './types.js';

describe('types', () => {
  it('EntityInstance has required fields', () => {
    expectTypeOf<EntityInstance>().toHaveProperty('id');
    expectTypeOf<EntityInstance>().toHaveProperty('availableCommands');
    expectTypeOf<EntityInstance['availableCommands']>().toEqualTypeOf<readonly CommandDescriptor[]>();
  });

  it('EntityTypeRegistration detailEndpoint is a function', () => {
    expectTypeOf<EntityTypeRegistration['detailEndpoint']>().toBeFunction();
  });

  it('EntitySelection carries id and type', () => {
    expectTypeOf<EntitySelection>().toHaveProperty('id');
    expectTypeOf<EntitySelection>().toHaveProperty('type');
  });

  it('EntityTreeNode children are recursive', () => {
    expectTypeOf<NonNullable<EntityTreeNode['children']>>()
      .toEqualTypeOf<readonly EntityTreeNode[]>();
  });
});
```

- [ ] **Step 6: Create barrel export index.ts**

```typescript
export type * from './types.js';
```

- [ ] **Step 7: Run yarn install and tests**

Run: `cd components/case-explorer && yarn install && npx vitest run`
Expected: All type tests pass.

- [ ] **Step 8: Commit**

```
feat(#87): case-explorer package scaffold and type definitions
```

---

### Task 2: entity-command-bar

**Files:**
- Create: `components/case-explorer/src/entity-command-bar.ts`
- Test: `components/case-explorer/src/entity-command-bar.test.ts`
- Modify: `components/case-explorer/src/index.ts`

**Interfaces:**
- Consumes: `CommandDescriptor`, `ParameterDescriptor`, `SelectOption` from types.ts
- Produces: `EntityCommandBar` component, `EntityCommandBarTopics.COMMAND_EXECUTED` event topic, `EntityCommandBarTopics.ENTITY_CHANGED` event topic

- [ ] **Step 1: Write failing tests**

Test cases:
1. Renders a button for each command in `availableCommands`
2. Destructive commands render with danger styling
3. Clicking a non-confirmation command POSTs to the command's endpoint
4. Clicking a confirmation command opens `blocks-confirm-dialog` first
5. On successful POST, emits `entity-changed` event via `emitPagesEvent`
6. On failed POST, shows error message
7. Commands with parameters render a parameter form before executing
8. Empty `availableCommands` renders nothing

Mock fetch for POST tests. Use `vi.fn()` for fetch mock (same pattern as notification-inbox api.test.ts).

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd components/case-explorer && npx vitest run`
Expected: FAIL — component not defined

- [ ] **Step 3: Implement entity-command-bar**

```typescript
@customElement('entity-command-bar')
export class EntityCommandBar extends LiveRegionMixin(LitElement) {
  @property({ attribute: false }) commands: readonly CommandDescriptor[] = [];
  @property({ type: String }) entityId = '';
  @property({ type: String }) entityType = '';
```

Key implementation points:
- Import `LiveRegionMixin` from `@casehubio/pages-primitives`
- Import `emitPagesEvent` from `@casehubio/blocks-ui-core`
- Import `'@casehubio/blocks-ui-core/dist/confirm-dialog/blocks-confirm-dialog.js'`
- Each command renders as a `<button>` with `@click` handler
- `severity === 'destructive'` → `confirmVariant="danger"` on the dialog
- POST uses injectable `fetchFn` (same pattern as NotificationApi) for testability
- Announce command result via `this.announce()` (LiveRegionMixin)
- Emit `emitPagesEvent(this, EntityCommandBarTopics.ENTITY_CHANGED, { entityType, entityId })`

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Update index.ts barrel exports**

```typescript
export type * from './types.js';
export { EntityCommandBar, EntityCommandBarTopics } from './entity-command-bar.js';
```

- [ ] **Step 6: Commit**

```
feat(#87): entity-command-bar — dynamic command rendering with confirmation
```

---

### Task 3: entity-list

**Files:**
- Create: `components/case-explorer/src/entity-list.ts`
- Test: `components/case-explorer/src/entity-list.test.ts`
- Modify: `components/case-explorer/src/index.ts`

**Interfaces:**
- Consumes: `EntityTypeRegistration`, `EntityListResponse`, `EntityInstance`, `EntitySelection` from types.ts; `list-pane` component from `@casehubio/blocks-ui-list-pane`
- Produces: `EntityList` component, `EntityListTopics.ENTITY_SELECTED` event topic

- [ ] **Step 1: Write failing tests**

Test cases:
1. Fetches from `registration.listEndpoint` on connectedCallback
2. Converts `EntityListResponse.entities` to `TypedDataSet` and sets on inner `list-pane`
3. Emits `EntitySelection { id, type }` on row activation (not raw `TypedRow`)
4. Renders "load more" control when `nextCursor` is present
5. "Load more" fetches next page, appends entities, rebuilds `TypedDataSet`
6. Filter changes reset cursor and re-fetch from first page
7. Renders filter controls from `registration.filters`
8. Shows loading state during fetch
9. Shows error state on fetch failure with retry action
10. Shows empty state when no entities returned

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement entity-list**

```typescript
@customElement('entity-list')
export class EntityList extends LiveRegionMixin(LitElement) {
  @property({ attribute: false }) registration?: EntityTypeRegistration;
  @property({ type: String }) endpoint = '';
  @property({ type: String, attribute: 'selection-topic' }) selectionTopic = '';
  @property({ attribute: false }) fetchFn: typeof fetch = fetch;

  @state() private _loading = false;
  @state() private _error: string | null = null;
  @state() private _entities: EntityInstance[] = [];
  @state() private _nextCursor: string | null = null;
  @state() private _filters: Record<string, string> = {};
```

Key implementation:
- Does NOT extend DataSourceMixin — owns cursor-aware JSON fetch directly
- Resolves endpoint from `registration?.listEndpoint ?? this.endpoint`
- On fetch success: convert entities to `TypedDataSet` via `fromRows()` from `@casehubio/pages-data`, set on `list-pane.dataSet`
- On row-activate from inner `list-pane`: extract entity ID from `TypedRow`, emit `EntitySelection { id, type: registration.type }` via `emitPagesEvent`
- Add `@casehubio/blocks-ui-list-pane` as a dependency in package.json
- Filter rendering: `FilterDescriptor` → `<select>`, `<input>`, etc. Filter changes call `_resetAndFetch()`

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Update index.ts and package.json**

Add `"@casehubio/blocks-ui-list-pane": "workspace:*"` to dependencies.

- [ ] **Step 6: Commit**

```
feat(#87): entity-list — cursor-aware entity listing with filter controls
```

---

### Task 4: entity-detail

**Files:**
- Create: `components/case-explorer/src/entity-detail.ts`
- Test: `components/case-explorer/src/entity-detail.test.ts`
- Modify: `components/case-explorer/src/index.ts`

**Interfaces:**
- Consumes: `EntityTypeRegistration`, `EntityInstance`, `EntitySelection`, `CommandDescriptor` from types.ts; `EntityCommandBar` from entity-command-bar.ts
- Produces: `EntityDetail` component

- [ ] **Step 1: Write failing tests**

Test cases:
1. Fetches `EntityInstance` from `detailEndpoint(id)` on `EntitySelection` event
2. Three-tier detail renderer resolution:
   a. Sub-type match in `detailRendererMap` → renders registered component
   b. Entity-type `detailRenderer` callback → renders callback result
   c. Default → renders state key-value table
3. Passes `availableCommands` to inner `entity-command-bar`
4. Renders relationship tabs from `registration.relationships` — each tab contains an `entity-list` for the child type with endpoint from `endpointTemplate` (parentId substituted)
5. Shows loading state during fetch
6. Shows error state on fetch failure
7. Shows empty state when no entity selected
8. Re-fetches on `entity-changed` event matching current entityId

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement entity-detail**

```typescript
@customElement('entity-detail')
export class EntityDetail extends LiveRegionMixin(LitElement) {
  @property({ attribute: false }) registration?: EntityTypeRegistration;
  @property({ type: String, attribute: 'selection-topic' }) selectionTopic = '';
  @property({ attribute: false }) fetchFn: typeof fetch = fetch;

  @state() private _entity: EntityInstance | null = null;
  @state() private _loading = false;
  @state() private _error: string | null = null;
  @state() private _activeTab = 0;
```

Key implementation:
- Listens for `EntitySelection` on `selectionTopic` via `onPagesEvent`
- On selection: fetch from `registration.detailEndpoint(selection.id)`
- Render detail content via `_resolveRenderer()`:
  - Check `registration.detailRendererMap?.[entity.type]` first (compound type e.g. `"worker:flow"`)
  - Then check `registration.detailRenderer`
  - Then default state table (render each key-value in `entity.state`)
- Render `<entity-command-bar .commands=${entity.availableCommands}>`
- Render relationship tabs: for each relationship, render `<entity-list .endpoint=${resolved} .registration=${childRegistration}>`
- Tab rendering uses ARIA `role="tablist"` / `role="tab"` with keyboard navigation

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Update index.ts**

- [ ] **Step 6: Commit**

```
feat(#87): entity-detail — polymorphic detail with three-tier rendering
```

---

### Task 5: entity-tree

**Files:**
- Create: `components/case-explorer/src/entity-tree.ts`
- Test: `components/case-explorer/src/entity-tree.test.ts`
- Modify: `components/case-explorer/src/index.ts`

**Interfaces:**
- Consumes: `EntityTreeNode`, `GroupInfo`, `EntitySelection` from types.ts
- Produces: `EntityTree` component, `EntityTreeTopics.NODE_SELECTED` event topic

- [ ] **Step 1: Write failing tests**

Test cases:
1. Renders tree nodes with icon, label, and status badge
2. Expand/collapse toggles `aria-expanded` and shows/hides children
3. Sub-case group nodes render M-of-N progress from `groupInfo`
4. Clicking a node emits `EntitySelection { id, type }` via `emitPagesEvent`
5. Lazy loading: expanding a node with `childrenEndpoint` fetches children and inserts them
6. Arrow key navigation: up/down between siblings, left to collapse, right to expand
7. `role="tree"` on container, `role="treeitem"` on nodes
8. Custom `nodeRenderer` callback overrides default node rendering
9. Nodes with `childCount > 0` but no inline `children` show expand affordance

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement entity-tree**

```typescript
@customElement('entity-tree')
export class EntityTree extends LiveRegionMixin(LitElement) {
  @property({ attribute: false }) nodes: readonly EntityTreeNode[] = [];
  @property({ type: String, attribute: 'selection-topic' }) selectionTopic = '';
  @property({ attribute: false }) nodeRenderer?: (node: EntityTreeNode) => TemplateResult;
  @property({ attribute: false }) fetchFn: typeof fetch = fetch;

  @state() private _expandedIds = new Set<string>();
  @state() private _loadingIds = new Set<string>();
  @state() private _selectedId: string | null = null;
  @state() private _lazyChildren = new Map<string, readonly EntityTreeNode[]>();
```

Key implementation:
- ARIA tree pattern: `role="tree"` on root `<ul>`, `role="treeitem"` on each `<li>`
- `aria-expanded` on nodes with children
- Keyboard handler: ArrowUp/Down/Left/Right/Enter/Space
- Lazy expand: on expand of node with `childrenEndpoint`, fetch and store in `_lazyChildren` map, merge with `children`
- Group node rendering: `${groupInfo.completedCount}/${groupInfo.requiredCount} of ${groupInfo.totalInGroup}`
- Default node render: `<span class="icon">${icon}</span><span class="label">${label}</span><span class="status-badge">${status}</span>`
- nodeRenderer override: `this.nodeRenderer?.(node) ?? this._defaultNodeRender(node)`
- On node click: `emitPagesEvent(this, selectionTopic + ':selected', { id, type })`

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Update index.ts**

- [ ] **Step 6: Commit**

```
feat(#87): entity-tree — collapsible hierarchy with lazy loading and ARIA tree
```

---

### Task 6: NavigationController

**Files:**
- Create: `components/case-explorer/src/navigation-controller.ts`
- Test: `components/case-explorer/src/navigation-controller.test.ts`
- Modify: `components/case-explorer/src/index.ts`

**Interfaces:**
- Consumes: `NavigationState`, `BreadcrumbEntry`, `EntityTypeRegistration`, `EntitySelection`, `EntityEvent` from types.ts
- Produces: `NavigationController` class (Lit ReactiveController)

- [ ] **Step 1: Write failing tests**

Test cases (unit tests — NavigationController is a pure state manager, no DOM):
1. Initial state: first entity type selected, list mode, empty breadcrumbs
2. `selectEntityType(type)` switches current type, resets selection and breadcrumbs
3. `selectEntity(selection)` sets `selectedEntityId`
4. `drillDown(entityType, entityId, label)` pushes breadcrumb, switches type, sets selection
5. `navigateBack(index)` pops breadcrumbs to given index, restores that entity type and selection
6. `setViewMode('tree')` switches to tree mode; `setViewMode('list')` switches back
7. `handleEntityEvent(event)` — if event.entityId matches selectedEntityId, triggers host re-render
8. Breadcrumb trail builds correctly through multi-level drill-down

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement NavigationController**

```typescript
import { type ReactiveController, type ReactiveControllerHost } from 'lit';

export class NavigationController implements ReactiveController {
  private _state: NavigationState;

  constructor(
    private readonly host: ReactiveControllerHost,
    entityTypes: readonly EntityTypeRegistration[],
  ) {
    this._state = {
      currentEntityType: entityTypes[0]?.type ?? '',
      selectedEntityId: null,
      viewMode: 'list',
      breadcrumbs: [],
      availableEntityTypes: entityTypes,
    };
    host.addController(this);
  }

  get state(): Readonly<NavigationState> { return this._state; }

  hostConnected(): void {}
  hostDisconnected(): void {}

  selectEntityType(type: string): void { ... }
  selectEntity(selection: EntitySelection): void { ... }
  drillDown(entityType: string, entityId: string, label: string): void { ... }
  navigateBack(breadcrumbIndex: number): void { ... }
  setViewMode(mode: 'list' | 'tree'): void { ... }
  handleEntityEvent(event: EntityEvent): boolean { ... }

  getRegistration(type: string): EntityTypeRegistration | undefined { ... }
}
```

Key implementation:
- Pure state manager — no DOM, no events (the host component emits events)
- Each mutation calls `this.host.requestUpdate()` to trigger re-render
- `drillDown` pushes `{ entityType: currentEntityType, entityId: selectedEntityId, label, listEndpoint }` to breadcrumbs before switching
- `navigateBack` slices breadcrumbs to index, restores state from the breadcrumb entry

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Update index.ts**

- [ ] **Step 6: Commit**

```
feat(#87): NavigationController — breadcrumbs, drill-down, view mode state
```

---

### Task 7: case-explorer (full composition)

**Files:**
- Create: `components/case-explorer/src/case-explorer.ts`
- Test: `components/case-explorer/src/case-explorer.test.ts`
- Modify: `components/case-explorer/src/index.ts`

**Interfaces:**
- Consumes: All previous components + `NavigationController` + `EntityTypeRegistration` + split-workbench
- Produces: `CaseExplorer` component

- [ ] **Step 1: Write failing tests**

Test cases:
1. Renders entity type tabs from `entityTypes` registrations
2. Selecting a tab switches the entity-list to that type
3. Shows entity-list in list mode, entity-tree in tree mode
4. View mode toggle switches between list and tree
5. Breadcrumb bar renders from NavigationController state
6. Clicking a breadcrumb navigates back
7. Entity selection in list mode loads entity-detail on the right
8. Drill-down from entity-detail switches left panel to tree mode
9. Uses split-workbench for responsive layout
10. `entity-changed` events from command-bar trigger re-fetch in visible components

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement case-explorer**

```typescript
@customElement('case-explorer')
export class CaseExplorer extends LiveRegionMixin(LitElement) {
  @property({ attribute: false }) entityTypes: readonly EntityTypeRegistration[] = [];

  private _nav?: NavigationController;

  override connectedCallback(): void {
    super.connectedCallback();
    this._nav = new NavigationController(this, this.entityTypes);
  }
```

Key implementation:
- Renders `<split-workbench>` with list/tree in left slot, entity-detail in right slot
- Entity type tabs: `role="tablist"` with `role="tab"` buttons
- Left panel: conditionally renders `<entity-list>` or `<entity-tree>` based on `_nav.state.viewMode`
- Right panel: `<entity-detail>` with current registration
- Breadcrumb bar above left panel
- View mode toggle: list/tree radio group
- Coordinates events: entity-selected → updates NavigationController → triggers re-render
- Add `@casehubio/blocks-ui-split-workbench` to package.json dependencies

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Update index.ts and package.json**

Add `"@casehubio/blocks-ui-split-workbench": "workspace:*"` to dependencies.

- [ ] **Step 6: Commit**

```
feat(#87): case-explorer — full composed entity browser with split-workbench
```

---

### Task 8: Presets and convenience components

**Files:**
- Create: `components/case-explorer/src/presets.ts`
- Create: `components/case-explorer/src/convenience/case-instance-list.ts`
- Create: `components/case-explorer/src/convenience/worker-list.ts`
- Create: `components/case-explorer/src/convenience/case-definition-browser.ts`
- Create: `components/case-explorer/src/convenience/case-detail-panel.ts`
- Create: `components/case-explorer/src/convenience/worker-detail-panel.ts`
- Test: `components/case-explorer/src/presets.test.ts`
- Test: `components/case-explorer/src/convenience/case-instance-list.test.ts`
- Modify: `components/case-explorer/src/index.ts`

**Interfaces:**
- Consumes: `EntityTypeRegistration`, `EntityList`, `EntityDetail` from previous tasks
- Produces: `caseInstanceType()`, `caseDefinitionType()`, `workerType()`, `gateType()`, `channelType()` factory functions; convenience wrapper components

- [ ] **Step 1: Write failing tests for presets**

Test cases:
1. `caseInstanceType({ listEndpoint })` returns valid `EntityTypeRegistration` with default columns (name, status, started, active workers)
2. `workerType({ listEndpoint })` returns registration with default columns (name, type, status, case, progress)
3. `caseDefinitionType({ listEndpoint })` returns registration with default columns (namespace, name, version, workers)
4. Presets can be spread and overridden: `{ ...workerType({ listEndpoint: '/api/workers' }), columnRenderers: { custom } }`
5. `workerType` declares `subTypes: ['worker:flow', 'worker:agent', 'worker:human']`
6. `caseInstanceType` declares relationships to workers and sub-cases

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement presets**

```typescript
export function caseInstanceType(config: { listEndpoint: string }): EntityTypeRegistration {
  return {
    type: 'case-instance',
    label: 'Cases',
    listEndpoint: config.listEndpoint,
    detailEndpoint: (id) => `${config.listEndpoint}/${id}`,
    columnConfig: [
      { key: 'summary', header: 'Name' },
      { key: 'status', header: 'Status' },
      { key: 'createdAt', header: 'Started' },
    ],
    relationships: [
      { childType: 'worker', label: 'Workers', endpointTemplate: `${config.listEndpoint}/{parentId}/workers` },
      { childType: 'case-instance', label: 'Sub-cases', endpointTemplate: `${config.listEndpoint}/{parentId}/sub-cases` },
    ],
    treeEndpoint: (rootId) => `${config.listEndpoint}/${rootId}/tree`,
    eventTopics: ['case-instance'],
  };
}

export function workerType(config: { listEndpoint: string }): EntityTypeRegistration { ... }
export function caseDefinitionType(config: { listEndpoint: string }): EntityTypeRegistration { ... }
export function gateType(config: { listEndpoint: string }): EntityTypeRegistration { ... }
export function channelType(config: { listEndpoint: string }): EntityTypeRegistration { ... }
```

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Implement convenience components**

Each convenience component is a thin wrapper that creates a default registration from its properties and renders the generic component. Example:

```typescript
@customElement('case-instance-list')
export class CaseInstanceList extends LitElement {
  @property({ type: String }) endpoint = '';
  @property({ attribute: false }) columnRenderers?: Record<string, ColumnRenderer>;
  @property({ attribute: false }) filters?: readonly FilterDescriptor[];
  @property({ type: String, attribute: 'selection-topic' }) selectionTopic = 'case';

  override render() {
    const reg = { ...caseInstanceType({ listEndpoint: this.endpoint }), columnRenderers: this.columnRenderers, filters: this.filters };
    return html`<entity-list .registration=${reg} selection-topic=${this.selectionTopic}></entity-list>`;
  }
}
```

Write one test per convenience component: renders inner generic component with correct registration.

- [ ] **Step 6: Write convenience component tests**

- [ ] **Step 7: Run all tests**

Run: `cd components/case-explorer && npx vitest run`
Expected: All tests pass.

- [ ] **Step 8: Update index.ts with all exports**

```typescript
export type * from './types.js';
export { EntityCommandBar, EntityCommandBarTopics } from './entity-command-bar.js';
export { EntityList, EntityListTopics } from './entity-list.js';
export { EntityDetail } from './entity-detail.js';
export { EntityTree, EntityTreeTopics } from './entity-tree.js';
export { NavigationController } from './navigation-controller.js';
export { CaseExplorer } from './case-explorer.js';
export { caseInstanceType, caseDefinitionType, workerType, gateType, channelType } from './presets.js';
export { CaseInstanceList } from './convenience/case-instance-list.js';
export { WorkerList } from './convenience/worker-list.js';
export { CaseDefinitionBrowser } from './convenience/case-definition-browser.js';
export { CaseDetailPanel } from './convenience/case-detail-panel.js';
export { WorkerDetailPanel } from './convenience/worker-detail-panel.js';
```

- [ ] **Step 9: Typecheck and build**

Run: `cd components/case-explorer && npx tsc --noEmit && npx tsc -p tsconfig.build.json`
Expected: Clean build, no errors.

- [ ] **Step 10: Commit**

```
feat(#87): presets and convenience components — drop-in entity browser wrappers
```

---

## Build Order Summary

```
Task 1: types.ts (foundation)
   ↓
Task 2: entity-command-bar (standalone, no component deps)
   ↓
Task 3: entity-list (uses list-pane, emits EntitySelection)
   ↓
Task 4: entity-detail (uses entity-command-bar, listens for EntitySelection)
   ↓
Task 5: entity-tree (parallel with Task 4 — no dependency)
   ↓
Task 6: NavigationController (pure state, no component deps)
   ↓
Task 7: case-explorer (composes Tasks 2-6)
   ↓
Task 8: presets + convenience (wraps Tasks 3-4)
```

Tasks 4 and 5 are independent — could be parallelised. All others are sequential.
