# Agency Pattern Views — Full YAML-to-UI Coverage

Epic: casehubio/casehub-mosaic#277

## Vision

Every agency capability — orchestration, deliberation, perception, communication, governance, optimisation, social — gets a dedicated visualization component that renders its PatternCategory YAML spec. The type system mirrors the 7-category hierarchy from casehubio/blocks#337 (agency repo restructure). Together with the existing playbook visualization (coordination-pipeline-view), this gives full UI coverage for building agentic applications from YAML definitions.

## Architecture

### Dispatch Model (D1)

Agent-viewer dispatches pattern views via a switch statement on `PatternCategory`'s category+type discriminant:

```typescript
switch (spec.category) {
  case 'orchestration':
    switch (spec.type) {
      case 'supervisor': return html`<agency-orchestration-supervisor-view .spec=${spec}></...>`;
      case 'choreography': return html`<agency-orchestration-choreography-view .spec=${spec}></...>`;
      // ...
    }
  case 'deliberation':
    switch (spec.type) {
      case 'voting': return html`<agency-voting-view .spec=${spec}></...>`;
      // ...
    }
  // exhaustive over all categories
}
```

PatternCategory is a closed discriminated union. The switch gives compile-time exhaustiveness — adding a new type to the union forces updating the dispatch. No runtime registry needed.

### Component Placement (D3)

New pattern views live as subdirectories within `packages/agent-viewer/`:

```
packages/agent-viewer/src/
├── views/
│   ├── orchestration/
│   │   ├── supervisor-view.ts
│   │   ├── choreography-view.ts
│   │   ├── saga-view.ts
│   │   └── routing-strategy-view.ts
│   ├── deliberation/
│   │   ├── negotiation-view.ts
│   │   ├── coalition-view.ts
│   │   └── normative-view.ts
│   ├── perception/
│   │   ├── observation-view.ts
│   │   ├── narrative-view.ts
│   │   └── summarisation-view.ts
│   ├── communication/
│   │   └── channel-execution-view.ts
│   ├── governance/
│   │   └── judgment-view.ts
│   ├── optimisation/
│   │   └── prompt-optimisation-view.ts
│   └── social/
│       ├── memory-hygiene-view.ts
│       └── cognition-view.ts
├── voting-view.ts           # existing — stays in place
├── debate-view.ts           # existing — stays in place
├── coordination-pipeline-view.ts  # existing — playbook, not agency
├── agent-viewer.ts          # dispatch container
└── ...
```

Standalone workbenches that compose multiple views stay in `components/`:
- `components/conversation-viewer/` — already a standalone composition
- `components/cognitive-workbench/` — promoted from `examples/`, composes cognition views

### Naming Convention (D6)

Custom element tags: `agency-<category>-<type>-view`

Examples:
- `agency-orchestration-supervisor-view`
- `agency-deliberation-negotiation-view`
- `agency-governance-judgment-view`
- `agency-perception-observation-view`

Existing views (voting-view, debate-view) keep their current names to avoid breaking consumers — they predate the category-aware naming.

### Two Visualization Families (D7)

| Family | Data type | Purpose | Existing examples |
|--------|-----------|---------|-------------------|
| Playbook (steps) | CoordinationPipeline | Step sequencing — phases, barriers, swim lanes | coordination-pipeline-view |
| Agency (patterns) | PatternCategory variants | Multi-agent coordination — routing, deliberation, perception | voting-view, debate-view |

These coexist in agent-viewer. Playbook views render how steps execute. Agency views render how agents coordinate. Different data, different purpose.

### Shared Infrastructure (D5)

**`patternViewStyles`** — shared CSS export from mosaic-core:

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

**`PatternViewMixin`** — thin Lit mixin in agent-viewer:

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

**#276 — Routing Strategy View (`agency-orchestration-routing-view`)**
Agent selection decision visualization. Shows the routing decision tree: guard conditions, match outcomes, selected agents with capability badges. Tabular view of routing history (if execution data available). Property-only for spec, optional endpoint for routing history.

