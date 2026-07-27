# HANDOFF — casehub-blocks-ui

**Branch:** issue-97-session-workbench (closed)
**Date:** 2026-07-27
**Issues:** casehubio/blocks-ui#97, #98

## What landed

Two issues closed on one branch:

- **#97** — Worker session management components: session-list (CRUD + SSE), session-detail (terminal/git/health/events tabs), session-workbench (split-pane composition), showcase page with mock data. Unblocks casehubio/devtown#123.
- **#98** — Dark mode theme fixes across 7 components and 5 example pages. Replaced hardcoded `background: white`, fake `--pages-*-color` tokens, and `--pages-gray-*` references with real `--pages-neutral-*` scale tokens. Fixed entity-list and session-list `row-activate` handler using `detail.key` instead of non-existent `detail.index`/`detail.rowIndex`. Default showcase theme → `casehub-dark`.

**Stats:** 6 commits (squashed to 5), pushed to both fork and upstream

## What's left

- Update `docs/repos/casehub-blocks-ui.md` in parent repo with latest component descriptions · S · Low (casehubio/parent#393)
- Downstream apps (aml, chat-app, claudony, clinical, life, openclaw) need template updates for blocks- prefix · M · Low (each app session)
- pages-table pagination buttons still use light backgrounds (upstream pages fix needed) · S · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #85 | Column resizing interaction with single-grid model | M | High | |
| #84 | Variable row heights with cell spanning | L | High | |
