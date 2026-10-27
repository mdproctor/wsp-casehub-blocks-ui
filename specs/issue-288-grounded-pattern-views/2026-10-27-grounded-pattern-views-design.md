# Design: Grounded Pattern Views — Execution/Reports Architecture

**Issue:** casehubio/casehub-mosaic#288
**Date:** 2026-10-27
**Branch:** issue-288-grounded-pattern-views

## Problem

Wave 3 pattern views (perception, optimisation, social) render configuration data
but have no architectural separation between what a system IS configured to do
(Execution) and what it DID (Reports). Additionally, CognitionConfig in mosaic-core
has the right 13-section structure (mirroring `CognitionDefinition` from
`agency/agentic-yaml`) but each section is typed as opaque `Record<string, unknown>`
instead of having typed fields matching the Java `*ConfigSpec` records.

## Scope

- Add Execution/Reports tab architecture to PatternViewMixin
- Extract shared header CSS into `patternViewStyles` (currently duplicated across 6 views)
- In-place rework of 6 views with Execution tab rendering real config
- Reports tabs render stub placeholders (report TS types do not exist yet)
- Type CognitionConfig's 13 sections to match `CognitionDefinition` from `agency/agentic-yaml`
- Full rework of cognition-view to render typed config sections
- File issues on peer repos for any needed upstream changes
- Close #287 (this work fulfils its acceptance criteria)

Out of scope: defining report data types, building report rendering, editing peer repos.

## Architecture

### PatternViewMixin Tab Extension

PatternViewMixin (`packages/agent-viewer/src/shared-styles.ts`) gains:

- `_activeTab: 'execution' | 'reports'` — reactive state, defaults to `'execution'`
- `renderTabBar(): TemplateResult` — ARIA `role="tablist"` with two `role="tab"` buttons.
  Full ARIA compliance matching detail-pane convention:
  - `aria-selected` on active tab
  - `aria-controls="panel-execution"` / `aria-controls="panel-reports"` on tab buttons
  - `id="tab-execution"` / `id="tab-reports"` on tab buttons
  - Roving `tabindex` (`0` on active, `-1` on inactive)
  - Arrow key navigation (left/right cycle)
  - Home/End keys to jump to first/last tab
- `renderTabPanel(): TemplateResult` — `role="tabpanel"` with `id="panel-${tab}"`
  and `aria-labelledby="tab-${tab}"`, dispatches to `renderExecution()` or `renderReports()`
- `renderExecution(): TemplateResult` — override point, default returns
  `renderEmptyState('No execution data')`
- `renderReports(): TemplateResult` — override point, default returns
  `renderEmptyState('No report data available')`
- Shared header styles extracted from views into `patternViewStyles` — `.header`,
  `.header-label`, `.header-name`, `.type-badge`, `.header-desc`. Per-category badge
  colour via `--pattern-badge-color` CSS custom property.
- Tab styles added to `patternViewStyles` — consistent tab bar across all pattern views

Each view's `render()` becomes: `renderTabBar()` + `renderTabPanel()`. Existing render
body moves into `renderExecution()`. Reports returns stub placeholder. Views set
`--pattern-badge-color` for their category.

**Nested tab bar:** These Execution/Reports tabs render inside `agent-viewer`'s
Pattern tab content area, creating a two-level tab hierarchy. This is intentional —
the outer tabs (Eidos/Pattern/Steps) select the content domain, the inner tabs
select the data perspective. Keyboard focus is scoped: arrow keys within each tab
bar navigate that level only. The `role="region"` on the pattern view host provides
the ARIA boundary between the two tab levels.

### CognitionConfig Type Rework

**Data source clarification:** The frontend receives YAML-derived data shaped as
`CognitionDefinition` (from `agency/agentic-yaml`), NOT `CognitionConfig.java`
(neocortex CDI runtime toggles). The existing mosaic-core `CognitionConfig` with
13 `Record<string, unknown>` sections is structurally correct — it mirrors
`CognitionDefinition`'s 13 optional `*ConfigSpec` sections. The fix is typing
those sections, not replacing with boolean toggles.

Replace in `packages/mosaic-core/src/types/patterns/social.ts`:

**Before:**
```typescript
interface CognitionConfig {
  drive?: Record<string, unknown>;
  mood?: Record<string, unknown>;
  retention?: Record<string, unknown>;
  // ... 10 more opaque modules
}
```

