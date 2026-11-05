# HANDOFF — casehub-mosaic

## Last Session (2026-11-05)

Issue #282 — Thin CLAUDE.md, bring consumer and contributor guides to full coverage.

### What happened

1. **Thinned CLAUDE.md** — replaced the 70-line dense Key Directories table with a 25-line grouped summary pointing to guides for detail
2. **Updated consumer guide** — packages 4→10, components 37→66, organized by domain. Fixed stale references: diagram-core migration to pages, status domain count (11→14), case-flow-viewer name mismatch.
3. **Updated contributor guide** — added 5 packages (graph-stencil-org, agent-avatar-2d, agent-viewer, avatar, lsp-schemas) and 29 components to Full Module Details. Fixed diagram-core section to reflect migration to `@casehubio/pages-diagram-core`.
4. **Issues closed** — #282

### Branch state

- Landed on main as `b2ef478c` (1 squashed commit)
- Branch `issue-282-thin-claude-md` stamped closed

### Key decisions

- Detailed component descriptions live in the guides (consumer + contributor), not CLAUDE.md — no separate catalog file needed
- CLAUDE.md retains a grouped navigation index (package groups + component domains) sufficient for routing to the right guide
- Legacy packages (blocks-ui-core, blocks-ui-schema) excluded from guides — empty/dist-only remnants from the blocks→agency rename

## Known Issues

- **Canvas pans during hold-to-drag** — pages#474, partially addressed
- **Property editing unavailable** — pre-existing yamlPaths/buildFlatGraph mismatch
- **Pre-existing typecheck errors** — channel-activity, org-diagram strictness issues
- **Pre-existing test failures** — 5 in deliberation views (voting/debate threshold rendering)
- **Clinical repo** — extensive stale `blocks-` references in docs, specs, source

## What's Next

**Next session: #234 — interactive chain navigator with inline editing.**

Start fresh from main (stale `issue-234-chain-navigator` branch has only #233 fixes, all already on main — no #234 work was started). Needs brainstorming for traversal algorithm, inline editing UX, and cycle detection. See issue body for full spec.

The chain navigator is a third tab (Table / Chain / Diagram) or replaces the table for chain-forming relationship kinds (SUPERVISES, ESCALATES_TO, DELEGATES_TO, REPORTS_TO). Key files:
- `components/agent-relationship-editor/` — the host component
- `packages/graph-stencil-org/src/adapter/ego-subgraph.ts` — single-hop extraction (needs transitive version)

Other open issues:
- **#315** — Cognitive state dashboard (L/Med)
- **#307** — Avatar streaming (M/Med)
- **#305** — Avatar theme tokens (XS/Low)
- **#306** — Avatar text-only mode (S/Low)
- **#235** — org-diagram degraded mode

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
