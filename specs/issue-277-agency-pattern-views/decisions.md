## D1: Pattern View Registry Architecture

**Choice:** Map-based registry in mosaic-core
**Alternatives:**
- Convention-based tag naming — zero infrastructure but brittle naming, no fallback detection, can't override
- Dynamic import registry — lazy loading is over-engineering for a bundled library consumed by downstream apps
**Rationale:** Follows the existing `registerStatus` pattern in mosaic-core. Views self-register at import time via side-effect. Agent-viewer looks up `"category:type"` keys and dynamically renders the element. Clear fallback for unregistered views.
**Trade-offs:** Requires view module to be imported before agent-viewer renders (side-effect registration), but this is already standard in the codebase.
**Sources:** `packages/mosaic-core/src/types/status.ts` (registerStatus pattern), `packages/agent-viewer/src/agent-viewer.ts` (current dispatch logic)
**Exploration:** quick
**Status:** captured

## D2: Execution Batching Strategy

**Choice:** Hybrid: foundation batch + category waves
**Alternatives:**
- Category-sequential — one category at a time in dependency order. Clean boundaries but uneven batch sizes (1-7 issues per batch)
- Thin-slice by complexity — group by implementation size. Uniform batches but half-finished categories at each milestone
**Rationale:** Front-loads registry infrastructure that everything depends on. Batches small categories together (Governance with Orchestration, Communication with Deliberation) for balanced wave sizes. Each wave delivers complete categories.
**Trade-offs:** More complex ordering than pure sequential. Category boundaries less crisp within a wave.
**Sources:** Epic #277 issue body (category listing and dependency structure)
**Exploration:** quick
**Status:** captured

## D3: Component Placement

**Choice:** Per-category packages (packages/pattern-views-orchestration/, etc.)
**Alternatives:**
- Extend agent-viewer — add all views to existing package. Grows monolithic (25+ views) and mixes container with content
- Single pattern-views package — one package with category subdirs. Middle ground but still monolithic, no independent consumption
**Rationale:** Mirrors the type system hierarchy (each PatternCategory variant has its own interface). Clean import boundaries. Each category is independently consumable by downstream apps. Manageable package sizes.
**Trade-offs:** More packages to maintain, more workspace entries. But Yarn workspace overhead is minimal.
**Depends on:** D1 (registry enables decoupled packages)
**Sources:** `packages/agent-viewer/` (current structure), `packages/mosaic-core/src/types/patterns/` (per-category type files)
**Exploration:** quick
**Status:** captured

## D4: Wave Composition

**Choice:** 5-phase hybrid execution
- Batch 0 (Foundation): pattern view registry in mosaic-core, migrate voting/debate to registry, gallery scaffold (#258)
- Wave 1 (Orchestration + Governance): #267 supervisor, #276 routing strategy, #244 choreography, #254 saga, #255 case definition diagram, #268 judgment
- Wave 2 (Deliberation + Communication): #269 negotiation, #270 coalition, #271 normative conflict, #245 work item card (HITL)
- Wave 3 (Perception + Optimisation + Social): #272 observation, #273 decision narrative, #246 summarisation pipeline, #253 summarisation funnel, #274 prompt optimisation, #231 cognitive workbench (promote), #275 memory hygiene
- Wave 4 (Infrastructure): #247 desired state graph, #248 app topology, #249 adaptive topology, #250 project dashboard, #251 focus mode, #252 desired state topology
**Alternatives:** (none presented — composition derived from D2 batching strategy)
**Rationale:** Builds on existing orchestration strength (3 closed issues). Groups small categories with larger ones for balanced wave sizes. Infrastructure last because it doesn't depend on the pattern-view registry.
**Trade-offs:** Wave 3 is the largest (7 issues across 3 categories). Could be split but all three categories are greenfield with similar complexity.
**Depends on:** D2 (batching strategy)
**Sources:** Epic #277 issue body, child issue states from GitHub
**Exploration:** quick
**Status:** captured

## D5: Pattern View Shared Infrastructure

**Choice:** Conventions only — no base class or mixin
**Alternatives:**
- PatternViewMixin — Lit mixin adding spec property, ARIA setup, loading/error states. More DRY but adds abstraction layer
- PatternViewBase class — abstract base class. Strongest contract but limits inheritance and mixin composition (DataSourceMixin, LiveRegionMixin)
**Rationale:** Existing views (voting-view, debate-view) extend LitElement directly and work well. The contract is simple: accept a `spec` property typed as the appropriate PatternCategory variant, set up ARIA in connectedCallback. Shared utilities (StatusBadge, stateCategoryStyles) are already importable from mosaic-core. YAGNI — don't create an abstraction before proving the need.
**Trade-offs:** ARIA boilerplate and empty state rendering repeated across views. Acceptable for 15-20 views.
**Sources:** `packages/agent-viewer/src/voting-view.ts`, `packages/agent-viewer/src/debate-view.ts` (existing pattern)
**Exploration:** quick
**Status:** captured
