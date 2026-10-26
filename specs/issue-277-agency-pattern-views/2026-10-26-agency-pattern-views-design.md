# Agency Pattern Views — Full YAML-to-UI Coverage

Epic: casehubio/casehub-mosaic#277

## Vision

Every agency capability — orchestration, deliberation, perception, communication, governance, optimisation, social — gets a dedicated visualization component that renders its PatternCategory YAML spec. The type system mirrors the 7-category hierarchy from casehubio/blocks#337 (agency repo restructure). Playbook pattern specs are handled by a thin spec-descriptor view; the existing coordination-pipeline-view renders rich runtime pipeline data separately. Together this gives full UI coverage for building agentic applications from YAML definitions.

## Architecture

### Dispatch Model (D1)

Agent-viewer dispatches pattern views via a switch statement on `PatternCategory`'s category+type discriminant:

```typescript
switch (spec.category) {
  case 'orchestration':
    switch (spec.type) {
      case 'supervisor': return html`<agency-orchestration-supervisor-view .spec=${spec}></...>`;
      case 'choreography': return html`<agency-orchestration-choreography-view .spec=${spec}></...>`;
      case 'saga': return html`<agency-orchestration-saga-view .spec=${spec}></...>`;
      case 'decomposition': return html`<agency-orchestration-decomposition-view .spec=${spec}></...>`;
      default: const _o: never = spec.type; return _o;
    }
  case 'deliberation':
    switch (spec.type) {
      case 'voting': return html`<agency-deliberation-voting-view .spec=${spec}></...>`;
      case 'debate': return html`<agency-deliberation-debate-view .spec=${spec}></...>`;
      case 'negotiation': return html`<agency-deliberation-negotiation-view .spec=${spec}></...>`;
      case 'coalition': return html`<agency-deliberation-coalition-view .spec=${spec}></...>`;
      case 'normative': return html`<agency-deliberation-normative-view .spec=${spec}></...>`;
      default: const _d: never = spec.type; return _d;
    }
  case 'perception':
    switch (spec.type) {
      case 'summarisation': return html`<agency-perception-summarisation-view .spec=${spec}></...>`;
      case 'narrative': return html`<agency-perception-narrative-view .spec=${spec}></...>`;
      case 'observation': return html`<agency-perception-observation-view .spec=${spec}></...>`;
      default: const _p: never = spec.type; return _p;
    }
  case 'communication':
    switch (spec.type) {
      case 'channel-execution': return html`<agency-base-pattern-view .spec=${spec}></...>`;
      case 'channel-summary': return html`<agency-base-pattern-view .spec=${spec}></...>`;
      default: const _c: never = spec.type; return _c;
    }
  case 'governance':
    switch (spec.type) {
      case 'judgment': return html`<agency-governance-judgment-view .spec=${spec}></...>`;
      case 'oversight': return html`<agency-base-pattern-view .spec=${spec}></...>`;
      case 'attestation': return html`<agency-base-pattern-view .spec=${spec}></...>`;
      case 'trust': return html`<agency-base-pattern-view .spec=${spec}></...>`;
      default: const _g: never = spec.type; return _g;
    }
  case 'optimisation':
    switch (spec.type) {
      case 'prompt-optimisation': return html`<agency-optimisation-prompt-view .spec=${spec}></...>`;
      default: const _x: never = spec.type; return _x;
    }
  case 'social':
    switch (spec.type) {
      case 'memory-hygiene': return html`<agency-social-memory-hygiene-view .spec=${spec}></...>`;
      case 'cognition': return html`<agency-social-cognition-view .spec=${spec}></...>`;
      case 'reflection': return html`<agency-base-pattern-view .spec=${spec}></...>`;
      default: const _s: never = spec.type; return _s;
    }
  case 'playbook':
    return html`<agency-playbook-spec-view .spec=${spec}></agency-playbook-spec-view>`;
  default: const _: never = spec; return _;
}
```

PatternCategory is a closed discriminated union of 8 interfaces (7 agency + playbook). The `default: never` guards give true compile-time exhaustiveness — adding a new type to the union causes a type error at the unhandled default. No runtime registry needed.

