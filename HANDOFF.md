# HANDOFF — casehub-blocks-ui

**Branch:** main (no active branch)
**Date:** 2026-07-29
**Issues:** #102 (closed)

## What landed

Fixed CI flakiness and session-list row selection (#102). Four production fixes: entity-tree Array.isArray guard, channel-feed scrollIntoView optional chaining, channel-topic-bar active class misplaced in size attribute, session-list selection="single" + selectedKeys tracking. Removed three stale test files (themes.test.ts, trend-source-mixin DataSource tests, fetch-source extraction test). CI green.

Closed three completed epics: #56 (app delivery), #35 (cross-repo migration), #36 (openclaw). All five consuming apps fully migrated. Zero open issues.

## What's left

- Project main has uncommitted changes from another session — blocks-timeline, channel-activity, examples, blocks-ui-core commitment-pill work · M · Med
- Update `docs/repos/casehub-blocks-ui.md` in parent repo with latest component descriptions · S · Low (casehubio/parent#393)
- pages-table pagination buttons still use light backgrounds (upstream pages fix) · S · Low

## Known issues

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## What's next

Zero open issues on blocks-ui. New work requires filing issues first.
