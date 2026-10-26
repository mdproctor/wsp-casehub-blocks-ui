## D1: Pattern View Dispatch Architecture

**Choice:** Switch dispatch with exhaustiveness checking in agent-viewer
**Alternatives:**
- Map-based registry in mosaic-core — runtime flexibility but loses compile-time safety for a closed type system. Side-effect import ordering creates silent failures.
- Convention-based tag naming — brittle, no fallback detection
- Dynamic import registry — over-engineering for a bundled library
**Rationale:** PatternCategory is a closed discriminated union in mosaic-core. A switch statement gives compile-time exhaustiveness — adding a new type to the union forces updating the dispatch. Runtime registries trade this safety for flexibility that isn't needed (the type system is not an open extension point). The switch is ~40 lines and breaks visibly when types change.
**Trade-offs:** Adding a new pattern type requires touching agent-viewer's switch. Acceptable because adding a type also requires changing the PatternCategory union — the switch update is part of the same change.
**Sources:** `packages/mosaic-core/src/types/patterns/index.ts` (PatternCategory union), `packages/agent-viewer/src/agent-viewer.ts:262-278` (current dispatch)
**Exploration:** quick → revised after decision review (R1-01)
**Status:** revised

## D2: Execution Batching Strategy

**Choice:** Hybrid: foundation batch + category waves
**Alternatives:**
- Category-sequential — one category at a time in dependency order. Clean boundaries but uneven batch sizes (1-7 issues per batch)
- Thin-slice by complexity — group by implementation size. Uniform batches but half-finished categories at each milestone
**Rationale:** Front-loads shared infrastructure that everything depends on. Batches small categories together (Governance with Orchestration, Communication with Deliberation) for balanced wave sizes. Each wave delivers complete categories.
**Trade-offs:** More complex ordering than pure sequential. Category boundaries less crisp within a wave. Note: batching is primarily project management, not architectural constraint — views within and across categories are technically independent once dispatch exists.
**Sources:** Epic #277 issue body (category listing and dependency structure)
**Exploration:** quick
**Status:** captured

## D3: Component Placement

**Choice:** Subdirectories within agent-viewer package
**Alternatives:**
- Per-category packages (packages/pattern-views-orchestration/, etc.) — mirrors type hierarchy but 1-view packages (Optimisation, Communication) have disproportionate overhead. The "independently consumable" claim is dubious — apps want agent visualization, not individual categories.
- Single pattern-views package — middle ground but still monolithic, no organizational clarity
**Rationale:** `src/views/orchestration/`, `src/views/deliberation/`, etc. within agent-viewer gives organizational clarity without package-per-category overhead. Standalone workbenches that compose multiple views (cognitive-workbench, conversation-viewer) stay in `components/` as they already are — they're compositions, not individual pattern views. This aligns with how coordination-pipeline-view already lives in agent-viewer.
**Trade-offs:** agent-viewer grows larger (25+ source files across subdirectories). Acceptable for a coherent visualization package.
**Depends on:** D1 (switch dispatch keeps everything in one package)
**Sources:** `packages/agent-viewer/` (current structure), reviewer R1-03 (factual correction: 5 of 7 type interfaces share base.ts)
**Exploration:** quick → revised after decision review (R1-03, R1-04)
**Status:** revised

## D4: Wave Composition