**#244 — Choreography View (`agency-orchestration-choreography-view`)**
Reactive blackboard architecture. Shared state (blackboard) in the center, agents observing and writing. Event-driven activation visualization — which agents respond to which state changes. Activation spec badges. Property-only data.

**#254 — Saga View (`agency-orchestration-saga-view`)**
Forward path + compensation chain. Two-column layout: left shows the forward execution path (steps with agent assignments), right shows compensation steps in reverse order. Step status badges, compensation triggers. Property-only data.

**#255 — Case Definition Diagram (`agency-case-definition-diagram`)**
YAML case definition drill-down through all execution models. Reuses pages-graph-canvas + ELK layout (like agency-dag-viewer). Shows bindings, workers, milestones with trust score pills and adaptive decision badges. Dual data mode (endpoint for case definition YAML, property for inline).

### Governance Category

**#268 — Judgment View (`agency-governance-judgment-view`)**
Triggers, callers, escalation chains, agreement visualization. Three-section layout: trigger config (alwaysYield/neverYield/iterationBased/confidenceThreshold), caller topology (single/fanOut/escalationChain with agent cards), agreement spec (unanimous/majority/threshold). Property-only data.

### Deliberation Category

**#269 — Negotiation View (`agency-deliberation-negotiation-view`)**
Proposals, acceptance policies, rounds. Shows proposal lifecycle: submitted → counter-proposed → accepted/rejected. Round progression with agent participation. Acceptance policy rules. Property-only data.

**#270 — Coalition Formation View (`agency-deliberation-coalition-view`)**
Capability coverage evaluation. Agent capability matrix on the left, coalition candidates on the right. Coverage bars showing how each coalition configuration covers the required capabilities. Property-only data.

**#271 — Normative Conflict Resolution View (`agency-deliberation-normative-view`)**
Rule arbitration strategies. Shows conflicting norms/rules, arbitration strategy selection, resolution outcome. Tree-based conflict visualization with priority levels. Property-only data.

### Communication Category

**#245 — Work Item Card HITL (`agency-communication-hitl-view`)**
Human-in-the-loop work item card. Shows the task description, agent context, required human action, response form. Integrates with the existing work-item-detail patterns. Property-only data.

### Perception Category

**#272 — Observation/Affordance Pipeline View (`agency-perception-observation-view`)**
Perception filtering and tiered rendering. Pipeline visualization: raw observations → filter → affordance extraction → tiered output (critical/important/background). Animated flow direction. Dual data mode (spec + endpoint for live observations).

**#273 — Decision Narrative Pipeline View (`agency-perception-narrative-view`)**
Signals to narratives pipeline. Shows how raw signal events are transformed through summarisation stages into structured decision narratives. Pipeline flow with intermediate products visible. Property-only data.

**#246 — Summarisation Pipeline View (`agency-perception-summarisation-view`)**
Event stream processing pipeline. Multi-stage pipeline: event stream → chunking → LLM summarisation → structured output. Stage metrics (throughput, latency). Dual data mode (spec + EventStreamController for live pipeline metrics).

**#253 — Summarisation Funnel View (`agency-perception-summarisation-funnel-view`)**
Funnel visualization of summarisation compression. Shows input event count → intermediate summaries → final output. Compression ratio badges. Property-only data.

### Optimisation Category

**#274 — Prompt Optimisation Pipeline View (`agency-optimisation-prompt-view`)**
Variant selection and scoring. Shows prompt variants with A/B comparison, scoring metrics (accuracy, latency, cost), selection history. Table with sparklines for variant performance trends. Dual data mode (spec + endpoint for scoring data).

### Social Category

**#231 — Cognitive Workbench (promoted from examples)**
Brain graph visualization. Promote from `examples/cognitive-workbench/` to `components/cognitive-workbench/`. Uses pages-graph-canvas with brain layout (ELK). Extraction pipeline view. Dual data mode.

