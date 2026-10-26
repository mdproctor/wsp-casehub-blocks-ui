# HANDOFF — casehub-mosaic

## Last Session (2026-10-26)

Pattern view implementation session for epic #277. Built 8 new dedicated pattern views replacing all Wave 1 and Wave 2 placeholders in the agent-viewer dispatch.

### What happened

1. **Orchestration views (Wave 1)** — 4 new views replacing placeholder stubs:
   - `supervisor-view`: hub-and-spoke layout with routing strategy badge, worker spoke grid, aggregation card
   - `choreography-view`: shared blackboard with activation badge, reactive agent grid
   - `saga-view`: numbered forward steps with compensation arrows, retry policy metadata
   - `decomposition-view`: task decomposer with decomposition strategy badge, replan policy
2. **Deliberation views (Wave 2)** — 3 new views:
   - `negotiation-view`: buyer/seller parties with proposal/counter-proposal rounds, accepted/deadlocked outcomes
   - `coalition-view`: coalition members with capability coverage tags and coverage summary
   - `normative-view`: competing rule advocates with conflict icon, arbiter judgment card, binding ruling
3. **Governance view** — 1 new view:
   - `judgment-view`: vertical escalation chain with escalate arrows, judgment gate badge
4. **Agent card responsive fix** — changed from fixed `width: 280px` to `min-width: 200px` + `display: flex` to prevent overlapping at any panel size. Unified all grid containers to `minmax(220px, 1fr)` with `16px` gap.
5. **Gallery showcase** — all 8 views added as standalone sections AND agent-viewer compositions. Fixed variable ordering bug where viewer compositions referenced `descriptor` before initialization.

### Branch state

- **Branch:** `issue-277-agency-pattern-views` — 15 commits, clean
- **Typecheck:** passes
- **Tests:** 213 pass, 5 pre-existing failures (voting/debate threshold rendering)
- **Playwright:** all views verified, zero console errors

### Key decisions

- Agent cards are responsive (min-width, not fixed width) — grid cells size the cards
- Orchestration views use peach (#fab387) category badges, deliberation uses purple (#cba6f7), governance uses red (#f38ba8)
- Each view follows PatternViewMixin pattern with patternViewStyles base
- Gallery sections numbered: 2a–2l (agent-viewer compositions), 3a–3h (standalone), 4a–4d (orchestration)

## Previous Session (2026-10-25)

Analysis session for playbook UI improvement. Started with #257 (align pattern views to real YAML), discovered the problem was cross-repo, expanded to epic #283.

### What happened

1. **Audited mosaic PatternSpec vs engine YAML** — found PatternSpec already dead in source (#266 landed), real gap was CoordinationPipeline/CoordinationStep types and category entries without Java backing
2. **Discovered extensible registry pattern** — agency has `PatternCompiler` with 7 strategy registries + `AgencySchemaGenerator` producing JSON Schema for 24 discriminated types
3. **Cross-repo naming inconsistency** — `*Spec` (47 types in agentic-yaml) vs `*Definition` (9 types in summarisation-yaml), same semantic role. Pure drift.
4. **Entry-level drift audit** — of 20+ mosaic category entries, 3 match Java PatternType exactly, 15 are real Java concepts in other repos (agency, neocortex, qhorus) but not formally connected, 1 naming mismatch
5. **Created epic #283** with sub-issues: agency#353, mosaic#284, neocortex#673, engine#1258
6. **Slot 219** ran the cross-repo work — all 4 issues closed, work landed on all canonical mains
7. **Slot 219 archived**, empty #257 branches deleted

### Key decisions made in slot 219

- `PatternSpec` → `PatternDefinition` (agency naming unification)
- `CapabilityCategory` + `CapabilityType` enums added to agency-core
- `CapabilityCatalogGenerator` produces JSON taxonomy artifact
- Mosaic staleness test enforces TS/Java parity at CI
- Schema-aligned discriminated union sub-spec types in mosaic

## Known Issues

- **Canvas pans during hold-to-drag** — pages#474, partially addressed
- **Property editing unavailable** — pre-existing yamlPaths/buildFlatGraph mismatch
- **Pre-existing typecheck errors** — channel-activity, org-diagram strictness issues
- **Clinical repo** — extensive stale `blocks-` references in docs, specs, source (registerPanel calls use `blocks-*` element names)

## What's Next

Continue with **epic #277** — remaining Wave 3 placeholders:

**Perception** (3 views):
- #246 — Summarisation pipeline view — M/Med
- #272 — Observation/affordance pipeline view — M/Med
- #273 — Decision narrative pipeline view — M/Med

**Optimisation** (1 view):
- #274 — Prompt optimisation pipeline view — S/Med

**Social** (2 views):
- #231 — Cognitive workbench — M/Med
- #275 — Memory hygiene view — S/Med

**Other outstanding** (not pattern views):
- #276 — Routing strategy visualization — M/High
- #254 — Saga compensation visualization (richer than current) — L/High
- #255 — YAML case definition diagram — XL/High
- #258 — Gallery demo data from real YAML — M/Med (unblocked)

## Cross-Repo Changes (2026-10-07)

- **Branch:** `issue-530-document-diff-extraction` — DocumentDiff reduced to thin subclass extending PagesDocumentDiff from `@casehubio/pages-document-diff`
- **Commit:** `1b8f92b` — 1084 lines removed, 30 added. Thread/timeline/selection domain logic retained.
- **Depends on:** `casehubio/casehub-pages#530` — `pages-document-diff` package must be published before this branch can build.

## Cross-Module

- pages#473 — radial-layout migration (CLOSED)
- pages#474 — hold-to-drag pan + layout suppression (layout fix landed, pan fix open)
- pages#530 — Milkdown editor, includes `pages-document-diff` extraction (branch `issue-530-document-diff-extraction` here)
