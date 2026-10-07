# HANDOFF — casehub-blocks-ui

## Last Session

Cleanup and audit session. Drained 23-item plan (LSP schema cleanup queue + codebase audit). Then tackled 5 more fix/migration issues. Key changes:

- **moveSwfTask bug fix** — now rewires ALL `then:` references on move, not just the edge source
- **case-dependency-graph** migrated from d3-force/d3-selection to pages `computeElkLayout(algorithm: 'force')` + `pages-graph-canvas`
- **addSwfTask** now produces disconnected nodes — adds `then: exit` to previous last task before appending
- **case diagram reverted to ELK layout** — stack-column unsuitable for bipartite case topology (workers lateral to bindings)
- **Worker node selection size fix** — collapsed height 130→165 to match actual stencil render height
- **radial-layout migrated** to pages graph-renderer (pages#473 landed, blocks-ui now imports)
- **Layout suppression during drag** — `_runLayout` deferred while NodeMoveCoordinator is active (pages#474)

## Known Issues

- **Canvas pans during hold-to-drag** — pages#474, partially addressed (layout suppression landed, pan suppression still open)
- **Property editing unavailable** — "No YAML path for task node" in case diagram examples. Pre-existing `yamlPaths`/`buildFlatGraph` mismatch.
- **Pre-existing typecheck errors** — channel-activity, org-diagram, and several other components have `exactOptionalPropertyTypes` strictness issues

## What's Next

Open issues are all feature work — no remaining cleanup. Three tracks:
- **Evolution conductor** — slot 206 has epic #174 done, epic #188 (operational completeness) is next
- **Agent setup wizard** — #166, paused in stack
- **Org diagram** — #157, rich agent properties

## Cross-Repo Changes (2026-10-07)

- **Branch:** `issue-530-document-diff-extraction` — DocumentDiff reduced to thin subclass extending PagesDocumentDiff from `@casehubio/pages-document-diff`
- **Commit:** `1b8f92b` — 1084 lines removed, 30 added. Thread/timeline/selection domain logic retained.
- **Depends on:** `casehubio/casehub-pages#530` — `pages-document-diff` package must be published before this branch can build.

## Cross-Module

- pages#473 — radial-layout migration (CLOSED)
- pages#474 — hold-to-drag pan + layout suppression (layout fix landed, pan fix open)
- pages#530 — Milkdown editor, includes `pages-document-diff` extraction (branch `issue-530-document-diff-extraction` here)