**#275 — Memory Hygiene View (`agency-social-memory-hygiene-view`)**
Confidence scoring and eviction visualization. Memory entries table with confidence scores, staleness indicators, eviction policy thresholds. Sorted by confidence (low first for eviction candidates). Dual data mode (spec + endpoint for memory state).

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

### Batch 0 — Foundation

| Task | Description |
|------|-------------|
| Add `patternViewStyles` to mosaic-core | Shared CSS export |
| Create `PatternViewMixin` in agent-viewer | Spec property + ARIA + empty state |
| Create `src/views/` directory structure | Category subdirectories |
| Expand agent-viewer switch dispatch | Exhaustive over all PatternCategory types with placeholders |
| Gallery scaffold (#258) | Real YAML demo data structure |

### Wave 1 — Orchestration + Governance (6 issues)

| # | View | Complexity |
|---|------|------------|
| #267 | Supervisor | M — hub-and-spoke layout, agent cards |
| #276 | Routing strategy | M — decision tree + optional history table |
| #244 | Choreography | M — blackboard + reactive agents |
| #254 | Saga | S — two-column forward/compensation |
| #255 | Case definition diagram | L — graph-canvas + ELK, trust decorations |
| #268 | Judgment | S — three-section layout |

### Wave 2 — Deliberation + Communication (4 issues)

| # | View | Complexity |
|---|------|------------|
| #269 | Negotiation | M — proposal lifecycle, rounds |
| #270 | Coalition formation | M — capability matrix, coverage bars |
| #271 | Normative conflict | S — tree-based conflict display |
| #245 | Work item card HITL | S — task card with response form |

### Wave 3 — Perception + Optimisation + Social (7 issues)

| # | View | Complexity |
|---|------|------------|
| #272 | Observation pipeline | M — pipeline with tiered output |
| #273 | Decision narrative | M — signal-to-narrative pipeline |
| #246 | Summarisation pipeline | M — multi-stage pipeline, live metrics |
| #253 | Summarisation funnel | S — funnel visualization |
| #274 | Prompt optimisation | M — variant table, scoring sparklines |
| #231 | Cognitive workbench | L — promote from examples, graph layout |
| #275 | Memory hygiene | S — memory table with confidence scores |

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
- Each new `agency-*` component exports a Props interface
- AgencyComponentRegistry entry added (per PP-20260907)
- `yarn workspace @casehubio/mosaic-schema run generate` after each wave

## Protocols Checklist

| Protocol | Applies to | Action |
|----------|-----------|--------|
| PP-20260713 (customisation pattern) | Views with render callbacks | Typed config + callbacks, inline styles in callback output |
| PP-20260930 (event namespace) | Views emitting bubbling events | `<component-prefix>:<action>` naming |
| PP-20260907 (registry props) | All new agency-* elements | Export Props interface, add to AgencyComponentRegistry |

## References

- `packages/mosaic-core/src/types/patterns/` — PatternCategory type system
- `packages/mosaic-core/src/types/orchestration.ts` — YAML spec types and runtime types
- `packages/agent-viewer/src/agent-viewer.ts:262-278` — current dispatch logic
- `packages/agent-viewer/src/voting-view.ts` — existing pattern view reference
- `packages/agent-viewer/src/coordination-pipeline-view.ts` — playbook visualization (CoordinationPipeline type)
- `components/conversation-viewer/` — standalone deliberation workbench
- `examples/cognitive-workbench/` — barebones cognitive visualization (#231)
- casehubio/blocks#337 — Java agency repo restructure (7-category hierarchy)
- `docs/protocols/mosaic/component-customisation-pattern.md` — PP-20260713
- `docs/protocols/mosaic/custom-event-namespace-prefix.md` — PP-20260930
- `docs/protocols/mosaic/component-registry-props.md` — PP-20260907
- `docs/guides/contributor-guide.md` — architecture and conventions
