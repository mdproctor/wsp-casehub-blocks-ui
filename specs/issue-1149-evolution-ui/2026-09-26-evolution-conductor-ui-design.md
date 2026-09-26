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

TypeScript interfaces mirroring the engine's Java records. Every field maps 1:1 to the engine's Java record — no subsetting, no renaming.

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
export type GateOutcome = 'APPROVED' | 'REJECTED';

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

// --- Conductor inbox (mirrors ConductorInboxEntry.java) ---

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
  readonly queuedAt: string; // ISO-8601
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

The workbench summary bar computes its display from `EvolutionStateSnapshot` client-side — no separate summary type. `healthDelta` is a field on `EvolutionStateSnapshot` (computed server-side from the health window).

## 3. Shared API (`api.ts`)

Single `EvolutionApi` class wrapping all evolution REST endpoints. Injectable `fetchFn` for testability:

```typescript
export class EvolutionApi {
  constructor(
    private readonly baseUrl: string,
    private readonly fetchFn: typeof fetch = fetch,
  ) {}

  // --- Deny patterns (complete engine API) ---
  async getDenyPatterns(caseId: string, tenancyId: string): Promise<DenyPatternView>
  async addDenyPattern(caseId: string, tenancyId: string, pattern: string): Promise<void>
  async removeDenyPattern(caseId: string, tenancyId: string, pattern: string): Promise<void>

  // --- Watch patterns (getWatchPatterns pending engine issue) ---
  async getWatchPatterns(caseId: string, tenancyId: string): Promise<WatchPattern[]>
  async addWatchPattern(caseId: string, tenancyId: string, input: WatchPatternInput): Promise<void>
  async removeWatchPattern(caseId: string, tenancyId: string, patternId: string): Promise<void>

  // --- Gate policy ---
  async getGatePolicy(caseId: string, tenancyId: string): Promise<GatePolicy>
  async setGatePolicy(caseId: string, tenancyId: string, policy: GatePolicy): Promise<void>

  // --- Domain metadata (pending engine issue) ---
  async getStages(caseId: string): Promise<StageDescriptor[]>
  async getCategories(caseId: string): Promise<CategoryDescriptor[]>

  // --- State / streams / inbox ---
  async getEvolutionState(caseId: string, tenancyId: string): Promise<EvolutionStateSnapshot>
  async getStreamProgress(caseId: string, tenancyId: string): Promise<ImprovementStreamView[]>
  async getInbox(caseId: string, tenancyId: string): Promise<ConductorInboxEntry[]>
  async resolveGate(caseId: string, tenancyId: string, entryId: string,
      outcome: GateOutcome, reason?: string, feedback?: string): Promise<void>

  // --- Operational actions ---
  async pauseCategory(caseId: string, tenancyId: string,
      category: string, durationMinutes: number): Promise<void>
  async unpauseCategory(caseId: string, tenancyId: string, category: string): Promise<void>
  async blockImprovement(caseId: string, tenancyId: string,
      improvementId: string, blockedBy: string): Promise<void>
  async unblockImprovement(caseId: string, tenancyId: string, improvementId: string): Promise<void>
  async resetCircuitBreaker(caseId: string): Promise<void>
}

export interface WatchPatternInput {
  readonly category?: string;
  readonly areaId?: string;
  readonly targetPattern?: string;
  readonly minEstimatedSize?: number;
}
```

The `@PlatformQuery`/`@PlatformMutation` annotations on `EvolutionMcpAdapter` auto-generate REST endpoints at `/api/engine/evolution/*`. The `EvolutionApi` class targets these. All 19 engine methods are covered — the remaining engine methods (`getTickHistory`, `getSummary`, `getReadinessReport`, `getResearchCorpus`, `getArtifactTrail`, `triggerReadinessValidation`) are deferred until the workbench needs them (tick history is consumed indirectly via `blocks-timeline`, the others are future tabs).