**Playbook dispatch note:** `PlaybookPatternCategory` is a thin spec descriptor (`{category, type, name?, description?}`). It is structurally incompatible with `CoordinationPipeline` (which has `phases: PhaseItem[]` and is the data type for `coordination-pipeline-view`). The playbook case dispatches to `agency-playbook-spec-view`, which renders the spec descriptor fields (playbook type label, name, description). The rich pipeline visualization (`coordination-pipeline-view`) is a separate concern — it renders runtime `CoordinationPipeline` data accessed through the Steps tab, not through the Pattern dispatch.

### Component Placement (D3)

New pattern views live as subdirectories within `packages/agent-viewer/`:

```
packages/agent-viewer/src/
├── shared-styles.ts         # patternViewStyles + PatternViewMixin
├── views/
│   ├── base-pattern-view.ts         # shared view for types with no category-specific fields
│   ├── playbook-spec-view.ts        # thin PlaybookPatternCategory renderer
│   ├── orchestration/
│   │   ├── supervisor-view.ts
│   │   ├── choreography-view.ts
│   │   ├── saga-view.ts
│   │   ├── decomposition-view.ts
│   │   └── routing-detail.ts        # sub-component for routing data
│   ├── deliberation/
│   │   ├── voting-view.ts           # migrated from src/voting-view.ts
│   │   ├── debate-view.ts           # migrated from src/debate-view.ts
│   │   ├── negotiation-view.ts
│   │   ├── coalition-view.ts
│   │   └── normative-view.ts
│   ├── perception/
│   │   ├── observation-view.ts
│   │   ├── narrative-view.ts
│   │   ├── summarisation-view.ts
│   │   └── summarisation-funnel.ts  # sub-component of summarisation-view
│   ├── governance/
│   │   └── judgment-view.ts
│   ├── optimisation/
│   │   └── prompt-optimisation-view.ts
│   └── social/
│       ├── memory-hygiene-view.ts
│       └── cognition-view.ts
├── coordination-pipeline-view.ts    # existing — renders CoordinationPipeline runtime data
├── agent-viewer.ts                  # dispatch container
└── ...
```

