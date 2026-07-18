# HANDOFF — casehub-blocks-ui

**Date:** 2026-07-18
**Branch:** `main`
**Priority:** #83 closed — schema-form migrated to pages as @casehubio/pages-form. Epic #81 children (#76–#80) now target pages.

---

## Last Session

Moved schema-form from blocks-ui-core to casehub-pages as a new `pages-form` package. Removed schema-form source, example page, and barrel export from blocks-ui. Updated work-item-detail to use `<pages-schema-form>` with a local `SchemaFormElement` interface.

In pages: created pages-form package (22 tests), filled gaps (nested object editing, array editing with add/remove), aligned CSS with pages' form input styling, created three-tab gallery example (Schema/HTML/YAML), wired tsPath auto-detection in generate-samples.js, merged Form Components into Schema Form as tabbed comparison.

Key insight: the initial migration copied blocks-ui's schema-form wholesale without auditing pages' existing form infrastructure. Pages already had 6 form input Web Components, type definitions, and gallery examples. The audit revealed schema-form's genuine value is the orchestrator layer (schema→form, display mode, nesting, arrays, field registry, form-level submit) — not the individual field rendering.

## Immediate Next Step

Epic #81 (schema-form enhancements) should be worked in the pages session. Children #76–#80 need re-filing against casehub-pages.

## What's Left

- Channel-activity gap analysis — connectors has 30 qhorus UI files diverged from blocks-ui's channel-activity · M · Med
- Parent deep-dive doc (`docs/repos/casehub-blocks-ui.md`) stale · S · Low
- casehub-pages#196 — table enhancements · M · Med
- Engine REST endpoint for routing decision data · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #33 | Subscription editor | M | Med | Blocked by pages #81 |
| #34 | Notification preferences UI | M | Med | Blocked by pages #81 |
| #26 | Data-table row and column spanning | M | Med | Clinical regulatory grid |

## References

- Previous: `git show HEAD~1:HANDOFF.md`
