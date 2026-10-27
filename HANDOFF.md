# HANDOFF — casehub-mosaic

## Last Session (2026-10-27)

Issue #288 — Execution/Reports tab architecture for pattern views, CognitionConfig type rework grounded against CognitionDefinition from agency/agentic-yaml.

### What happened

1. **CognitionConfig type rework** — replaced 12 `Record<string, unknown>` sections with 13 typed `*ConfigSpec` interfaces matching `CognitionDefinition` from `agency/agentic-yaml`. Design review caught a critical mistake: initial audit targeted `CognitionConfig.java` (CDI runtime toggles, never flows to frontend) instead of `CognitionDefinition` (YAML schema type). The existing 13-section structure was correct — only the depth was wrong.
2. **Execution/Reports tab architecture** — PatternViewMixin gained `renderTabBar()`/`renderTabPanel()` with override points (`renderExecution()`/`renderReports()`). Full ARIA: roving tabindex, `aria-controls`, `aria-labelledby`, Home/End keys. Shared header CSS extracted to `patternViewStyles` with `--pattern-badge-color`.
3. **View migrations** — all 6 Wave 3 views (summarisation, observation, narrative, prompt-optimisation, cognition, memory-hygiene) wrapped with Execution/Reports tabs. Migration was mechanical for 5 views (move render body into renderExecution).
4. **cognition-view rework** — replaced generic module card grid with typed config rendering: config fields with Duration highlighting, axis weight bar charts for drive config, nested sub-sections for mood baseline and narrative synthesis gate.
5. **Cross-repo issues filed** — agency#368 (CognitionDefinition REST endpoint), neocortex#713 (toggle-to-section mapping documentation)
6. **Issues closed** — #288, #287 (cognition-view enrichment fulfilled by this work)
7. **Blog entry** — "The Type That Was Already Right" covering the CognitionConfig vs CognitionDefinition conflation

### Branch state

- Landed on main as `bb521cd` + `2436528` (2 squashed commits)
- Branch `issue-288-grounded-pattern-views` stamped closed

### Key decisions

- Execution/Reports tabs are in PatternViewMixin (override methods, not TabDefinition composition) — fixed 2-tab structure doesn't justify the flexibility
- CognitionConfig mirrors CognitionDefinition (YAML schema), NOT CognitionConfig.java (CDI runtime) — the frontend receives YAML-derived data
- `RelationshipStageConfigSpec` left as `Record<string, unknown>` — full tier structure deferred
- Reports tabs are stubs — report TS types don't exist yet

## Known Issues

- **Canvas pans during hold-to-drag** — pages#474, partially addressed
- **Property editing unavailable** — pre-existing yamlPaths/buildFlatGraph mismatch
- **Pre-existing typecheck errors** — channel-activity, org-diagram strictness issues
- **Pre-existing test failures** — 5 in deliberation views (voting/debate threshold rendering)
- **Clinical repo** — extensive stale `blocks-` references in docs, specs, source

## What's Next

Priority order:

1. **#289** — Fix agent-viewer TypeScript errors blocking mosaic parent bootstrap (S/Low — build blocker)
2. **#257** — Align pattern views to real PatternSpec YAML (M/Med — partially addressed by slot 219, needs triage)
3. **#256** — Align eidos-view to real AgentDescriptor model (M/Med — same grounding pattern as #288)
4. **#258** — Gallery demo data from real YAML (L/High — cross-repo, needs engine projections)
5. **#282** — Thin CLAUDE.md (S/Low — housekeeping)

Epic #277 sub-issues mostly closed. Remaining open: #255 (YAML case definition diagram — XL/High), #257, #258.

## Cross-Repo Changes (2026-10-07)

- **Branch:** `issue-530-document-diff-extraction` — DocumentDiff reduced to thin subclass extending PagesDocumentDiff from `@casehubio/pages-document-diff`
- **Commit:** `1b8f92b` — 1084 lines removed, 30 added. Thread/timeline/selection domain logic retained.
- **Depends on:** `casehubio/casehub-pages#530` — `pages-document-diff` package must be published before this branch can build.

## Cross-Module

- pages#473 — radial-layout migration (CLOSED)
- pages#474 — hold-to-drag pan + layout suppression (layout fix landed, pan fix open)
- pages#530 — Milkdown editor, includes `pages-document-diff` extraction (branch `issue-530-document-diff-extraction` here)
- agency#368 — CognitionDefinition REST endpoint for frontend consumption
- neocortex#713 — Document CognitionConfig toggle-to-CognitionDefinition section mapping
