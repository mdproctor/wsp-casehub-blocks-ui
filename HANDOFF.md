# HANDOFF — casehub-blocks-ui

**Branch:** issue-100-commitment-strategy-pill-adoption (closed)
**Date:** 2026-07-29
**Issues:** casehubio/blocks-ui#100, #101

## What landed

Two issues closed on one branch:

- **#100** — Refactored `commitmentLifecycleStrategy` to delegate to `stateProgressionStrategy` with the canonical 7-state model. Deleted the parallel 4-stage ontology (`COMMITMENT_STAGES`). Default resolver switched from `linearResolveStatus` (index-based) to `defaultResolveStatus` (transition-based — handles branching state machines). Fixed DELEGATED terminal state gap by expanding `StageConfig.terminal` to include `'transfer'`.

- **#101** — Adopted `commitment-state-pill` across channel-activity (channel-message, channel-task-panel, channel-correlation-panel, channel-thread). Promoted pill and `stateCategoryStyles` from commitment-viz to blocks-ui-core to avoid component-to-component dependency (ARC42STORIES §2). Fixed commitment lookup key from `msg.id` to `correlationId`. Removed dead `commitments` property from channel-feed. DRY'd `_isTerminal` via `isTerminalCommitmentState`.

**Stats:** 5 commits squashed to 2, pushed to fork and upstream

## Known issues

- **pages-component source/dist mismatch** (carried from prior session): pages repo source has a refactored SourceConnector API that breaks DataSourceAdapter.connect(). Examples vite config works around this by NOT aliasing pages-component to source.
- **Pre-existing test failures in channel-activity `dist/`**: 98 compiled test files run alongside source tests due to vitest config not excluding `dist/`. Not introduced by this branch.
