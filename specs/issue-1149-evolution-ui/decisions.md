# Decisions — #1149 Evolution Conductor UI

## D1: Combined spec covering all five issues

**Choice:** One design spec for all 3 editors (#175-#177) + workbench (#178) + sample page (#179)
**Alternatives:**
- Per-editor specs — more granular but risks divergent patterns across editors that share a domain
- Two specs (editors + workbench) — unnecessary split; the workbench design depends on editor API contracts
**Rationale:** The editors share a domain API layer, TypeScript types, and consumption model. A combined spec captures the shared pattern once and specialises per editor, ensuring consistency.
**Trade-offs:** Larger spec document, but the design is more coherent.
**Sources:** casehubio/blocks-ui#174 (epic), #175-#179 (issues)
**Exploration:** quick
**Status:** captured

## D2: Dual data mode + file engine issues for API gaps

**Choice:** Design editors with dual data mode (endpoint or property). File engine issues for missing `getWatchPatterns()`, `getStages()`, `getCategories()` endpoints.
**Alternatives:**
- Dual data mode only — defers engine issues, risks them being forgotten
- Engine-first — blocks on engine slot work, delays UI progress
**Rationale:** Dual data mode is the established blocks-ui pattern. Most components already support it. Filing engine issues ensures the API gaps are tracked without blocking UI work. Pre-release platform — API additions are cheap.
**Trade-offs:** Endpoint mode for watch-pattern-editor and gate-policy-editor won't be functional until engine issues land.
**Sources:** EngineEvolutionApi.java (no getWatchPatterns/getStages/getCategories), WatchPatternStore.findActive() (exists internally), DenyPatternView.java (complete API)
**Exploration:** quick
**Status:** captured

## D3: Shared types/API package, independent editor elements

**Choice:** One component package `components/evolution-config/` with shared `types.ts` + `api.ts` + three independent LitElement editors. No shared mixin or base class.
**Alternatives:**
- Three separate packages — duplicates types and API class across packages
- Shared CrudEditorMixin — only 2 of 3 editors are CRUD lists (gate-policy is a stage pipeline); premature abstraction for 2 consumers
- Shared base class — too rigid, prevents editors from diverging when needed
**Rationale:** Follows notification-inbox precedent (7 elements sharing api.ts and types.ts in one package). The shared layer is data and API, not UI structure. The CRUD lifecycle is ~30 lines of state that each editor can own. Gate-policy-editor is structurally different (per-stage selector, not add/remove list).
**Trade-offs:** Some boilerplate repeated across deny/watch editors. Extractable to a mixin later if a pattern solidifies.
**Sources:** components/notification-inbox/src/ (precedent: api.ts, types.ts, 7 elements), components/preferences-editor/src/api.ts (API class pattern)
**Depends on:** D1 (combined spec)
**Exploration:** quick
**Status:** captured

## D4: Gate-policy-editor as grouped table with inline mode selectors

**Choice:** Table with stage rows grouped by domain. Columns: stage name, ordinal, gate checkpoint indicator, mode dropdown (GATED/AUTO/NOTIFY). Gate checkpoint stages get three-way dropdown; non-checkpoint stages get AUTO/NOTIFY only. Optional compact pipeline summary above the table as a read-only at-a-glance visualisation.
**Alternatives:**
- Horizontal stage pipeline with inline mode controls — visually striking but unwieldy for multi-domain support, harder to scan, fights the data shape (a mapping is a table)
- Card grid (one card per stage) — wastes space, poor information density
**Rationale:** The policy is fundamentally a mapping (stage → mode). A table is the most information-dense and scannable representation, consistent with blocks-ui's preferences-editor and mute-list patterns. Multi-domain stage grouping is natural in a grouped table. A pipeline visualisation can complement as a summary header without replacing the editing surface.
**Trade-offs:** Less visually distinctive than a pipeline view. The pipeline summary is optional read-only decoration.
**Sources:** GatePolicy.java:21-40 (Map<String, GateMode>), StageDescriptor.java:18 (ordinal, gateCheckpoint, domainId), preferences-editor pattern, grouped-data-view pattern
**Depends on:** D3 (independent editor elements)
**Exploration:** quick
**Status:** captured

## D5: Workbench as summary bar + tabbed content

**Choice:** Compact KPI summary bar (health score, active streams, circuit breaker, compliance) always visible at top. Below: tabbed content for Timeline (blocks-timeline), Streams (pages-table), Inbox (pages-table), Configuration (three editors). Domain extensibility via `TabDefinition[]` — consuming apps add domain-specific tabs.
**Alternatives:**
- Split-workbench (left overview, right detail) — follows trust-workbench precedent but awkward for a dashboard that has no natural master-detail relationship; left pane becomes a dashboard-within-a-dashboard
- Card grid (blocks-plan-model-dashboard pattern) — all sections visible simultaneously; good density but no tab-based extensibility; configuration editors need their own space
**Rationale:** The evolution workbench is a dashboard, not a master-detail view. A summary bar + tabs provides clean structure with natural tab-based extensibility. Existing `detail-pane` component already supports `TabDefinition[]` with badges, lazy element creation, ARIA tablist, and keyboard navigation. Domain apps (devtown, trading) add tabs for domain-specific views without modifying blocks-ui.
**Trade-offs:** Only one tab visible at a time — users can't see timeline and streams simultaneously. Mitigated by the summary bar providing at-a-glance health.
**Sources:** detail-pane (TabDefinition[], badge support), kpi-metric-row (summary bar), D9 engine decision (blocks-ui = composable primitives, apps compose)
**Depends on:** D3 (package structure), D4 (gate-policy in table form fits a tab)
**Exploration:** quick
**Status:** captured
