# Design: Grounded Pattern Views — Execution/Reports Architecture

**Issue:** casehubio/casehub-mosaic#288
**Date:** 2026-10-27
**Branch:** issue-288-grounded-pattern-views

## Problem

Wave 3 pattern views (perception, optimisation, social) render configuration data
but have no architectural separation between what a system IS configured to do
(Execution) and what it DID (Reports). Additionally, CognitionConfig in mosaic-core
is fabricated — 13 opaque `Record<string, unknown>` modules instead of the real Java
structure (23 boolean toggles + per-subsystem typed config beans).

## Scope

- Add Execution/Reports tab architecture to PatternViewMixin
- In-place rework of 6 views with Execution tab rendering real config
- Reports tabs render stub placeholders (report TS types do not exist yet)
- Replace fabricated CognitionConfig with types matching Java source
- Full rework of cognition-view to render subsystem toggle grid + typed config detail
- File issues on peer repos for any needed upstream changes

Out of scope: defining report data types, building report rendering, editing peer repos.

## Architecture

### PatternViewMixin Tab Extension

PatternViewMixin (`packages/agent-viewer/src/shared-styles.ts`) gains:

- `_activeTab: 'execution' | 'reports'` — reactive state, defaults to `'execution'`
- `renderTabBar(): TemplateResult` — ARIA `role="tablist"` with two `role="tab"` buttons,
  `aria-selected` on active, arrow-key navigation (left/right cycle), matching
  detail-pane tab convention
- `renderTabPanel(): TemplateResult` — `role="tabpanel"` with `aria-labelledby` pointing
  to active tab, dispatches to `renderExecution()` or `renderReports()`
- `renderExecution(): TemplateResult` — override point, default returns
  `renderEmptyState('No execution data')`
- `renderReports(): TemplateResult` — override point, default returns
  `renderEmptyState('No report data available')`
- Tab styles added to `patternViewStyles` — consistent tab bar across all pattern views

Each view's `render()` becomes: `renderTabBar()` + `renderTabPanel()`. Existing render
body moves into `renderExecution()`. Reports returns stub placeholder.

### CognitionConfig Type Rework

Replace in `packages/mosaic-core/src/pattern-categories/social.ts`:

**Before:**
```typescript
interface CognitionConfig {
  drive?: Record<string, unknown>;
  mood?: Record<string, unknown>;
  retention?: Record<string, unknown>;
  // ... 10 more opaque modules
}
```

**After:**
```typescript
interface CognitionConfig {
  // ~23 boolean toggles matching Java CognitionConfig
  attentionEnabled?: boolean;
  moodEnabled?: boolean;
  drivesEnabled?: boolean;
  reflectionEnabled?: boolean;
  consolidationEnabled?: boolean;
  entityKnowledgeEnabled?: boolean;
  perspectiveComparisonEnabled?: boolean;
  domainActivationEnabled?: boolean;
  salienceEnabled?: boolean;
  behavioralEnabled?: boolean;
  temporalFocusEnabled?: boolean;
  subThoughtsEnabled?: boolean;
  characterDrivesEnabled?: boolean;
  needsPyramidEnabled?: boolean;
  memoryHygieneEnabled?: boolean;
  mentalModelEnabled?: boolean;
  userModelEnabled?: boolean;
  strategyEnabled?: boolean;
  narrativeEnabled?: boolean;
  innerLifeEnabled?: boolean;
  personalityEvolutionEnabled?: boolean;
  beliefRevisionEnabled?: boolean;
  goalsEnabled?: boolean;
}

// Per-subsystem typed configs (for subsystems that have Java config beans)
interface MoodConfig {
  baselineValence?: number;
  baselineArousal?: number;
  baselineDominance?: number;
  decayTimeConstant?: number;
  maxDisplacement?: number;
  moodInfluence?: number;
  staleStateTimeout?: string; // Duration
}

interface DriveConfig {
  axisWeights?: Record<string, number>; // DriveAxis -> weight
  changeThreshold?: number;
  // modulation params
}

interface BeliefRevisionConfig {
  enabled?: boolean;
  // 3 thresholds
}

// Additional per-subsystem types: CognitiveGoalConfig, GoalProposalConfig,
// GoalEscalationConfig, MentalModelConfig, UserModelConfig,
// StrategyLearningConfig, NarrativeConfig, InnerLifeConfig,
// PersonalityEvolutionConfig, NormDetectionConfig, MoodCongruenceConfig,
// SchererAppraisalConfig
```

SocialPatternCategory updated: `cognition` field becomes `CognitionConfig` (toggles)
with optional per-subsystem config properties alongside it.

### Per-View Rework

| View | Execution tab | Reports tab | Scope |
|------|--------------|-------------|-------|
| summarisation-view | Existing PipelineSpec rendering (no changes) | Stub placeholder | Migration only |
| observation-view | Existing WorldSpec rendering (no changes) | Stub placeholder | Migration only |
| narrative-view | Existing PipelineSpec + stage labels (no changes) | Stub placeholder | Migration only |
| prompt-optimisation-view | Existing PromptOptimisationPipeline rendering (no changes) | Stub placeholder | Migration only |
| cognition-view | Full rework: subsystem toggle grid + typed config detail | Stub placeholder | Major rework |
| memory-hygiene-view | Existing RetentionConfig rendering (no changes) | Stub placeholder | Migration only |

### cognition-view Rework Detail

Replaces generic module card grid with:

1. **Subsystem toggle grid** — each of the ~23 subsystems as a card showing
   enabled/disabled badge. Subsystems with typed configs are expandable.
2. **Config detail** — for subsystems with typed config beans (MoodConfig,
   DriveConfig, etc.), expanding a card shows the config fields with appropriate
   rendering (numbers, durations, weight maps).
3. **Toggle-only subsystems** (~10: attention, temporalFocus, etc.) — show
   on/off badge only, no expandable detail.

## Testing

- Each view test file gains ARIA assertions for tab structure (tablist, tab, tabpanel roles)
- Existing rendering assertions move under "Execution tab" describe blocks
- cognition-view tests: subsystem grid rendering, toggle badges, config detail expansion
- PatternViewMixin tests: tab switching, keyboard navigation, default empty states

## Gallery

Update `gallery.html` demo data for cognition-view to use new CognitionConfig shape
(boolean toggles + subsystem configs instead of opaque Record modules).

## Cross-Repo Issues

File issues on peer repos for:
- Any subsystem config beans whose Java fields could not be fully audited
- Report type definitions when they become available (perception reports,
  optimisation scores, etc.)

## Limitations

- Reports tabs are stubs — actual report rendering blocked on TS type definitions
  for runtime data (StepDecisionSummary, DecisionNarrative, optimisation scores, etc.)
- CognitionConfig toggle list based on Java audit — may need updating if neocortex
  adds/removes subsystems
- Per-subsystem config detail rendering is best-effort based on available Java field
  information; some config types may have incomplete field lists

## References

- `packages/agent-viewer/src/shared-styles.ts` — PatternViewMixin current implementation
- `packages/mosaic-core/src/pattern-categories.ts` — current sub-spec types
- `components/detail-pane/src/detail-pane.ts` — TabDefinition ARIA convention
- neocortex/cognition-api `CognitionConfig.java` — real toggle structure
- neocortex `CognitionDefaultBeans.java` — per-subsystem config bean types
- casehubio/casehub-mosaic#287 — follow-up for enriching cognition-view
- Slot 219 commits — prior type alignment work
