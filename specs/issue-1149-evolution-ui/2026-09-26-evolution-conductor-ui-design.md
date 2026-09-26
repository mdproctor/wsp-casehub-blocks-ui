# Evolution Conductor UI — Design Spec

**Epic:** casehubio/blocks-ui#174
**Parent epic:** casehubio/engine#1149
**Issues:** #175 (deny-pattern-editor), #176 (watch-pattern-editor), #177 (gate-policy-editor), #178 (evolution-workbench), #179 (sample page)
**Decisions:** D1–D5 in `decisions.md`
**Date:** 2026-09-26

## Problem

The evolution conductor has a complete engine-side implementation (5 domain-agnostic SPIs, persistence, wiring, generic extraction — landed in engine#1148), but no UI exposure. Operators cannot manage deny patterns, watch patterns, or gate policies without direct API calls. There is no dashboard for monitoring evolution health, active improvement streams, or pending gate decisions.

## Scope

Five new deliverables in blocks-ui:

1. **deny-pattern-editor** (#175) — CRUD for static + dynamic deny patterns
2. **watch-pattern-editor** (#176) — CRUD for escalation watch patterns
3. **gate-policy-editor** (#177) — per-stage gate mode configuration
4. **evolution-workbench** (#178) — dashboard composing health, timeline, streams, inbox, and editors
5. **sample page** (#179) — standalone reference view with inline data

All components use dual data mode (endpoint or property) and follow existing blocks-ui conventions.

## 1. Package Structure

One new component package and one workbench package:

```
components/
  evolution-config/          ← #175, #176, #177
    src/
      types.ts               TypeScript interfaces mirroring engine Java records
      api.ts                 EvolutionApi class wrapping REST endpoints
      events.ts              Custom event constants
      deny-pattern-editor.ts  #175
      watch-pattern-editor.ts #176
      gate-policy-editor.ts   #177
      index.ts               Public exports
      *.test.ts              Per-element test files
    package.json
    tsconfig.json
  evolution-workbench/       ← #178, #179
    src/
      evolution-workbench.ts  Dashboard composition
      sample-data.ts          Inline data for sample page
      index.ts
      *.test.ts
    package.json
    tsconfig.json
```

The editors share `types.ts` and `api.ts` but each is an independent `LitElement` with its own rendering, column config, and form schema (D3). The gate-policy-editor is structurally different from the CRUD editors (D4).

## 2. Shared Types (`types.ts`)

TypeScript interfaces mirroring the engine's Java records:

```typescript
// --- Deny patterns ---

export interface DynamicDenyEntry {
  readonly pattern: string;
  readonly addedBy: string;
  readonly addedAt: string; // ISO-8601
}

export interface DenyPatternView {
  readonly staticPatterns: readonly string[];
  readonly dynamicPatterns: readonly DynamicDenyEntry[];
}

// --- Watch patterns ---

export interface WatchPattern {
  readonly id: string;
  readonly category: string | null;
  readonly areaId: string | null;
  readonly targetPattern: string | null;
  readonly minEstimatedSize: number | null;
  readonly createdAt: string; // ISO-8601
}

// --- Gate policy ---

export type GateMode = 'GATED' | 'AUTO' | 'NOTIFY';

export interface GatePolicy {
  readonly modes: Record<string, GateMode> | null;
  readonly gateTimeoutMinutes: number | null;
}

// --- Domain metadata (from ImprovementCategoryProvider) ---

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

// --- Evolution state (subset for workbench summary) ---

export interface EvolutionSummaryMetrics {
  readonly healthScore: number;
  readonly healthDelta: number;
  readonly activeImprovementCount: number;
  readonly pendingInboxCount: number;
  readonly circuitBreakerState: string;
  readonly evolutionEnabled: boolean;
}

// --- Conductor inbox ---

export type InboxEntryStatus = 'PENDING' | 'APPROVED' | 'REJECTED';

export interface ConductorInboxEntry {
  readonly id: string;
  readonly category: string;
  readonly stage: string;
  readonly target: string | null;
  readonly status: InboxEntryStatus;
  readonly createdAt: string;
  readonly resolvedAt: string | null;
}

// --- Improvement streams ---

export interface ImprovementStreamView {
  readonly improvementCaseId: string;
  readonly category: string;
  readonly target: string | null;
  readonly currentStage: string;
  readonly blockedBy: string | null;
  readonly conflictBlocked: boolean;
  readonly startedAt: string;
}
```

## 3. Shared API (`api.ts`)

Single `EvolutionApi` class wrapping all evolution REST endpoints. Injectable `fetchFn` for testability:

```typescript
export class EvolutionApi {
  constructor(
    private readonly baseUrl: string,
    private readonly fetchFn: typeof fetch = fetch,
  ) {}

  // --- Deny patterns (complete API) ---
  async getDenyPatterns(caseId: string, tenancyId: string): Promise<DenyPatternView>
  async addDenyPattern(caseId: string, tenancyId: string, pattern: string): Promise<void>
  async removeDenyPattern(caseId: string, tenancyId: string, pattern: string): Promise<void>

  // --- Watch patterns (getWatchPatterns pending engine issue) ---
  async getWatchPatterns(caseId: string, tenancyId: string): Promise<WatchPattern[]>
  async addWatchPattern(caseId: string, tenancyId: string, input: WatchPatternInput): Promise<void>
  async removeWatchPattern(caseId: string, tenancyId: string, patternId: string): Promise<void>

  // --- Gate policy ---
  async setGatePolicy(caseId: string, tenancyId: string, policy: GatePolicy): Promise<void>

  // --- Domain metadata (pending engine issue) ---
  async getStages(caseId: string): Promise<StageDescriptor[]>
  async getCategories(caseId: string): Promise<CategoryDescriptor[]>

  // --- State / streams / inbox ---
  async getEvolutionState(caseId: string, tenancyId: string): Promise<EvolutionSummaryMetrics>
  async getStreamProgress(caseId: string, tenancyId: string): Promise<ImprovementStreamView[]>
  async getInbox(caseId: string, tenancyId: string): Promise<ConductorInboxEntry[]>
  async resolveGate(caseId: string, tenancyId: string, entryId: string,
      outcome: InboxEntryStatus, reason?: string, feedback?: string): Promise<void>
}

export interface WatchPatternInput {
  readonly category?: string;
  readonly areaId?: string;
  readonly targetPattern?: string;
  readonly minEstimatedSize?: number;
}
```

The `@PlatformQuery`/`@PlatformMutation` annotations on `EvolutionMcpAdapter` auto-generate REST endpoints at `/api/engine/evolution/*`. The `EvolutionApi` class targets these.

## 4. Shared Events (`events.ts`)

```typescript
export const EvolutionEventTopics = {
  DENY_PATTERN_CHANGED: 'evolution:deny-pattern-changed',
  WATCH_PATTERN_CHANGED: 'evolution:watch-pattern-changed',
  GATE_POLICY_CHANGED: 'evolution:gate-policy-changed',
  GATE_RESOLVED: 'evolution:gate-resolved',
} as const;

export function emitEvolutionEvent(el: HTMLElement, topic: string, detail?: unknown): void {
  el.dispatchEvent(new CustomEvent('pages-event', {
    bubbles: true, composed: true,
    detail: { topic, ...(detail !== undefined ? { payload: detail } : {}) },
  }));
}
```

Events use `pages-event` CustomEvent — no direct component-to-component coupling. The workbench listens for change events to refresh its summary bar.

## 5. deny-pattern-editor (`<blocks-deny-pattern-editor>`)

**Element tag:** `blocks-deny-pattern-editor`

### Props

```typescript
export interface DenyPatternEditorProps {
  endpoint?: string;             // REST base URL (endpoint mode)
  caseId?: string;               // required for endpoint mode
  tenancyId?: string;            // required for endpoint mode
  patterns?: DenyPatternView;    // inline data mode
  readonly?: boolean;            // disable mutations
}
```

### Layout

Two sections:

1. **Structural patterns** — read-only list showing hardcoded deny patterns from `DenyPatternProvider` implementations. Rendered as `<ul>` with monospace font and a "Structural" chip header. These protect the evolution loop's own control path from self-modification.

2. **Dynamic patterns** — editable table (pages-table) showing user-added patterns from `DenyPatternStore`. Columns:

| Column | Width | Content |
|--------|-------|---------|
| Pattern | `1fr` | Monospace pattern string |
| Added by | `120px` | User identifier |
| Added at | `160px` | Formatted timestamp |
| Actions | `60px` | Delete button |

Below the table: inline add form (pages-form) with a single text input for the pattern string. Add button triggers `addDenyPattern()` then refreshes the table.

Delete: confirmation dialog (PagesConfirmDialog) before removal.

### Data Flow

- **Endpoint mode:** `connectedCallback` creates `EvolutionApi` from `endpoint`, fetches `getDenyPatterns(caseId, tenancyId)`. Mutations call API then re-fetch.
- **Inline mode:** renders from `patterns` property. Mutations emit `pages-event` with topic `evolution:deny-pattern-changed` carrying the mutation details. Parent is responsible for persisting.
- After each mutation: emits `evolution:deny-pattern-changed`.

### ARIA

- Host: `role="region"`, `aria-label="Deny pattern editor"`
- Structural list: `role="list"`, items `role="listitem"`
- Dynamic table: provided by pages-table
- Add form: `role="form"`, `aria-label="Add deny pattern"`

## 6. watch-pattern-editor (`<blocks-watch-pattern-editor>`)

**Element tag:** `blocks-watch-pattern-editor`

### Props

```typescript
export interface WatchPatternEditorProps {
  endpoint?: string;
  caseId?: string;
  tenancyId?: string;
  patterns?: readonly WatchPattern[];  // inline data mode
  categories?: readonly CategoryDescriptor[];  // for category dropdown
  readonly?: boolean;
}
```

### Layout

Single table (pages-table) showing active watch patterns:

| Column | Width | Content |
|--------|-------|---------|
| Category | `120px` | Category badge (or "Any" if null) |
| Area | `120px` | Capability area filter (or "Any") |
| Target pattern | `1fr` | Monospace glob pattern (or "Any") |
| Min size | `80px` | Minimum estimated change size (or "—") |
| Created | `140px` | Formatted timestamp |
| Actions | `60px` | Delete button |

Below the table: inline add form (pages-form) with JSON Schema:
- `category`: optional select from `categories` prop (or free text if no categories provided)
- `areaId`: optional text
- `targetPattern`: optional text (glob pattern)
- `minEstimatedSize`: optional number

At least one field must be non-empty — a completely empty watch pattern matches everything and would create noise.

Delete: confirmation dialog before removal.

### Data Flow

Same dual-data pattern as deny-pattern-editor. Emits `evolution:watch-pattern-changed`.

### ARIA

- Host: `role="region"`, `aria-label="Watch pattern editor"`
- Table: provided by pages-table
- Add form: `role="form"`, `aria-label="Add watch pattern"`

## 7. gate-policy-editor (`<blocks-gate-policy-editor>`)

**Element tag:** `blocks-gate-policy-editor`

### Props

```typescript
export interface GatePolicyEditorProps {
  endpoint?: string;
  caseId?: string;
  tenancyId?: string;
  stages?: readonly StageDescriptor[];  // inline data mode — available stages
  policy?: GatePolicy;                  // inline data mode — current policy
  readonly?: boolean;
}
```

### Layout

Grouped table with inline mode selectors. Stages grouped by `domainId`, ordered by `ordinal` within each group.

| Column | Width | Content |
|--------|-------|---------|
| Stage | `1fr` | Stage name |
| Ordinal | `60px` | Position in domain pipeline |
| Gate checkpoint | `80px` | Checkpoint indicator badge (●/—) |
| Mode | `140px` | `<select>` dropdown: GATED/AUTO/NOTIFY for checkpoints, AUTO/NOTIFY for non-checkpoints |

Domain header rows use `grouped-data-view` pattern — domain name as a section header.

Below the table: gate timeout configuration — a number input for `gateTimeoutMinutes` with the current value (default: 1440 = 24h).

A "Save" button submits the entire policy. Changes are batched (unlike deny/watch which mutate one-at-a-time) because `GatePolicy` is a single object (`Map<String, GateMode>` + timeout).

### Data Flow

- **Endpoint mode:** Fetches stages via `getStages(caseId)`, current policy from case configuration. On save, calls `setGatePolicy()`.
- **Inline mode:** Renders from `stages` and `policy` properties. On save, emits `evolution:gate-policy-changed` with the new `GatePolicy` object.
- Changes tracked via local `_pendingModes: Map<string, GateMode>`. Dirty state tracked — Save button disabled until changes exist.

### ARIA

- Host: `role="form"`, `aria-label="Gate policy editor"`
- Domain groups: `role="group"`, `aria-label` with domain name
- Mode selectors: native `<select>` elements (inherent ARIA)
- Save button: `aria-disabled` when no changes

## 8. evolution-workbench (`<blocks-evolution-workbench>`)

**Element tag:** `blocks-evolution-workbench`

### Props

```typescript
export interface EvolutionWorkbenchProps {
  endpoint?: string;
  caseId?: string;
  tenancyId?: string;
  tabs?: readonly TabDefinition[];     // domain-extensible extra tabs
  // Inline data mode (all optional):
  metrics?: EvolutionSummaryMetrics;
  streams?: readonly ImprovementStreamView[];
  inbox?: readonly ConductorInboxEntry[];
  denyPatterns?: DenyPatternView;
  watchPatterns?: readonly WatchPattern[];
  stages?: readonly StageDescriptor[];
  categories?: readonly CategoryDescriptor[];
  gatePolicy?: GatePolicy;
}
```

### Layout

**Summary bar** (always visible): `kpi-metric-row` with 5 metrics:

| Metric | Source | Display |
|--------|--------|---------|
| Health | `healthScore` | Percentage with sparkline (TrendSourceMixin) |
| Active | `activeImprovementCount` | Count |
| Inbox | `pendingInboxCount` | Count with badge highlight when > 0 |
| Circuit breaker | `circuitBreakerState` | StatusBadge |
| Enabled | `evolutionEnabled` | On/Off indicator |

**Tabbed content** (below summary bar) using `detail-pane` tab pattern:

| Tab | Badge | Content |
|-----|-------|---------|
| Timeline | — | `blocks-timeline` with evolution-events strategy |
| Streams | active count | pages-table with `ImprovementStreamView` data |
| Inbox | pending count | pages-table with `ConductorInboxEntry` data + resolve actions |
| Configuration | — | Three editors stacked vertically with section headers |

Domain-extensible tabs via the `tabs` property — consuming apps add `TabDefinition` entries for domain-specific views (code-evolution CI panel, trading risk panel, etc.).

### Three consumption tiers

1. **Standalone** — set `endpoint`, `caseId`, `tenancyId`. Workbench fetches all data from REST API.
2. **Panel-hosted** — registered as a pages panel via `registerPanel`. Receives case context from host app.
3. **Inline data** — all data passed as properties. No network calls. Used by the sample page (#179).

### Event handling

Listens for `evolution:deny-pattern-changed`, `evolution:watch-pattern-changed`, `evolution:gate-policy-changed`, and `evolution:gate-resolved` events from child editors. On any change event, refreshes the summary bar metrics.

### ARIA

- Host: `role="region"`, `aria-label="Evolution workbench"`
- Summary bar: via kpi-metric-row's built-in ARIA
- Tabs: via detail-pane's built-in ARIA (tablist/tab/tabpanel)

## 9. Sample Page (#179)

Static HTML page at `components/evolution-workbench/sample/index.html` that instantiates `<blocks-evolution-workbench>` in inline data mode with representative sample data. Serves as:

- Development aid for visual verification
- Documentation reference for consuming apps
- Regression test baseline

Sample data in `sample-data.ts` covers: 5 health metrics, 3 active streams (one blocked), 2 inbox entries (one pending), structural + dynamic deny patterns, 3 watch patterns, code-evolution stages with a mixed gate policy.

## 10. Engine Issues to File

API gaps requiring engine-side additions:

| Issue | What | Why |
|-------|------|-----|
| `getWatchPatterns(caseId, tenancyId)` | Expose `WatchPatternStore.findActive()` via `EngineEvolutionApi` and `EvolutionMcpAdapter` | watch-pattern-editor needs to list existing patterns |
| `getStages(caseId)` | Query `ImprovementCategoryRegistry.stagesForDomain()` for all registered domains | gate-policy-editor needs domain-contributed stage metadata |
| `getCategories(caseId)` | Query `ImprovementCategoryRegistry.allCategories()` | watch-pattern-editor category dropdown, workbench category display |
| `getGatePolicy(caseId, tenancyId)` | Read current gate policy from case configuration | gate-policy-editor needs current policy to pre-populate |

All four are thin adapter methods — the underlying data is already available internally. `@PlatformQuery` on the MCP adapter generates the REST endpoints automatically.

## 11. Test Strategy

### Per-element tests

Each editor test file covers:

| Dimension | What to test |
|-----------|-------------|
| Rendering | Correct table columns, row counts, section layout |
| ARIA | Role, aria-label on host, form regions |
| Inline data mode | Renders from properties without endpoint |
| Endpoint data mode | API calls triggered on connect, data rendered |
| Add flow | Form submission creates entry, emits change event |
| Delete flow | Confirmation dialog shown, delete triggers removal, emits change event |
| Readonly mode | No add form, no delete buttons, no mutations |
| Error handling | API failure shows error state, doesn't crash |

Gate-policy-editor additionally: mode dropdown constraint (GATED only for checkpoints), dirty state tracking, save button enable/disable, domain grouping.

Workbench additionally: tab navigation, summary bar metrics, event-driven refresh, domain-extensible tabs, three consumption tiers.

### Schema registry

Add all new elements to `BlocksComponentRegistry` in `packages/blocks-ui-schema/src/registry.ts`:

```typescript
'blocks-deny-pattern-editor': DenyPatternEditorProps,
'blocks-watch-pattern-editor': WatchPatternEditorProps,
'blocks-gate-policy-editor': GatePolicyEditorProps,
'blocks-evolution-workbench': EvolutionWorkbenchProps,
```

Run `yarn workspace @casehubio/blocks-ui-schema run generate` to produce Zod schemas.

## References

- `EngineEvolutionApi.java` — engine API surface (deny patterns, watch patterns, gate policy, state, streams, inbox)
- `EvolutionMcpAdapter.java` — REST exposure via `@PlatformQuery`/`@PlatformMutation`
- `DenyPatternView.java` — static + dynamic deny pattern view record
- `WatchPattern.java` — watch pattern record
- `GatePolicy.java:21-40` — gate policy record with `Map<String, GateMode>`
- `StageDescriptor.java` — domain-contributed stage metadata
- `CategoryDescriptor.java` — domain-contributed category metadata
- `ImprovementConfig.java` — case configuration including gatePolicy
- `WatchPatternStore.java` — internal store with `findActive()` (not exposed via API)
- `EvolutionStateSnapshot.java` — full state view (healthScore, componentScores, categoryStates, activeStreams, etc.)
- `components/notification-inbox/src/` — precedent for multi-element package with shared api.ts and types.ts
- `components/preferences-editor/src/api.ts` — API class pattern with injectable fetchFn
- `components/notification-inbox/src/mute-list.ts` — CRUD table + inline add form + delete confirmation precedent
- PP-20260713-8ea1af — component customisation via typed config + render callbacks + factory overrides
- D9 engine decision — blocks-ui provides composable primitives; apps compose domain workbench
- D1–D5 in `decisions.md`
