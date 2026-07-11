# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-11
**Branch:** `main` — #48 closed
**Priority:** #49 (TypedDataSet integration) is the critical next step.

---

## Last Session

Completed #48 — migrated `pages-data-table` and four a11y mixins from blocks-ui to pages. Created two new pages packages: `pages-primitives` (a11y mixins) and `pages-data-table`. Updated all 12 blocks-ui consumer imports to point directly at pages packages (no re-exports). Design review: 4 rounds, 15 issues, all resolved ($14.38). Filed #49 for TypedDataSet integration after discovering the old `PagesTable` still renders in pages examples from stale dist — two overlapping tables is unacceptable.

Key discovery: Lit components in blocks-ui were built on `rows: unknown[]` instead of `TypedDataSet` — a type-safety regression. #49 addresses this: make `pages-data-table` consume `TypedDataSet` natively, then rename to `pages-table`, replace the stale `PagesTable`, and migrate examples.

## Immediate Next Step

Pick up #49 — TypedDataSet integration. Start with brainstorming the `pages-data-table` API change: how to accept `TypedDataSet` while keeping standalone `rows`/`columns` support for blocks-ui consumers.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #49 | TypedDataSet integration + rename pages-data-table → pages-table | L | High | Blocks PagesTable example migration; 43 old tests to recover from git |
| #46 | pages-data-table shows "No data" in trust-score-panel/audit-trail-viewer | S | Med | Paused on stack |
| #33 | Subscription editor component | M | Med | |
| #34 | Notification preferences and suppression UI | M | Med | |
| #26 | Data-table row and column spanning | M | Med | |
| #35 | Cross-repo component migration tracking | L | High | Epic |