**Choice:** 5-phase hybrid execution
- Batch 0 (Foundation): shared styles + PatternViewMixin, switch dispatch expansion in agent-viewer, gallery scaffold (#258)
- Wave 1 (Orchestration + Governance): #267 supervisor, #276 routing strategy, #244 choreography, #254 saga, #255 case definition diagram, #268 judgment
- Wave 2 (Deliberation + Communication): #269 negotiation, #270 coalition, #271 normative conflict, #245 work item card (HITL)
- Wave 3 (Perception + Optimisation + Social): #272 observation, #273 decision narrative, #246 summarisation pipeline, #253 summarisation funnel, #274 prompt optimisation, #231 cognitive workbench (promote), #275 memory hygiene
- Wave 4 (Infrastructure): #247 desired state graph, #248 app topology, #249 adaptive topology, #250 project dashboard, #251 focus mode, #252 desired state topology
**Alternatives:** (none presented — composition derived from D2 batching strategy)
**Rationale:** Builds on existing orchestration strength (3 closed issues). Groups small categories with larger ones for balanced wave sizes. Wave 4 is infrastructure visualizations — technically independent of the pattern-view dispatch but part of the epic scope.
**Trade-offs:** Wave 3 is the largest (7 issues across 3 categories). Wave 4 items don't use PatternCategory types — they're project infrastructure, not agency patterns. Included per user decision to cover the full epic.
**Depends on:** D2 (batching strategy), D1 revised (switch dispatch replaces registry in Batch 0)
**Sources:** Epic #277 issue body, child issue states from GitHub
**Exploration:** quick → updated after D1 revision
**Status:** captured

## D5: Pattern View Shared Infrastructure

**Choice:** Shared `patternViewStyles` CSS export + light `PatternViewMixin`
**Alternatives:**
- Conventions only — each view extends LitElement directly. YAGNI misapplied: with 20 planned views of identical shape, the pattern IS known.
- PatternViewBase class — abstract base class. Limits single inheritance and conflicts with DataSourceMixin composition.
**Rationale:** 20 views will share identical `:host` CSS, ARIA setup (role="region", aria-label, aria-busy), and spec-property contracts. A `patternViewStyles` CSS export from mosaic-core prevents 20 copies of the same CSS from diverging. A thin PatternViewMixin adds the `spec` property, shared ARIA setup in connectedCallback, and empty-state rendering — composable with DataSourceMixin and LiveRegionMixin just like other Lit mixins in the codebase.
**Trade-offs:** Mixin is a new abstraction to learn. Acceptable because it follows the same pattern as DataSourceMixin which every contributor already knows.
**Sources:** `packages/agent-viewer/src/voting-view.ts` (shared CSS pattern), reviewer R1-05 (YAGNI challenge), reviewer R1-06 (mixin vs base class distinction)
**Exploration:** quick → revised after decision review (R1-05, R1-06)
**Status:** revised

## D6: View Naming Convention

**Choice:** `agency-<category>-<type>-view` custom element tag names
**Rationale:** Follows the `agency-` prefix convention (per contributor guide). Including category prevents collision (e.g. `agency-orchestration-supervisor-view` vs a hypothetical `agency-supervisor`). The `-view` suffix distinguishes pattern view components from other agency-* components. Source files named `<type>-view.ts` within category subdirectories.
**Trade-offs:** Longer tag names. Acceptable for clarity.
**Sources:** Contributor guide (agency- prefix convention), reviewer R1-13
**Exploration:** quick
**Status:** captured

## D7: Relationship to CoordinationPipeline Type

**Choice:** CoordinationPipeline is a playbook visualization type; OrchestrationPatternCategory views are agency pattern visualizations. They coexist — different data, different purpose.
**Rationale:** `CoordinationPipeline` (with `phases`, `barriers`, `swimLanes`, `matchRouting`) renders step-by-step playbook coordination — it's the visual representation of `PlaybookPatternCategory`. `OrchestrationPatternCategory` views (supervisor, choreography, saga, decomposition) render multi-agent coordination patterns with `routing`, `decomposition`, `activation`, `aggregation`, `termination` specs. The data shapes are different: CoordinationPipeline is about step sequencing; orchestration patterns are about agent routing and coordination strategies. The existing `coordination-pipeline-view` stays as-is in agent-viewer, alongside the new orchestration views.
**Trade-offs:** Two visualization families for "coordination" concepts may confuse contributors. The distinction (playbook steps vs agency patterns) should be documented.
**Sources:** `packages/agent-viewer/src/coordination-types.ts`, `packages/mosaic-core/src/types/patterns/orchestration.ts`, reviewer R1-12
**Exploration:** quick
**Status:** captured

## D8: Data Source Strategy

**Choice:** Mixed by complexity — simple views are property-only, complex views get dual data mode
**Rationale:** Pure pattern views (supervisor, negotiation, etc.) render a static spec — property delivery is sufficient. Views that show live execution state (summarisation pipeline, prompt optimisation with scoring) or need their own data (topology views, project dashboard) use DataSourceMixin/EventStreamController for endpoint-based fetch. The PatternViewMixin provides the spec property; views that need additional data compose with DataSourceMixin independently.
**Trade-offs:** No single data delivery pattern across all views. The spec knows which views need dual mode; each is decided individually during implementation.
**Sources:** User answer during clarifying questions, reviewer R1-11
**Exploration:** quick
**Status:** captured
