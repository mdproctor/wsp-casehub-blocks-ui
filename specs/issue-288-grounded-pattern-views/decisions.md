## D1: Scope — Execution tabs with real config, Reports tab stubs

**Choice:** Execution + Reports stubs
**Alternatives:**
- Execution only — defers Reports entirely, but leaves no placeholder for the tab pattern
- Full Execution + Reports — speculative on report shapes, larger scope without upstream types
- Infrastructure only — delays per-view rework, no visible progress
**Rationale:** Execution tab rendering is fully grounded (types exist). Reports stubs establish the tab pattern and placeholder UI without speculating on report data shapes. Report type interfaces noted as TODOs.
**Trade-offs:** Report stubs show empty state until upstream types land. Views may need a second pass when report types arrive.
**Sources:** Issue #288, mosaic-core pattern-categories.ts, slot 219 commits
**Exploration:** quick
**Status:** captured

## D2: Tab infrastructure — PatternViewMixin override methods

**Choice:** Override-method pattern in PatternViewMixin
**Alternatives:**
- TabDefinition composition — overkill for fixed 2-tab structure, detail-pane's element-creation model doesn't fit self-rendering views
- Decorator/slot pattern — more DOM complexity, unusual for this codebase
**Rationale:** Fixed Execution/Reports tabs don't vary per view. Override methods (renderExecution/renderReports) are idiomatic Lit, migration is mechanical (move existing render body into renderExecution), consistent ARIA via mixin.
**Trade-offs:** Less flexible if a view ever needs a third tab (unlikely — would be a different component).
**Sources:** detail-pane TabDefinition API, PatternViewMixin current implementation
**Exploration:** quick
**Status:** captured

## D3: CognitionConfig — Type the 13+ modules from Java source

**Choice:** Type the modules now based on Java CognitionConfig audit
**Alternatives:**
- Leave opaque for #287 — keeps scope smaller but cognition-view stays fabricated
**Rationale:** Java audit revealed CognitionConfig is fundamentally wrong — not 13 opaque Record<string, unknown> modules but 23 boolean toggles + separate per-subsystem config beans (MoodConfig, DriveConfig, etc.). Fixing this now grounds the view properly.
**Trade-offs:** Broader scope. Some subsystems are toggle-only with no config record — those get boolean-only interfaces. File agency issues if Java source is missing typed configs.
**Sources:** neocortex/cognition-api CognitionConfig.java, CognitionDefaultBeans.java
**Exploration:** quick
**Depends on:** D1 (scope includes typing work)
**Status:** captured

## D4: Migration strategy — In-place rework

**Choice:** In-place rework of existing views
**Alternatives:**
- Rebuild from scratch — higher risk, current rendering logic may be lost
- Parallel views + deprecate — pre-release policy prohibits deprecation markers
**Rationale:** Each view already renders config correctly. Wrapping existing render body in renderExecution() is mechanical. No new files needed, minimal churn.
**Trade-offs:** Views accumulate incremental changes rather than clean-sheet design.
**Sources:** summarisation-view.ts, observation-view.ts (current rendering patterns)
**Exploration:** quick
**Depends on:** D2 (tab architecture determines migration shape)
**Status:** captured

## D5: Cross-repo changes — File issues, don't edit

**Choice:** File issues on peer repos for needed changes
**Alternatives:**
- Edit peer repos directly — violates session boundary (peer repos have their own Claude sessions)
**Rationale:** casehub-mosaic session cannot commit to agency/neocortex repos. Issues document what's needed; peer sessions pick them up.
**Trade-offs:** Dependency on peer sessions to implement. Mosaic types may drift if issues aren't actioned.
**Sources:** User instruction, CLAUDE.md peer repos list
**Exploration:** quick
**Status:** captured