**After (matching CognitionDefinition from agency/agentic-yaml):**
```typescript
interface CognitionConfig {
  drive?: DriveConfigSpec;
  mood?: MoodConfigSpec;
  personality?: PersonalityEvolutionConfigSpec;
  userModel?: UserModelConfigSpec;
  strategyLearning?: StrategyLearningConfigSpec;
  mentalModel?: MentalModelConfigSpec;
  narrative?: NarrativeConfigSpec;
  goalProposal?: GoalProposalConfigSpec;
  goalEscalation?: GoalEscalationConfigSpec;
  normDetection?: NormDetectionConfigSpec;
  collectiveGoal?: CollectiveGoalConfigSpec;
  retention?: RetentionConfigSpec;
  cognitiveGoal?: CognitiveGoalConfigSpec;
}

interface MoodConfigSpec {
  baseline?: MoodBaseline;
  decayTimeConstant?: string;   // Duration
  maxDisplacement?: number;
  moodInfluence?: number;
  staleStateTimeout?: string;   // Duration
}

interface MoodBaseline {
  pleasure?: number;
  arousal?: number;
  dominance?: number;
}

interface DriveConfigSpec {
  axisWeights?: Record<string, number>;  // DriveAxis -> weight
  changeThreshold?: number;
  moodPleasureModulation?: number;
  moodArousalModulation?: number;
  personalityModulationStrength?: number;
  maxIntensity?: number;
  minIntensity?: number;
  affiliationDecayThreshold?: number;
  affiliationStaleDuration?: string;     // Duration
  autonomyConfidenceFloor?: number;
  narrativeModulationStrength?: number;
}

interface PersonalityEvolutionConfigSpec {
  decayFactor?: number;
  l2Ceiling?: number;
  dampeningFactor?: number;
}

interface UserModelConfigSpec {
  minSignalsForSynthesis?: number;
  synthesisCooldown?: string;            // Duration
  decayRate?: number;
  positiveWeight?: number;
  negativeWeight?: number;
  stageConfig?: RelationshipStageConfigSpec;
  expectedTickInterval?: string;         // Duration
  evictionTimeout?: string;              // Duration
  memoryDomain?: string;
  caseType?: string;
  maxObservationsInPrompt?: number;
}

interface StrategyLearningConfigSpec {
  minSignalsForConversationCase?: number;
  minCasesForReflection?: number;
  maxReflectionSources?: number;
  maxGuidelines?: number;
  defaultDimensionValue?: number;
  maxBufferSize?: number;
  staleStateTimeout?: string;            // Duration
  memoryDomain?: string;
  engagementCaseType?: string;
  profileCaseType?: string;
}

interface MentalModelConfigSpec {
  beliefHalfLife?: string;               // Duration
  desireHalfLife?: string;               // Duration
  intentionHalfLife?: string;            // Duration
  confidenceFloor?: number;
  projectionFloor?: number;
  minSignalsForInference?: number;
  inferenceCooldown?: string;            // Duration
  maxSignalsInPrompt?: number;
  maxBufferSize?: number;
  evictionTimeout?: string;              // Duration
  expectedTickInterval?: string;         // Duration
  memoryDomain?: string;
  caseType?: string;
}

interface NarrativeConfigSpec {
  synthesisGate?: NarrativeSynthesisGateSpec;
  maxEpisodes?: number;
  maxThemes?: number;
  themeSalienceFloor?: number;
  maxReflectionsPerSynthesis?: number;
  memoryDomain?: string;
  caseType?: string;
}

interface NarrativeSynthesisGateSpec {
  minNewReflections?: number;
  noveltyThreshold?: number;
  quietPeriodBypass?: string;            // Duration
}

interface GoalProposalConfigSpec {
  proposalThreshold?: number;
  relevanceThreshold?: number;
  maxDriveGoals?: number;
  staleAfter?: string;                   // Duration
  cooldown?: string;                     // Duration
  failureAbandonmentThreshold?: number;
}

interface GoalEscalationConfigSpec {
  escalationSalienceThreshold?: number;
  minAxisAlignmentWeight?: number;
  crossAxisMinWeight?: number;
  minCrossAxisCount?: number;
  escalationCycles?: number;
  demotionCycles?: number;
  maxPrimaryDriveGoals?: number;
}

interface CognitiveGoalConfigSpec {
  enabled?: boolean;
  caseCreationThreshold?: number;
  surfacingCooldown?: string;            // Duration
  driveWeight?: number;
  minimumSurfacingPriority?: number;
}

interface NormDetectionConfigSpec {
  minObservationsForNorm?: number;
  establishedThreshold?: number;
  decliningThreshold?: number;
  minAgentsForNorm?: number;
  memoryDomain?: string;
  caseType?: string;
}

interface CollectiveGoalConfigSpec {
  alignmentThreshold?: number;
  minAlignedAgents?: number;
  cooldown?: string;                     // Duration
}

// RetentionConfigSpec already exists in mosaic-core — no changes needed
```