## 4. Shared Events (`events.ts`)

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

Follows `emitNotificationEvent` precedent — generic parameter preserves type safety, `EventTarget` is more general than `HTMLElement`, payload is always present. Events use `pages-event` CustomEvent — no direct component-to-component coupling. The workbench listens for change events to refresh its summary bar.

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
- **Inline mode:** Renders from `patterns` property. Component does NOT optimistically update — it emits `evolution:deny-pattern-changed` with `{ action: 'add' | 'remove', pattern: string }` payload. Parent updates the `patterns` property; component re-renders on property change. This is unidirectional data flow — the component never mutates its input.
- After each mutation (both modes): emits `evolution:deny-pattern-changed`.

### Deferred Requirements

**Pattern testing/preview against sample data** (from #175): Deferred to a follow-up issue. Requires the component to know the current improvement pipeline's target set to test patterns against — depends on engine endpoint `getStreamProgress`. Will be added as an optional "Test pattern" action that shows matching improvement streams before committing.

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

Same dual-data pattern as deny-pattern-editor. Inline mode emits `evolution:watch-pattern-changed` with `{ action: 'add' | 'remove', input?: WatchPatternInput, patternId?: string }`. Parent assigns IDs and updates the `patterns` property — the component never generates IDs itself.

### Deferred Requirements

**Pattern priority ordering** (from #176): Deferred. Engine's `WatchPatternStore` has no ordering field — watch patterns are evaluated as a flat set. Priority ordering requires engine-side support (an `ordinal` field on `WatchPattern`). File as engine issue.

**Notification channel configuration per pattern** (from #176): Deferred. The engine's `WatchPattern` record has no channel fields — notification routing is handled by the `EscalationProvider` SPI and `EscalationPolicy`, not per-pattern. Channel configuration belongs in the notification-inbox's channel-preferences editor, not here.

**Escalation thresholds** (from #176): The `minEstimatedSize` field on `WatchPattern` serves as a threshold filter. Additional threshold types (health score threshold, confidence threshold) would require engine-side changes to `WatchPattern`. Deferred.

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

- **Endpoint mode:** Fetches stages via `getStages(caseId)` and current policy via `getGatePolicy(caseId, tenancyId)`. On save, calls `setGatePolicy()`.
- **Inline mode:** Renders from `stages` and `policy` properties. On save, emits `evolution:gate-policy-changed` with the new `GatePolicy` object.
- Changes tracked via local `_pendingModes: Map<string, GateMode>`. Dirty state tracked — Save button disabled until changes exist.

### Deferred Requirements

**Policy preview showing impact** (from #177): Deferred. Requires knowing active improvement streams and their stages to show "these 3 improvements would be auto-approved under this policy." Depends on `getStreamProgress` integration. File as follow-up issue.

### ARIA

- Host: `role="form"`, `aria-label="Gate policy editor"`
- Domain groups: `role="group"`, `aria-label` with domain name
- Mode selectors: native `<select>` elements (inherent ARIA)
- Save button: `aria-disabled` when no changes

## 8. evolution-workbench (`<blocks-evolution-workbench>`)

**Element tag:** `blocks-evolution-workbench`

### Component Reuse Assessment

The epic plans reusing existing components. Here's the assessment:

| Conductor concept | Epic's component | Decision | Rationale |
|---|---|---|---|
| Health scores | `kpi-metric-row` | **Reuse** | Direct fit — configure with evolution metrics |
| Tick history | `blocks-timeline` | **Reuse** | High fit — use `event-chronology` strategy with evolution event formatting |
| Gate decisions | `approval-gate` | **Defer** | Over-featured for conductor inbox (quorum, evidence slots not needed). Conductor gate resolution is approve/reject with optional reason. Use pages-table with resolve action column for now. Reassess when the inbox needs structured evidence workflows. |
| Pending inbox | `notification-inbox` | **Not reused** | Different data shape — conductor inbox entries have stage, confidence, escalation triggers. Notification inbox is optimised for notification channels, not gate decisions. |
| Compliance levels | `compliance-summary` | **Reuse** | Direct fit — add as a workbench tab showing area compliance levels from `EvolutionStateSnapshot.areaComplianceLevels` |
| Improvement streams | `work-item-inbox` | **Not reused** | Different columns — improvement streams have stage, blockedBy, conflictBlocked. Work items have priority, SLA, queue. Custom pages-table is more appropriate. |
| Audit trail | `audit-trail-viewer` | **Defer** | Requires evolution-specific audit events (tick traces, improvement lifecycle). File as follow-up issue — add an Audit tab consuming tick history and improvement case events. |
| Confidence scores | `trust-score-panel` | **Defer** | Partial fit for capability health breakdown. File as follow-up issue — add a Health Detail tab with trust-score-panel showing per-capability scores. |

### Props

```typescript
export interface EvolutionWorkbenchProps {
  endpoint?: string;
  caseId?: string;
  tenancyId?: string;
  tabs?: readonly TabDefinition[];     // domain-extensible extra tabs
  pushUrl?: string;                    // EventStreamController SSE endpoint
  pushTopics?: readonly string[];      // SSE topic filter
  // Inline data mode (all optional):
  state?: EvolutionStateSnapshot;
  streams?: readonly ImprovementStreamView[];
  inbox?: readonly ConductorInboxEntry[];
  denyPatterns?: DenyPatternView;
  watchPatterns?: readonly WatchPattern[];
  stages?: readonly StageDescriptor[];
  categories?: readonly CategoryDescriptor[];
  gatePolicy?: GatePolicy;
}
```

### detail-pane standalone mode

The current `detail-pane` requires a selection event to render tabs (shows empty state when `_item` is null). The evolution workbench is a standalone dashboard with no master-detail flow.

**Required enhancement to detail-pane:** Add a `standalone` boolean property. When `standalone` is true, skip the `_item` check and always render tabs. The `_item` defaults to `{}` internally to satisfy badge callbacks. This is a one-line change to the render guard — the tab rendering, keyboard navigation, ARIA, and badge support are all reused.

This enhancement benefits the platform — any future dashboard-style component can use detail-pane in standalone mode.

### Layout

**Summary bar** (always visible): `kpi-metric-row` with 5 metrics computed from `EvolutionStateSnapshot`:

| Metric | Source field | Display |
|--------|-------------|---------|
| Health | `healthScore` | Percentage with sparkline (TrendSourceMixin) |
| Active | `activeImprovementCount` | Count |
| Inbox | `pendingInboxCount` | Count with badge highlight when > 0 |
| Circuit breaker | `circuitBreakerState` | StatusBadge + reset action button |
| Enabled | `evolutionEnabled` | On/Off indicator |

The circuit breaker metric includes a reset action (calls `resetCircuitBreaker()` in endpoint mode) to handle the most common operational action inline.

**Tabbed content** (below summary bar) using `detail-pane` with `standalone` mode:

| Tab | Badge | Content |
|-----|-------|---------|
| Timeline | — | `blocks-timeline` with `event-chronology` strategy, formatting evolution events (tick, proposal, gate decision, health change, circuit breaker transition) via column renderers |
| Streams | active count | pages-table with `ImprovementStreamView` data. Columns: category, target, stage, blocked status. Row actions: block/unblock improvement. |
| Inbox | pending count | pages-table with `ConductorInboxEntry` data. Columns: stage, category, summary, confidence, escalation triggers, status, queued time, remaining timeout. Row actions: approve/reject with reason dialog. |
| Compliance | — | `compliance-summary` showing `areaComplianceLevels` from state snapshot |
| Configuration | — | Three editors stacked vertically with section headers |

Domain-extensible tabs via the `tabs` property — consuming apps add `TabDefinition` entries for domain-specific views (code-evolution CI panel, trading risk panel, etc.).

### Real-time Update Strategy

The evolution conductor runs ticks continuously. The workbench uses three mechanisms to stay current:

1. **Summary bar:** `kpi-metric-row` with optional `pushUrl` — supports EventStreamController push and polling fallback (`refreshInterval`). Receives SSE events with updated `EvolutionStateSnapshot` on each tick.
2. **Inbox tab:** EventStreamController for new gate entries (same pattern as notification-inbox). New PENDING entries trigger a badge increment.
3. **Streams tab:** Polls via `refreshInterval` or receives SSE push. Stream state changes (stage transitions, blocking) update inline.
4. **Editors:** Refresh after own mutations (already specified). External changes (another operator modifies deny patterns) are picked up on tab switch (lazy re-fetch) — real-time push for configuration data is not needed.

### Three consumption tiers

1. **Standalone** — set `endpoint`, `caseId`, `tenancyId`. Workbench fetches all data from REST API. Optionally set `pushUrl` for real-time updates.
2. **Panel-hosted** — registered via `registerPanel('evolution-workbench', 'blocks-evolution-workbench')`. The host app calls `configure({ endpoint, caseId, tenancyId })` before mount. Case context is extracted from the host's panel registration, not from selection events.
3. **Inline data** — all data passed as properties. No network calls. Used by the sample page (#179).

All tiers expose a `configure()` method:

```typescript
configure(props: Partial<EvolutionWorkbenchProps>): void {
  if (props.endpoint !== undefined) this.endpoint = props.endpoint;
  if (props.caseId !== undefined) this.caseId = props.caseId;
  if (props.tenancyId !== undefined) this.tenancyId = props.tenancyId;
  if (props.tabs !== undefined) this.tabs = props.tabs;
  // ... all other props
}
```

### Event handling

Listens for `evolution:deny-pattern-changed`, `evolution:watch-pattern-changed`, `evolution:gate-policy-changed`, and `evolution:gate-resolved` events from child editors. On any change event, refreshes the summary bar metrics by re-fetching `getEvolutionState()`.

### ARIA

- Host: `role="region"`, `aria-label="Evolution workbench"`
- Summary bar: via kpi-metric-row's built-in ARIA
- Tabs: via detail-pane's built-in ARIA (tablist/tab/tabpanel)

## 9. Sample Page (#179)

Static HTML page at `components/evolution-workbench/sample/index.html` that instantiates `<blocks-evolution-workbench>` in inline data mode with representative sample data. Serves as:

- Development aid for visual verification
- Documentation reference for consuming apps — demonstrates domain extension points
- Regression test baseline

Sample data in `sample-data.ts` covers: full `EvolutionStateSnapshot` with component scores, 3 active streams (one blocked, one at a gate checkpoint), 4 inbox entries (pending + approved + timed_out + auto_approved), structural + dynamic deny patterns, 3 watch patterns, code-evolution stages with a mixed gate policy, area compliance levels.

**Domain extension demo:** The sample page includes a custom `TabDefinition` showing how a consuming app adds a domain-specific tab:

```typescript
const tradingTab: TabDefinition = {
  id: 'trading-risk',
  label: 'Trading Risk',
  order: 50,
  elementTag: 'sample-trading-risk-panel',
};
```

A minimal `<sample-trading-risk-panel>` element in the sample directory renders mock trading risk metrics, demonstrating the extension API.

## 10. Engine Issues to File

API gaps requiring engine-side additions:

| Issue | What | Why |
|-------|------|-----|
| `getWatchPatterns(caseId, tenancyId)` | Expose `WatchPatternStore.findActive()` via `EngineEvolutionApi` and `EvolutionMcpAdapter` | watch-pattern-editor needs to list existing patterns |
| `getStages(caseId)` | Query `ImprovementCategoryRegistry.stagesForDomain()` for all registered domains | gate-policy-editor needs domain-contributed stage metadata |
| `getCategories(caseId)` | Query `ImprovementCategoryRegistry.allCategories()` | watch-pattern-editor category dropdown, workbench category display |
| `getGatePolicy(caseId, tenancyId)` | Read current gate policy from case configuration | gate-policy-editor needs current policy to pre-populate |

All four are thin adapter methods — the underlying data is already available internally. `@PlatformQuery` on the MCP adapter generates the REST endpoints automatically.

## 11. Deferred Items (file as issues)

| Item | Source | Rationale | File against |
|------|--------|-----------|-------------|
| Pattern testing/preview | #175 | Needs `getStreamProgress` to test patterns against active streams | blocks-ui |
| Policy preview (impact analysis) | #177 | Needs `getStreamProgress` to show which improvements would be affected | blocks-ui |
| Watch pattern priority ordering | #176 | Engine's `WatchPattern` has no ordinal field | engine |
| Watch pattern escalation thresholds | #176 | Engine's `WatchPattern` has limited threshold fields | engine |
| Audit trail tab | #174 reuse plan | Needs evolution-specific audit event formatting | blocks-ui |
| Health detail tab (trust-score-panel) | #174 reuse plan | Needs per-capability health score data integration | blocks-ui |
| Responsive layout | #178 | Breakpoints, collapse points, mobile layout | blocks-ui |

## 12. Test Strategy

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

Workbench additionally: tab navigation, summary bar metrics, event-driven refresh, domain-extensible tabs, three consumption tiers, `configure()` method, panel-hosted registration.

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

- `EngineEvolutionApi.java` — engine API surface (19 methods: deny patterns, watch patterns, gate policy, state, streams, inbox, operational actions)
- `EvolutionMcpAdapter.java` — REST exposure via `@PlatformQuery`/`@PlatformMutation`
- `DenyPatternView.java` — static + dynamic deny pattern view record
- `WatchPattern.java` — watch pattern record (id, category, areaId, targetPattern, minEstimatedSize, createdAt)
- `GatePolicy.java:21-40` — gate policy record with `Map<String, GateMode>`
- `StageDescriptor.java` — domain-contributed stage metadata (id, name, ordinal, gateCheckpoint, domainId)
- `CategoryDescriptor.java` — domain-contributed category metadata
- `ImprovementConfig.java` — case configuration including gatePolicy
- `ConductorInboxEntry.java` — 14-field inbox entry (caseId, id, stage, status, category, areaId, improvementCaseId, summary, escalationTriggers, confidence, queuedAt, resolvedAt, timeoutMinutes, decision)
- `ConductorInboxEntry.Status` — 6-value enum (PENDING, APPROVED, REJECTED, REDIRECTED, TIMED_OUT, AUTO_APPROVED)
- `EscalationTrigger.java` — layer (CATEGORY_RULE, WATCH_PATTERN, CONFIDENCE_SCORE) + reason
- `ConductorDecision.java` — outcome, reason, feedback
- `WatchPatternStore.java` — internal store with `findActive()` (not exposed via API)
- `EvolutionStateSnapshot.java` — full state view (healthScore, componentScores, healthDelta, circuitBreakerState, categoryStates, activeStreams, complianceLevels, etc.)
- `components/detail-pane/src/detail-pane.ts:176-179` — item guard that needs standalone mode enhancement
- `components/notification-inbox/src/` — precedent for multi-element package with shared api.ts and types.ts
- `components/notification-inbox/src/events.ts` — emitNotificationEvent precedent (generic EventTarget, typed payload)
- `components/preferences-editor/src/api.ts` — API class pattern with injectable fetchFn
- `components/notification-inbox/src/mute-list.ts` — CRUD table + inline add form + delete confirmation precedent
- PP-20260713-8ea1af — component customisation via typed config + render callbacks + factory overrides
- D9 engine decision — blocks-ui provides composable primitives; apps compose domain workbench
- D1–D5 in `decisions.md`