Standalone workbenches that compose multiple views stay in `components/`:
- `components/conversation-viewer/` — already a standalone composition
- `components/cognitive-workbench/` — greenfield cognitive visualization (#231)

### Naming Convention (D6)

Custom element tags: `agency-<category>-<type>-view`

Examples:
- `agency-orchestration-supervisor-view`
- `agency-deliberation-voting-view`
- `agency-governance-judgment-view`
- `agency-perception-observation-view`

Existing views are renamed to follow the convention as part of Batch 0:
- `voting-view` → `agency-deliberation-voting-view`
- `debate-view` → `agency-deliberation-debate-view`

Standalone components (cognitive-workbench, work-item-hitl) are NOT pattern views and do NOT follow this convention — they are exempt from D1 dispatch.

### Two Visualization Families (D7)

| Family | Data type | Purpose | Existing examples |
|--------|-----------|---------|-------------------|
| Playbook (steps) | CoordinationPipeline | Step sequencing — phases, barriers, swim lanes | coordination-pipeline-view |
| Agency (patterns) | PatternCategory variants | Multi-agent coordination — routing, deliberation, perception | agency-deliberation-voting-view, agency-deliberation-debate-view |

These coexist in agent-viewer. Playbook views render how steps execute. Agency views render how agents coordinate. Different data, different purpose.

### Shared Infrastructure (D5)

**`patternViewStyles`** — shared CSS export from `packages/agent-viewer/src/shared-styles.ts`:

```typescript
export const patternViewStyles = css`
  :host {
    display: block;
    background: rgba(255, 255, 255, 0.02);
    border: 1px solid rgba(255, 255, 255, 0.08);
    border-radius: 8px;
    overflow: hidden;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
    color: #cdd6f4;
  }
`;
```

**`PatternViewMixin`** — thin Lit mixin in `packages/agent-viewer/src/shared-styles.ts`:

```typescript
export const PatternViewMixin = <T extends Constructor<LitElement>>(Base: T) =>
  class extends Base {
    @property({ attribute: false }) spec!: PatternCategory;

    override connectedCallback() {
      super.connectedCallback();
      this.setAttribute('role', 'region');
      this.setAttribute('aria-label', `${this.spec?.category ?? 'pattern'} view`);
    }

    // Update aria-busy based on loading state
    protected setLoading(loading: boolean) {
      this.setAttribute('aria-busy', String(loading));
    }

    protected renderEmptyState(message = 'No pattern data') {
      return html`<div class="empty-state">${message}</div>`;
    }
  };
```

Views use: `class SupervisorView extends PatternViewMixin(LitElement) { ... }`

Views needing live data additionally compose with DataSourceMixin: `class SummarisationView extends PatternViewMixin(DataSourceMixin(LitElement)) { ... }`

**Property migration:** The mixin standardises on `spec` as the property name for the PatternCategory data. Existing views (voting-view, debate-view) currently use `pattern`. Batch 0 renames `pattern` → `spec` in these views and updates the agent-viewer dispatch callsite (`.pattern=${p}` → `.spec=${p}`). This is a breaking change for any external callers — acceptable since this platform has no end users.

### Data Source Strategy (D8)

| View type | Data delivery | Rationale |
|-----------|---------------|-----------|
| Simple pattern views | Property-only (`spec` via PatternViewMixin) | Renders static YAML spec shape — hosting context fetches |
| Execution monitors | Dual (spec property + EventStreamController) | Live execution state pushes via SSE |
| Topology/infrastructure views | Dual (DataSourceMixin endpoint + property) | Graph data may come from endpoint or inline |
| Project dashboard | DataSourceMixin endpoint | Aggregates multiple data sources |

## Per-View Specifications

### Orchestration Category

**#267 — Supervisor View (`agency-orchestration-supervisor-view`)**
Hub-and-spoke agent routing visualization. Central supervisor node connected to worker agents. Shows routing spec (first-match/round-robin/sequential/llm-selected/select-all), termination conditions, failure policy. Worker agent cards around the hub. Property-only data.

**#244 — Choreography View (`agency-orchestration-choreography-view`)**
Reactive blackboard architecture. Shared state (blackboard) in the center, agents observing and writing. Event-driven activation visualization — which agents respond to which state changes. Activation spec badges. Property-only data.

**#254 — Saga View (`agency-orchestration-saga-view`)**
Forward path + compensation chain. Two-column layout: left shows the forward execution path (steps with agent assignments), right shows compensation steps in reverse order. Step status badges, compensation triggers. Property-only data.

**#255 — Decomposition View (`agency-orchestration-decomposition-view`)**
YAML case definition drill-down showing task decomposition strategy. Reuses pages-graph-canvas + ELK layout (like agency-dag-viewer). Shows decomposition strategy, agent bindings, workers, milestones with trust score pills and adaptive decision badges. Dual data mode (endpoint for case definition YAML, property for inline).

**#276 — Routing Detail (sub-component, not dispatched)**
Agent selection decision visualization, composed within supervisor-view and other orchestration views that have `routing` data. Shows the routing decision tree: guard conditions, match outcomes, selected agents with capability badges. This is NOT a pattern type — `routing` is a property on `OrchestrationPatternCategory`, not a value of `OrchestrationPatternType`. Rendered inline within parent views, not through D1 dispatch.

### Governance Category

`GovernancePatternCategory` extends `BasePatternCategory` with `type: 'judgment' | 'oversight' | 'attestation' | 'trust'` and no additional fields. Note: the `JudgmentSpec` (triggers, callers, agreement) is a field on `DeliberationPatternCategory`, not on `GovernancePatternCategory` — the governance `judgment` type and the deliberation `judgment` field are distinct concepts.

**#268 — Judgment View (`agency-governance-judgment-view`)**
Governance judgment policy visualization. Shows the judgment type label, assigned agents with role badges, failure policy configuration. Agent card grid with role assignment. Property-only data.

**Oversight, Attestation, Trust** → rendered by `agency-base-pattern-view` (see §Base Pattern View below).

### Deliberation Category

**#269 — Negotiation View (`agency-deliberation-negotiation-view`)**
Proposals, acceptance policies, rounds. Shows proposal lifecycle: submitted → counter-proposed → accepted/rejected. Round progression with agent participation. Acceptance policy rules. Property-only data.

**#270 — Coalition Formation View (`agency-deliberation-coalition-view`)**
Capability coverage evaluation. Agent capability matrix on the left, coalition candidates on the right. Coverage bars showing how each coalition configuration covers the required capabilities. Property-only data.

**#271 — Normative Conflict Resolution View (`agency-deliberation-normative-view`)**
Rule arbitration strategies. Shows conflicting norms/rules, arbitration strategy selection, resolution outcome. Tree-based conflict visualization with priority levels. Property-only data.

### Communication Category

`CommunicationPatternCategory` extends `BasePatternCategory` with `type: 'channel-execution' | 'channel-summary'` and no additional fields.

**Channel Execution, Channel Summary** → rendered by `agency-base-pattern-view` (see §Base Pattern View below).

### Perception Category

**#272 — Observation/Affordance Pipeline View (`agency-perception-observation-view`)**
Perception filtering and tiered rendering. Pipeline visualization: raw observations → filter → affordance extraction → tiered output (critical/important/background). Animated flow direction. Dual data mode (spec + endpoint for live observations).

**#273 — Decision Narrative Pipeline View (`agency-perception-narrative-view`)**
Signals to narratives pipeline. Shows how raw signal events are transformed through summarisation stages into structured decision narratives. Pipeline flow with intermediate products visible. Property-only data.

**#246 — Summarisation Pipeline View (`agency-perception-summarisation-view`)**
Event stream processing pipeline. Multi-stage pipeline: event stream → chunking → LLM summarisation → structured output. Stage metrics (throughput, latency). Dual data mode (spec + EventStreamController for live pipeline metrics).

**#253 — Summarisation Funnel (sub-component of #246, not dispatched)**
Funnel visualization of summarisation compression, composed within the summarisation pipeline view. Shows input event count → intermediate summaries → final output. Compression ratio badges. Not dispatched via D1 — a single `spec.type === 'summarisation'` case renders the pipeline view, which conditionally includes the funnel sub-component.

### Optimisation Category

**#274 — Prompt Optimisation Pipeline View (`agency-optimisation-prompt-view`)**
Variant selection and scoring. Shows prompt variants with A/B comparison, scoring metrics (accuracy, latency, cost), selection history. Table with sparklines for variant performance trends. Dual data mode (spec + endpoint for scoring data).

### Social Category

**#275 — Memory Hygiene View (`agency-social-memory-hygiene-view`)**
Confidence scoring and eviction visualization. Memory entries table with confidence scores, staleness indicators, eviction policy thresholds. Sorted by confidence (low first for eviction candidates). Dual data mode (spec + endpoint for memory state).

**Cognition View (`agency-social-cognition-view`)**
Cognitive pattern visualization. Renders cognition agent assignments, cognitive processing scope, and failure policy. Agent card grid. Property-only data. The standalone cognitive workbench (#231) in `components/cognitive-workbench/` is a separate composition that builds on this view.

**Provisional (#287):** The capability catalog marks `cognition` as `hasYamlRoot: true`, indicating the Java side has a rich YAML schema. The current `SocialPatternCategory` interface has no cognition-specific fields — when those fields are derived from the Java schema, this view will need enrichment beyond the base-field layout. Tracked as casehubio/casehub-mosaic#287.

**Reflection** → rendered by `agency-base-pattern-view` (see §Base Pattern View below).

### Base Pattern View (`agency-base-pattern-view`)

Single parameterized component for capability types whose category interfaces have no fields beyond `BasePatternCategory`. Renders: type label badge, name, description, agent card grid, failure policy configuration. The type label is derived from `spec.type` — no per-type rendering logic.

Dispatched for these 6 types:
- governance: `oversight`, `attestation`, `trust`
- communication: `channel-execution`, `channel-summary`
- social: `reflection`

One component, one test file, one CSS block, one gallery page. When a type later gains category-specific fields (e.g., when `GovernancePatternCategory` is enriched), that type graduates from the base view to its own component — a one-line dispatch change. Property-only data.

### Playbook Spec View (`agency-playbook-spec-view`)

Thin renderer for `PlaybookPatternCategory` — the pattern spec descriptor for playbook types (sequence, parallel, loop, conditional, match, try-catch). Renders the playbook type label, name, and description. This is NOT the rich pipeline visualization — `PlaybookPatternCategory` is a thin descriptor (`{category, type, name?, description?}`), structurally incompatible with `CoordinationPipeline` (which has `phases: PhaseItem[]`). The rich pipeline visualization (`coordination-pipeline-view`) renders runtime `CoordinationPipeline` data accessed through the Steps tab. Property-only data.

### Standalone Components (outside D1 dispatch)

**#245 — Work Item Card HITL**
Human-in-the-loop work item card. Shows the task description, agent context, required human action, response form. Integrates with the existing work-item-detail patterns. This is NOT a communication pattern type — `CommunicationPatternCategory.type` is `'channel-execution' | 'channel-summary'`, with no `hitl` value. Lives in `components/` as a standalone work-item extension. Property-only data.

**#231 — Cognitive Workbench**
Brain graph visualization. Greenfield development in `components/cognitive-workbench/`. Uses pages-graph-canvas with brain layout (ELK). Extraction pipeline view. Composes agency-social-cognition-view. Dual data mode.

### Infrastructure (Wave 4)

These are project infrastructure visualizations, not agency pattern views. They don't use PatternCategory types or the agent-viewer dispatch. They live in `components/`.

**#247 — Desired State Graph**
Reuses pages-graph-canvas + ELK. Shows `dependsOn` + `forEach` relationships as a DAG. Status-coloured nodes from reconciliation state. Dual data mode.

**#248 — Application Topology**
Service dependency DAG (extends existing `topology-viewer` patterns). Deployment view with per-cluster status. Reuses pages-graph-canvas. Dual data mode.

**#249 — Adaptive Topology**
Situation-triggered topology mutations. Shows base topology + mutation rules + current active mutations. Builds on #248 topology-viewer. Dual data mode.

**#250 — Project Dashboard**
Aggregate KPI view. Composes kpi-metric-row, trust-score-panel (compact), and execution-monitor (compact) into a grid layout. DataSourceMixin endpoint-driven.

**#251 — Focus Mode + Completeness Signals**
UI overlay that highlights incomplete areas and guides attention. Badge-based completeness indicators per section. Property-only data.

**#252 — Desired State Topology View**
Combines #247 and #248 — desired state graph with topology overlay. Shows target state vs current state diff. Dual data mode.

## Wave Execution Plan (D2, D4)

**Dependency:** #257 (Align pattern views to real blocks PatternSpec YAML) is OPEN. Its type alignment scope was largely resolved by #283 (CLOSED) and #284 (CLOSED), which established the current type system derived from the Java schema. Batch 0 depends on the type system being stable, which it now is. The remaining scope of #257 (aligning voting-view/debate-view to real YAML shapes) is subsumed by the migration tasks below.

### Batch 0 — Foundation

| Task | Description |
|------|-------------|
| Create `patternViewStyles` in `packages/agent-viewer/src/shared-styles.ts` | Shared CSS export (agent-viewer scoped, not mosaic-core) |
| Create `PatternViewMixin` in `packages/agent-viewer/src/shared-styles.ts` | Spec property + ARIA + empty state |
| Migrate voting-view to `src/views/deliberation/voting-view.ts` | Adopt PatternViewMixin, rename `pattern` → `spec`, rename element to `agency-deliberation-voting-view` |
| Migrate debate-view to `src/views/deliberation/debate-view.ts` | Adopt PatternViewMixin, rename `pattern` → `spec`, rename element to `agency-deliberation-debate-view` |
| Update agent-viewer dispatch | Rename `.pattern=${p}` → `.spec=${p}`, update element names, add exhaustive switch over all 22 agency types + playbook |
| Create `agency-base-pattern-view` in `src/views/base-pattern-view.ts` | Shared component for 6 types with no category-specific fields |
| Create `agency-playbook-spec-view` in `src/views/playbook-spec-view.ts` | Thin PlaybookPatternCategory renderer (type label, name, description) |
| Create `src/views/` directory structure | Category subdirectories |
| Gallery scaffold (#258) | Real YAML demo data structure |

### Wave 1 — Orchestration + Governance (6 views, 6 issues)

| # | View | Complexity |
|---|------|------------|
| #267 | Supervisor (includes #276 routing detail sub-component) | M — hub-and-spoke layout, agent cards, routing detail |
| #244 | Choreography | M — blackboard + reactive agents |
| #254 | Saga | S — two-column forward/compensation |
| #255 | Decomposition (renamed from case-definition-diagram) | L — graph-canvas + ELK, trust decorations |
| #268 | Judgment | S — agent card grid, base fields |
| — | Oversight, Attestation, Trust → `agency-base-pattern-view` | — (shared component, built once in Batch 0) |

### Wave 2 — Deliberation + Communication (4 views, 5 issues)

| # | View | Complexity |
|---|------|------------|
| #269 | Negotiation | M — proposal lifecycle, rounds |
| #270 | Coalition formation | M — capability matrix, coverage bars |
| #271 | Normative conflict | S — tree-based conflict display |
| — | Channel Execution, Channel Summary → `agency-base-pattern-view` | — (shared component, built once in Batch 0) |
| #245 | Work item card HITL (standalone, `components/`) | S — task card with response form |

### Wave 3 — Perception + Optimisation + Social (7 views, 7 issues)

| # | View | Complexity |
|---|------|------------|
| #272 | Observation pipeline | M — pipeline with tiered output |
| #273 | Decision narrative | M — signal-to-narrative pipeline |
| #246 | Summarisation pipeline (includes #253 funnel sub-component) | M — multi-stage pipeline, live metrics, funnel |
| #274 | Prompt optimisation | M — variant table, scoring sparklines |
| — | Cognition (provisional — will enrich when SocialPatternCategory gains fields) | XS — base-field view + cognitive workbench hook |
| — | Reflection → `agency-base-pattern-view` | — (shared component, built once in Batch 0) |
| #275 | Memory hygiene | S — memory table with confidence scores |
| #231 | Cognitive workbench (standalone, `components/`) | L — greenfield, graph layout |

### Wave 4 — Infrastructure (6 issues)

| # | View | Complexity |
|---|------|------------|
| #247 | Desired state graph | M — DAG with dependsOn/forEach |
| #248 | Application topology | M — extends topology-viewer patterns |
| #249 | Adaptive topology | M — builds on #248 |
| #250 | Project dashboard | M — composition of existing components |
| #251 | Focus mode | S — overlay with completeness badges |
| #252 | Desired state topology | M — combines #247 + #248 |

## Testing Strategy

Each view gets:
1. **Unit test** — renders with spec data, verifies DOM structure
2. **ARIA assertions** — role, aria-label present (per CLAUDE.md mandate)
3. **Empty state test** — renders gracefully with no/undefined spec
4. **Gallery page** — standalone demo with real YAML-derived data

Agent-viewer dispatch test:
- Exhaustiveness test verifying all PatternCategory types are handled
- Integration test creating agent-viewer with each category+type

Schema:
- Standalone `agency-*` components (`components/`) export a Props interface and get AgencyComponentRegistry entries (per PP-20260907)
- Pattern views (`packages/agent-viewer/src/views/`) are internal sub-components of agent-viewer and exempt from individual registry entries — they are not independently consumed by host applications
- `yarn workspace @casehubio/mosaic-schema run generate` after each wave for standalone components

## Protocols Checklist

| Protocol | Applies to | Action |
|----------|-----------|--------|
| PP-20260713 (customisation pattern) | Views with render callbacks | Typed config + callbacks, inline styles in callback output |
| PP-20260930 (event namespace) | Views emitting bubbling events | `<component-prefix>:<action>` naming |
| PP-20260907 (registry props) | Standalone agency-* elements in `components/` | Export Props interface, add to AgencyComponentRegistry. Pattern views in `packages/agent-viewer/src/views/` are internal sub-components and exempt — they are consumed only through agent-viewer's dispatch, not independently by host applications |

## References

- `packages/mosaic-core/src/types/patterns/` — PatternCategory type system
- `packages/mosaic-core/src/types/orchestration.ts` — YAML spec types and runtime types
- `packages/agent-viewer/src/agent-viewer.ts:262-278` — current dispatch logic
- `packages/agent-viewer/src/views/deliberation/voting-view.ts` — existing pattern view reference (post Batch 0 migration)
- `packages/agent-viewer/src/coordination-pipeline-view.ts` — playbook visualization (CoordinationPipeline type)
- `components/conversation-viewer/` — standalone deliberation workbench
- `packages/mosaic-core/src/types/patterns/catalog-staleness.test.ts` — catalog/TS type parity enforcement
- casehubio/blocks#337 — Java agency repo restructure (7-category hierarchy)
- `docs/protocols/mosaic/component-customisation-pattern.md` — PP-20260713
- `docs/protocols/mosaic/custom-event-namespace-prefix.md` — PP-20260930
- `docs/protocols/mosaic/component-registry-props.md` — PP-20260907
