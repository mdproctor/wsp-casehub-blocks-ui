# Decisions — #266 Pattern Type Hierarchy

## D1: Scope

**Choice:** Full cross-repo — all three workstreams (pages PlaybookPatternType, blocks-ui agency categories, shared aggregate)
**Alternatives:**
- Blocks-ui only — simpler but leaves pages unformalised and defers the aggregate
- Blocks-ui + aggregate — partial, still defers pages formalisation
**Rationale:** The type system should be complete in one pass. Downstream issues (#267, #268, #269) need the full hierarchy.
**Trade-offs:** Larger change set, cross-repo coordination required. Vendored copy in blocks-ui `.casehub-packages/` must be updated when pages types change (see D8).
**Sources:** casehubio/blocks-ui#266, casehubio/blocks#337
**Exploration:** quick
**Status:** captured

## D2: Breadth (revised after review)

**Choice:** Full category union (all 7 names), but detailed spec interfaces only for categories with active UI consumers (orchestration, deliberation). Other categories get a minimal base spec.
**Alternatives:**
- Full specs for all 7 categories — risks speculative field definitions for categories with no consumer, coupled to unstable #337 taxonomy
- Incremental (orch + delib only) — type system incomplete, blocks future category additions
**Rationale:** The category string union is cheap and stable as a taxonomy marker. Detailed spec interfaces should only be defined when a UI consumer exists to validate the fields. #337 is still OPEN — the 7-category names are stable but per-category field sets are not.
**Trade-offs:** Categories without specs get a minimal base interface. Must add per-category specs when views land (#267 et al).
**Sources:** casehubio/blocks#337 (7-category hierarchy, OPEN), R1-03 review finding
**Exploration:** quick
**Status:** revised

## D3: Casing convention (revised after review)

**Choice:** Lowercase throughout ('supervisor', 'voting', 'debate') with SSE normalization layer at the EventStream deserialization boundary
**Alternatives:**
- UPPERCASE — matches Java enum convention but not YAML/UI convention
- Both with mapping — more code, no migration, but permanent complexity
- Keep UPPERCASE for runtime types — two conventions coexist
**Rationale:** Matches YAML source format, UI convention, and the existing agent-viewer types. Java backend sends UPPERCASE over SSE — a normalization function at the EventStreamController boundary converts to lowercase. This is a clean architectural separation: wire format stays as-is, UI types are consistent.
**Trade-offs:** Normalization layer adds a deserialization step. execution-monitor.ts and all SSE consumers need updating.
**Sources:** agent-viewer/pattern-types.ts (lowercase), blocks-ui-core/types/orchestration.ts (UPPERCASE), R1-01 review finding
**Exploration:** quick
**Status:** revised

## D4: Aggregate type location (revised after review)

**Choice:** Aggregate in blocks-ui-core (downstream repo)
**Alternatives:**
- pages primitives — would require pages to know about agency categories (wrong dependency direction)
- New shared package — adds a package to manage for minimal benefit
**Rationale:** blocks-ui depends on pages via Yarn portal resolutions to vendored copies in `.casehub-packages/`. blocks-ui can import PlaybookPatternCategory from the vendored pages package and union it with its own agency categories. Pages stays clean.
**Trade-offs:** When pages changes PlaybookPatternType, the vendored copy in blocks-ui must be updated and the aggregate re-exported. Standard workflow: rebuild pages → update vendored copy → typecheck blocks-ui.
**Sources:** Dependency direction: pages → blocks-ui (blocks-ui consumes pages via Yarn portal to `.casehub-packages/`), R1-02 review finding
**Exploration:** quick
**Status:** revised

## D5: Type structure approach (rationale corrected after review)

**Choice:** Two-level discriminated union (Approach A) — `category` + `type` discriminants
**Alternatives:**
- Flat union with category tag — retains existing single-switch pattern, but grows unwieldy as pattern count increases
- Category modules with shared base — legitimate pattern but base risks becoming a bag; weaker TypeScript narrowing on `extends`
**Rationale:** Enables two-level dispatch: agent-viewer routes by category to a category-specific renderer, which then routes by type. This is aspirational — current dispatch is a flat switch on `type`. The new structure maps to the Java package hierarchy from #337 and supports category-level routing as views are added.
**Trade-offs:** Slightly more boilerplate than a flat union. Requires refactoring current flat dispatch.
**Sources:** blocks #337 package hierarchy, current agent-viewer renderPattern() dispatch (agent-viewer.ts:262-270), R1-04 review finding
**Exploration:** quick
**Status:** revised

## D6: Spec granularity (unchanged, acknowledged trade-off)

**Choice:** Per-category specs — one interface per category (OrchestrationPatternSpec, DeliberationPatternSpec)
**Alternatives:**
- Per-pattern specs — one interface per pattern type (SupervisorSpec, VotingSpec). Maximum narrowing, ~15 interfaces, zero ambiguity
- Per-category with pattern-specific sub-specs — category spec + nested details discriminated union
**Rationale:** Balance between the current 16-field bag and 15+ per-pattern interfaces. Fields are grouped by domain relevance. Some optional-field ambiguity remains within each category (e.g. maxRounds applies to debate but not voting) — accepted as a trade-off for lower interface count. Per-pattern narrowing can be added within a category later if needed.
**Trade-offs:** Not fully type-safe within a category. TypeScript won't prevent setting `maxRounds` on a voting spec.
**Sources:** Current PatternSpec (agent-viewer/pattern-types.ts) — 16 optional fields, R1-05 review finding
**Exploration:** quick
**Status:** captured

## D7: SSE normalization (new, from review)

**Choice:** Add a normalization function at the EventStreamController deserialization boundary that maps UPPERCASE Java enum values to lowercase UI types
**Alternatives:**
- No normalization — keep UPPERCASE in runtime types, lowercase in YAML-sourced types (two conventions coexist)
- Change Java backend — coordinate with agency repo to serialize lowercase (large scope)
**Rationale:** Clean architectural boundary. Wire format is a backend concern; UI types are a frontend concern. The normalization function is a single-point mapping, easy to test, and keeps the rest of the frontend codebase consistent.
**Trade-offs:** One more function to maintain. Must be updated when new pattern types or categories are added.
**Depends on:** D3 (lowercase convention)
**Sources:** R1-01 review finding, execution-monitor.ts SSE consumption
**Exploration:** quick
**Status:** captured

## D8: Type deduplication — single source of truth (new, from review)

**Choice:** Consolidate all pattern-related types into blocks-ui-core. agent-viewer's pattern-types.ts becomes a re-export. orchestration.ts UPPERCASE definitions are replaced with lowercase from the new category hierarchy.
**Alternatives:**
- Keep separate definitions with mapping — permanent duplication
- Move canonical types to agent-viewer — wrong level; blocks-ui-core is the domain types package
**Rationale:** blocks-ui-core already owns domain types (ExecutionState, AgentRef, etc.). Pattern categories are domain types. agent-viewer is a rendering package that should consume types, not define them. The structural differences between the two AgentRef interfaces (orchestration.ts has `readonly id`, agent-viewer's doesn't) are reconciled in the canonical definition.
**Trade-offs:** agent-viewer becomes dependent on blocks-ui-core for types (already true for other types). Vendored pages-step-viewer in .casehub-packages is unaffected — it's a separate pages package, not a copy of agent-viewer.
**Sources:** R1-06 review finding, blocks-ui-core/types/orchestration.ts, agent-viewer/pattern-types.ts
**Exploration:** quick
**Status:** captured
