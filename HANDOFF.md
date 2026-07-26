# HANDOFF — casehub-blocks-ui

**Branch:** issue-94-blocks-prefix-and-cleanup (closed)
**Date:** 2026-07-26
**Issues:** casehubio/blocks-ui#94, #93, #88

## What landed

Three issues on one branch — barrel import fix, element migration, full prefix rename:

- **#88** — Narrowed 17 `@casehubio/pages-primitives` barrel imports to `/a11y` sub-path. Eliminates transitive pages-modal registration that caused duplicate CustomElementRegistry crash in aliased bundler setups (GE-20260720-96fab8).
- **#93** — Migrated 19 native `<button>` elements in channel-activity to `<pages-button>` from `@casehubio/pages-ui-components`. Fixed 2 hardcoded colors in channel-nav to `--pages-*` tokens. Textarea/select kept native (imperative API would break with wrapper components).
- **#94** — Renamed all 87 custom element tags to `blocks-*` prefix (59 component tags + 27 example pages + 1 shell). 141 files changed. Platform namespace consistency: `pages-*` for pages, `blocks-*` for blocks-ui.

**Stats:** 3 commits after squash, pushed to both fork and upstream

## What's left

- Update `docs/repos/casehub-blocks-ui.md` in parent repo with latest component descriptions · S · Low (casehubio/parent#393)
- Downstream apps (chat-app, clinical) need template updates for blocks- prefix · M · Low (each app session)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #85 | Column resizing interaction with single-grid model | M | High | |
| #84 | Variable row heights with cell spanning | L | High | |