Helper type `RelationshipStageConfigSpec` (used by `UserModelConfigSpec`) will be
typed as `Record<string, unknown>` initially — the full tier structure can be
refined in a follow-up if needed.

### Per-View Rework

| View | Execution tab | Reports tab | Scope |
|------|--------------|-------------|-------|
| summarisation-view | Existing PipelineSpec rendering (no changes) | Stub placeholder | Migration only |
| observation-view | Existing WorldSpec rendering (no changes) | Stub placeholder | Migration only |
| narrative-view | Existing PipelineSpec + stage labels (no changes) | Stub placeholder | Migration only |
| prompt-optimisation-view | Existing PromptOptimisationPipeline rendering (no changes) | Stub placeholder | Migration only |
| cognition-view | Full rework: section grid with typed config detail | Stub placeholder | Major rework |
| memory-hygiene-view | Existing RetentionConfig rendering (no changes) | Stub placeholder | Migration only |

### cognition-view Rework Detail

Replaces generic module card grid with:

1. **Section grid** — each of the 13 `CognitionDefinition` sections as a card.
   Present sections show a config summary; absent (null) sections show as inactive/unconfigured.
2. **Config detail** — expanding a section card shows its typed fields with appropriate
   rendering: numbers as values, Durations as human-readable strings, weight maps as
   small bar charts or key-value lists, nested objects (MoodBaseline, NarrativeSynthesisGateSpec)
   as sub-sections.
3. **Section categories** — group sections visually: Drives & Mood (drive, mood),
   Models & Learning (userModel, mentalModel, strategyLearning, personality),
   Goals (goalProposal, goalEscalation, cognitiveGoal, collectiveGoal),
   Memory & Norms (retention, narrative, normDetection).

## Testing

- Each view test file gains ARIA assertions for tab structure:
  - `role="tablist"` present
  - Two `role="tab"` buttons with correct `aria-selected`, `aria-controls`, `tabindex`
  - `role="tabpanel"` with correct `id` and `aria-labelledby`
- Existing rendering assertions move under "Execution tab" describe blocks
- cognition-view tests: section grid rendering, config detail expansion, absent section display
- PatternViewMixin tests: tab switching, keyboard navigation (arrows, Home/End), default empty states

## Gallery

Update `gallery.html` demo data for all 6 views:
- cognition-view: new CognitionConfig shape (typed sections instead of opaque Records)
- All 6 views: verify tab bar renders correctly with existing demo data (migration-only
  views need no data changes — the tab wraps existing content)

## Cross-Repo Issues

Specific issues to file:

1. **agency** (casehubio/blocks) — request REST/GraphQL projection that serves
   `CognitionDefinition` data to the frontend (if not already available via the
   pattern spec endpoint)
2. **neocortex** — document which `CognitionConfig.java` toggles correspond to
   which `CognitionDefinition` sections (the mapping is implicit today)

Report type issues (deferred — no action in this issue):
- Perception reports: `StepDecisionSummary`, `DecisionNarrative` TS types
- Optimisation reports: scoring/variant selection result types
- Social reports: cognition runtime output types

## Issue #287

This work fulfils #287's acceptance criteria:
- SocialPatternCategory gains cognition-specific typed fields (13 `*ConfigSpec` interfaces)
- cognition-view renders the typed fields (section grid with config detail)
- Gallery updated with cognition-specific demo data

Close #287 when this branch lands.

## Limitations

- Reports tabs are stubs — actual report rendering blocked on TS type definitions
  for runtime data
- `RelationshipStageConfigSpec` (nested in UserModelConfigSpec) typed as
  `Record<string, unknown>` initially — full tier structure deferred
- CognitionDefinition section list based on Java audit — may need updating if
  agency adds/removes sections

## References

- `packages/agent-viewer/src/shared-styles.ts` — PatternViewMixin current implementation
- `packages/mosaic-core/src/types/patterns/social.ts` — current CognitionConfig type
- `packages/mosaic-core/src/types/patterns/index.ts` — pattern type exports
- `components/detail-pane/src/detail-pane.ts` — TabDefinition ARIA convention
- `agency/agentic-yaml` `CognitionDefinition.java` — authoritative YAML schema type
- `agency/agentic-yaml` `*ConfigSpec.java` records — per-section field definitions
- neocortex/cognition-api `CognitionConfig.java` — CDI runtime toggles (NOT the frontend type)
- casehubio/casehub-mosaic#287 — closed by this work
- Slot 219 commits — prior type alignment work
- Design review: `/Users/mdproctor/reviews/casehub-casehub-mosaic/issue-288-grounded-pattern-views-20261027-060119/`
