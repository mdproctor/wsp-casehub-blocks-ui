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
