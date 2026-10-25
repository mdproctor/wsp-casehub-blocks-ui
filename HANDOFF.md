# HANDOFF — casehub-mosaic

## Last Session (2026-10-25)

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

Continue with **epic #277** (Agency Pattern Views). Foundation is now solid (taxonomy aligned, staleness tests in place). Outstanding orchestration pattern views:
- #267 — Supervisor pattern view (hub-and-spoke) — M/Med
- #276 — Routing strategy visualization — M/High
- #244 — Choreography view (Blackboard) — M
- #254 — Saga compensation visualization — L/High
- #255 — YAML case definition diagram — XL/High

Plus deliberation (#269, #270, #271), perception (#246, #253, #272, #273), and other categories. #258 (gallery demo data from real YAML) is now unblocked.

## Cross-Repo Changes (2026-10-07)

- **Branch:** `issue-530-document-diff-extraction` — DocumentDiff reduced to thin subclass extending PagesDocumentDiff from `@casehubio/pages-document-diff`
- **Commit:** `1b8f92b` — 1084 lines removed, 30 added. Thread/timeline/selection domain logic retained.
- **Depends on:** `casehubio/casehub-pages#530` — `pages-document-diff` package must be published before this branch can build.

## Cross-Module

- pages#473 — radial-layout migration (CLOSED)
- pages#474 — hold-to-drag pan + layout suppression (layout fix landed, pan fix open)
- pages#530 — Milkdown editor, includes `pages-document-diff` extraction (branch `issue-530-document-diff-extraction` here)
