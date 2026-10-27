# Grounded Pattern Views Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #288 — Grounded pattern views: Execution/Reports architecture
**Issue group:** #288, #287 (closed by this work)

**Goal:** Add Execution/Reports tab architecture to all pattern views, type
CognitionConfig's 13 sections to match Java CognitionDefinition, and rework
cognition-view to render typed config detail.

**Architecture:** PatternViewMixin gains tab rendering (renderTabBar/renderTabPanel)
with override methods (renderExecution/renderReports). Views move existing render
body into renderExecution(). CognitionConfig sections get typed interfaces matching
agency/agentic-yaml *ConfigSpec records. Shared header CSS extracted to mixin.

**Tech Stack:** Lit 3, TypeScript, Vitest, Web Components

## Global Constraints

- All Duration fields in Java map to `string` in TypeScript
- ARIA compliance: roving tabindex, aria-controls, aria-labelledby, Home/End keys
- No `Record<string, unknown>` — every config section fully typed
- Pre-release: no deprecation markers, re-exports, or shims
- `RetentionConfig` already exists and is unchanged
- `RelationshipStageConfigSpec` typed as `Record<string, unknown>` initially

---

## Batch 1: Foundation — Tab infrastructure and typed CognitionConfig

### Task 1: Type CognitionConfig sections in mosaic-core

**Files:**
- Modify: `packages/mosaic-core/src/types/patterns/social.ts`
- Test: `packages/mosaic-core/src/types/patterns/social.test.ts` (create)

**Interfaces:**
- Produces: `DriveConfigSpec`, `MoodConfigSpec`, `MoodBaseline`,
  `PersonalityEvolutionConfigSpec`, `UserModelConfigSpec`,
  `StrategyLearningConfigSpec`, `MentalModelConfigSpec`, `NarrativeConfigSpec`,
  `NarrativeSynthesisGateSpec`, `GoalProposalConfigSpec`,
  `GoalEscalationConfigSpec`, `CognitiveGoalConfigSpec`,
  `NormDetectionConfigSpec`, `CollectiveGoalConfigSpec` — all exported from
  `social.ts`. `CognitionConfig` updated to use these instead of
  `Record<string, unknown>`.

- [ ] **Step 1: Write type assertion test**

Create `packages/mosaic-core/src/types/patterns/social.test.ts`:

```typescript
import { describe, it, expectTypeOf } from 'vitest';
import type {
  CognitionConfig, MoodConfigSpec, DriveConfigSpec,
  PersonalityEvolutionConfigSpec, NarrativeConfigSpec,
  GoalProposalConfigSpec, CognitiveGoalConfigSpec,
  SocialPatternCategory,
} from './social.js';

describe('CognitionConfig types', () => {
  it('cognition sections are typed interfaces, not Record<string, unknown>', () => {
    const config: CognitionConfig = {
      mood: {
        baseline: { pleasure: 0, arousal: 0, dominance: 0 },
        decayTimeConstant: 'PT4H',
        maxDisplacement: 1.0,
        moodInfluence: 0.5,
        staleStateTimeout: 'PT24H',
      },
      drive: {
        axisWeights: { curiosity: 0.8 },
        changeThreshold: 0.1,
        maxIntensity: 1.0,
        minIntensity: 0.0,
      },
      retention: { retentionThreshold: 0.6, confidenceWeight: 0.4 },
    };
    expectTypeOf(config.mood).toEqualTypeOf<MoodConfigSpec | undefined>();
    expectTypeOf(config.drive).toEqualTypeOf<DriveConfigSpec | undefined>();
  });

  it('SocialPatternCategory.cognition uses typed CognitionConfig', () => {
    expectTypeOf<SocialPatternCategory['cognition']>()
      .toEqualTypeOf<CognitionConfig | undefined>();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/mosaic-core/src/types/patterns/social.test.ts`
Expected: FAIL — `MoodConfigSpec` and `DriveConfigSpec` not exported

- [ ] **Step 3: Replace Record<string, unknown> with typed interfaces**

In `packages/mosaic-core/src/types/patterns/social.ts`, replace the 12
`Record<string, unknown>` fields on `CognitionConfig` with typed interfaces.
Keep `RetentionConfig` unchanged (already typed). Add all 13 `*ConfigSpec`
interfaces after `CognitionConfig`.

Full type definitions (matching CognitionDefinition from agency/agentic-yaml):

```typescript
export interface MoodBaseline {
  pleasure?: number;
  arousal?: number;
  dominance?: number;
}

export interface MoodConfigSpec {
  baseline?: MoodBaseline;
  decayTimeConstant?: string;
  maxDisplacement?: number;
  moodInfluence?: number;
  staleStateTimeout?: string;
}

export interface DriveConfigSpec {
  axisWeights?: Record<string, number>;
  changeThreshold?: number;
  moodPleasureModulation?: number;
  moodArousalModulation?: number;
  personalityModulationStrength?: number;
  maxIntensity?: number;
  minIntensity?: number;
  affiliationDecayThreshold?: number;
  affiliationStaleDuration?: string;
  autonomyConfidenceFloor?: number;
  narrativeModulationStrength?: number;
}

export interface PersonalityEvolutionConfigSpec {
  decayFactor?: number;
  l2Ceiling?: number;
  dampeningFactor?: number;
}

export interface UserModelConfigSpec {
  minSignalsForSynthesis?: number;
  synthesisCooldown?: string;
  decayRate?: number;
  positiveWeight?: number;
  negativeWeight?: number;
  stageConfig?: Record<string, unknown>;
  expectedTickInterval?: string;
  evictionTimeout?: string;
  memoryDomain?: string;
  caseType?: string;
  maxObservationsInPrompt?: number;
}

export interface StrategyLearningConfigSpec {
  minSignalsForConversationCase?: number;
  minCasesForReflection?: number;
  maxReflectionSources?: number;
  maxGuidelines?: number;
  defaultDimensionValue?: number;
  maxBufferSize?: number;
  staleStateTimeout?: string;
  memoryDomain?: string;
  engagementCaseType?: string;
  profileCaseType?: string;
}

export interface MentalModelConfigSpec {
  beliefHalfLife?: string;
  desireHalfLife?: string;
  intentionHalfLife?: string;
  confidenceFloor?: number;
  projectionFloor?: number;
  minSignalsForInference?: number;
  inferenceCooldown?: string;
  maxSignalsInPrompt?: number;
  maxBufferSize?: number;
  evictionTimeout?: string;
  expectedTickInterval?: string;
  memoryDomain?: string;
  caseType?: string;
}

export interface NarrativeSynthesisGateSpec {
  minNewReflections?: number;
  noveltyThreshold?: number;
  quietPeriodBypass?: string;
}

export interface NarrativeConfigSpec {
  synthesisGate?: NarrativeSynthesisGateSpec;
  maxEpisodes?: number;
  maxThemes?: number;
  themeSalienceFloor?: number;
  maxReflectionsPerSynthesis?: number;
  memoryDomain?: string;
  caseType?: string;
}

export interface GoalProposalConfigSpec {
  proposalThreshold?: number;
  relevanceThreshold?: number;
  maxDriveGoals?: number;
  staleAfter?: string;
  cooldown?: string;
  failureAbandonmentThreshold?: number;
}

export interface GoalEscalationConfigSpec {
  escalationSalienceThreshold?: number;
  minAxisAlignmentWeight?: number;
  crossAxisMinWeight?: number;
  minCrossAxisCount?: number;
  escalationCycles?: number;
  demotionCycles?: number;
  maxPrimaryDriveGoals?: number;
}

export interface CognitiveGoalConfigSpec {
  enabled?: boolean;
  caseCreationThreshold?: number;
  surfacingCooldown?: string;
  driveWeight?: number;
  minimumSurfacingPriority?: number;
}

export interface NormDetectionConfigSpec {
  minObservationsForNorm?: number;
  establishedThreshold?: number;
  decliningThreshold?: number;
  minAgentsForNorm?: number;
  memoryDomain?: string;
  caseType?: string;
}

export interface CollectiveGoalConfigSpec {
  alignmentThreshold?: number;
  minAlignedAgents?: number;
  cooldown?: string;
}
```

Update `CognitionConfig` to use these:

```typescript
export interface CognitionConfig {
  drive?: DriveConfigSpec;
  mood?: MoodConfigSpec;
  personalityEvolution?: PersonalityEvolutionConfigSpec;
  userModel?: UserModelConfigSpec;
  strategyLearning?: StrategyLearningConfigSpec;
  mentalModel?: MentalModelConfigSpec;
  narrative?: NarrativeConfigSpec;
  goalProposal?: GoalProposalConfigSpec;
  goalEscalation?: GoalEscalationConfigSpec;
  normDetection?: NormDetectionConfigSpec;
  collectiveGoal?: CollectiveGoalConfigSpec;
  retention?: RetentionConfig;
  cognitiveGoal?: CognitiveGoalConfigSpec;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn vitest run packages/mosaic-core/src/types/patterns/social.test.ts`
Expected: PASS

- [ ] **Step 5: Run full typecheck to catch downstream breakage**

Run: `yarn typecheck`
Expected: PASS — cognition-view.ts uses `keyof CognitionConfig` and
`cognition[key as keyof CognitionConfig]` which should still work since the
key set is unchanged (only value types changed).

- [ ] **Step 6: Commit**

```bash
git add packages/mosaic-core/src/types/patterns/social.ts packages/mosaic-core/src/types/patterns/social.test.ts
git commit -m "feat(mosaic-core): type CognitionConfig sections to match CognitionDefinition

Replace Record<string, unknown> with typed *ConfigSpec interfaces
matching agency/agentic-yaml CognitionDefinition. 13 section types
fully specified with fields from Java audit.

Refs #288"
```

---

### Task 2: Add tab infrastructure to PatternViewMixin

**Files:**
- Modify: `packages/agent-viewer/src/shared-styles.ts`
- Modify: `packages/agent-viewer/src/shared-styles.test.ts`

**Interfaces:**
- Consumes: `PatternCategory` from mosaic-core
- Produces: `renderTabBar()`, `renderTabPanel()`, `renderExecution()`,
  `renderReports()` methods on PatternViewMixin. `_activeTab` reactive state.
  Shared header styles in `patternViewStyles` (`.header`, `.header-label`,
  `.header-name`, `.type-badge`, `.header-desc`) with `--pattern-badge-color`
  CSS custom property.

- [ ] **Step 1: Write failing tests for tab infrastructure**

Add to `packages/agent-viewer/src/shared-styles.test.ts`:

```typescript
// Add new test element that uses tabs
@customElement('test-tabbed-pattern-view')
class TestTabbedPatternView extends PatternViewMixin(LitElement) {
  static override styles = [patternViewStyles];
  override renderExecution() {
    return html`<div class="exec-content">execution content</div>`;
  }
  override renderReports() {
    return html`<div class="reports-content">reports content</div>`;
  }
  override render() {
    if (!this.spec) return this.renderEmptyState();
    return html`${this.renderTabBar()}${this.renderTabPanel()}`;
  }
}

describe('PatternViewMixin tabs', () => {
  let el: TestTabbedPatternView;
  afterEach(() => el?.remove());

  async function create() {
    el = document.createElement('test-tabbed-pattern-view') as TestTabbedPatternView;
    el.spec = { category: 'deliberation', type: 'voting', name: 'test' };
    document.body.appendChild(el);
    await el.updateComplete;
  }

  it('renders tablist with two tabs', async () => {
    await create();
    const tablist = el.shadowRoot!.querySelector('[role="tablist"]');
    expect(tablist).toBeTruthy();
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    expect(tabs.length).toBe(2);
  });

  it('Execution tab selected by default', async () => {
    await create();
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    expect(tabs[0].getAttribute('aria-selected')).toBe('true');
    expect(tabs[1].getAttribute('aria-selected')).toBe('false');
  });

  it('tabs have aria-controls linking to panel', async () => {
    await create();
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    expect(tabs[0].getAttribute('aria-controls')).toBe('panel-execution');
    expect(tabs[1].getAttribute('aria-controls')).toBe('panel-reports');
  });

  it('tabpanel has id and aria-labelledby', async () => {
    await create();
    const panel = el.shadowRoot!.querySelector('[role="tabpanel"]');
    expect(panel!.getAttribute('id')).toBe('panel-execution');
    expect(panel!.getAttribute('aria-labelledby')).toBe('tab-execution');
  });

  it('renders execution content by default', async () => {
    await create();
    expect(el.shadowRoot!.querySelector('.exec-content')?.textContent).toBe('execution content');
    expect(el.shadowRoot!.querySelector('.reports-content')).toBeNull();
  });

  it('switches to reports tab on click', async () => {
    await create();
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    (tabs[1] as HTMLElement).click();
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.reports-content')?.textContent).toBe('reports content');
    expect(el.shadowRoot!.querySelector('.exec-content')).toBeNull();
  });

  it('roving tabindex: active tab has 0, inactive has -1', async () => {
    await create();
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    expect(tabs[0].getAttribute('tabindex')).toBe('0');
    expect(tabs[1].getAttribute('tabindex')).toBe('-1');
  });

  it('arrow right moves to next tab', async () => {
    await create();
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    (tabs[0] as HTMLElement).focus();
    tabs[0].dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowRight', bubbles: true }));
    await el.updateComplete;
    expect(tabs[1].getAttribute('aria-selected')).toBe('true');
  });

  it('Home key moves to first tab', async () => {
    await create();
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    (tabs[1] as HTMLElement).click();
    await el.updateComplete;
    tabs[1].dispatchEvent(new KeyboardEvent('keydown', { key: 'Home', bubbles: true }));
    await el.updateComplete;
    expect(tabs[0].getAttribute('aria-selected')).toBe('true');
  });

  it('End key moves to last tab', async () => {
    await create();
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    tabs[0].dispatchEvent(new KeyboardEvent('keydown', { key: 'End', bubbles: true }));
    await el.updateComplete;
    expect(tabs[1].getAttribute('aria-selected')).toBe('true');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/agent-viewer/src/shared-styles.test.ts`
Expected: FAIL — `renderTabBar`, `renderTabPanel`, `renderExecution`, `renderReports` do not exist

- [ ] **Step 3: Implement tab infrastructure in PatternViewMixin**

In `packages/agent-viewer/src/shared-styles.ts`:

1. Add `@state() _activeTab: 'execution' | 'reports' = 'execution'` to the mixin class
2. Add `renderTabBar()` — renders `role="tablist"` div with two buttons
3. Add `renderTabPanel()` — renders `role="tabpanel"` dispatching to renderExecution/renderReports
4. Add `renderExecution()` and `renderReports()` default implementations
5. Add `_onTabClick(tab)` and `_onTabKeydown(e)` handlers
6. Add shared header CSS and tab CSS to `patternViewStyles`
7. Import `state` from `lit/decorators.js`

```typescript
import { css, html, nothing, type CSSResult, type TemplateResult } from 'lit';
import { property, state } from 'lit/decorators.js';
import type { LitElement } from 'lit';
import type { PatternCategory } from '@casehubio/mosaic-core';

type Constructor<T = {}> = new (...args: any[]) => T;

export const patternViewStyles: CSSResult = css`
  :host {
    display: block;
    background: rgba(255, 255, 255, 0.02);
    border: 1px solid rgba(255, 255, 255, 0.08);
    border-radius: 8px;
    overflow: hidden;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
    color: #cdd6f4;
  }

  .empty-state {
    padding: 32px;
    text-align: center;
    color: #7f849c;
    font-size: 13px;
  }

  .header {
    padding: 10px 16px;
    border-bottom: 1px solid rgba(255, 255, 255, 0.08);
    display: flex;
    align-items: center;
    gap: 8px;
  }
  .header-label {
    font-size: 12px;
    font-weight: 600;
    color: #7f849c;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }
  .header-name {
    font-size: 13px;
    font-weight: 500;
    color: #f0f4ff;
  }
  .type-badge {
    padding: 1px 7px;
    border-radius: 4px;
    font-size: 11px;
    font-weight: 500;
    background: var(--pattern-badge-color, rgba(137, 180, 250, 0.2));
    color: var(--pattern-badge-text, #89b4fa);
  }
  .header-desc {
    margin-left: auto;
    font-size: 11px;
    color: #7f849c;
    max-width: 400px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .tab-bar {
    display: flex;
    border-bottom: 1px solid rgba(255, 255, 255, 0.08);
    padding: 0 16px;
  }
  .tab-button {
    background: none;
    border: none;
    border-bottom: 2px solid transparent;
    padding: 8px 16px;
    font-size: 12px;
    font-weight: 500;
    color: #7f849c;
    cursor: pointer;
    font-family: inherit;
    transition: color 0.15s, border-color 0.15s;
  }
  .tab-button:hover { color: #cdd6f4; }
  .tab-button[aria-selected="true"] {
    color: #cdd6f4;
    border-bottom-color: #89b4fa;
  }
  .tab-button:focus-visible {
    outline: 2px solid #89b4fa;
    outline-offset: -2px;
    border-radius: 2px;
  }
`;

export const PatternViewMixin = <T extends Constructor<LitElement>>(Base: T) => {
  class PatternViewHost extends Base {
    @property({ attribute: false }) spec!: PatternCategory;
    @state() _activeTab: 'execution' | 'reports' = 'execution';

    override connectedCallback() {
      super.connectedCallback();
      this.setAttribute('role', 'region');
      this.setAttribute('aria-label', `${this.spec?.category ?? 'pattern'} view`);
    }

    protected setLoading(loading: boolean) {
      this.setAttribute('aria-busy', String(loading));
    }

    protected renderEmptyState(message = 'No pattern data'): TemplateResult {
      return html`<div class="empty-state">${message}</div>`;
    }

    protected renderTabBar(): TemplateResult {
      const tabs: Array<{ id: 'execution' | 'reports'; label: string }> = [
        { id: 'execution', label: 'Execution' },
        { id: 'reports', label: 'Reports' },
      ];
      return html`
        <div class="tab-bar" role="tablist">
          ${tabs.map(t => html`
            <button
              role="tab"
              id="tab-${t.id}"
              aria-selected="${String(this._activeTab === t.id)}"
              aria-controls="panel-${t.id}"
              tabindex="${this._activeTab === t.id ? 0 : -1}"
              @click=${() => this._switchTab(t.id)}
              @keydown=${this._onTabKeydown}
              class="tab-button"
            >${t.label}</button>
          `)}
        </div>
      `;
    }

    protected renderTabPanel(): TemplateResult {
      return html`
        <div
          role="tabpanel"
          id="panel-${this._activeTab}"
          aria-labelledby="tab-${this._activeTab}"
        >
          ${this._activeTab === 'execution'
            ? this.renderExecution()
            : this.renderReports()}
        </div>
      `;
    }

    protected renderExecution(): TemplateResult {
      return this.renderEmptyState('No execution data');
    }

    protected renderReports(): TemplateResult {
      return this.renderEmptyState('No report data available');
    }

    private _switchTab(tab: 'execution' | 'reports') {
      this._activeTab = tab;
      this.requestUpdate();
    }

    private _onTabKeydown(e: KeyboardEvent) {
      const order: Array<'execution' | 'reports'> = ['execution', 'reports'];
      const idx = order.indexOf(this._activeTab);
      let next = idx;
      if (e.key === 'ArrowRight') next = (idx + 1) % order.length;
      else if (e.key === 'ArrowLeft') next = (idx - 1 + order.length) % order.length;
      else if (e.key === 'Home') next = 0;
      else if (e.key === 'End') next = order.length - 1;
      else return;
      e.preventDefault();
      this._activeTab = order[next];
      this.requestUpdate();
      this.updateComplete.then(() => {
        const btn = this.shadowRoot?.querySelector(`#tab-${order[next]}`) as HTMLElement;
        btn?.focus();
      });
    }
  }
  return PatternViewHost as Constructor<PatternViewHost> & T;
};
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/agent-viewer/src/shared-styles.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/agent-viewer/src/shared-styles.ts packages/agent-viewer/src/shared-styles.test.ts
git commit -m "feat(agent-viewer): add Execution/Reports tab infrastructure to PatternViewMixin

renderTabBar/renderTabPanel with full ARIA compliance: roving tabindex,
aria-controls, aria-labelledby, arrow/Home/End key navigation.
Shared header CSS extracted to patternViewStyles with --pattern-badge-color.

Refs #288"
```

---

## Batch 2: View migrations — 5 mechanical tab wraps

### Task 3: Migrate perception views (summarisation, observation, narrative)

**Files:**
- Modify: `packages/agent-viewer/src/views/perception/summarisation-view.ts`
- Modify: `packages/agent-viewer/src/views/perception/observation-view.ts`
- Modify: `packages/agent-viewer/src/views/perception/narrative-view.ts`
- Modify: `packages/agent-viewer/src/views/perception/summarisation-view.test.ts`
- Modify: `packages/agent-viewer/src/views/perception/observation-view.test.ts`
- Modify: `packages/agent-viewer/src/views/perception/narrative-view.test.ts`

**Interfaces:**
- Consumes: `renderTabBar()`, `renderTabPanel()`, `renderExecution()` from PatternViewMixin (Task 2)

For each of the 3 perception views, the migration is identical:

- [ ] **Step 1: Add ARIA tab tests to each view's test file**

For each test file (`summarisation-view.test.ts`, `observation-view.test.ts`,
`narrative-view.test.ts`), add these tests:

```typescript
it('renders tablist with Execution and Reports tabs', async () => {
  await create(fixture());
  const tablist = el.shadowRoot!.querySelector('[role="tablist"]');
  expect(tablist).toBeTruthy();
  const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
  expect(tabs.length).toBe(2);
  expect(tabs[0].textContent?.trim()).toBe('Execution');
  expect(tabs[1].textContent?.trim()).toBe('Reports');
});

it('renders execution content by default', async () => {
  await create(fixture());
  const panel = el.shadowRoot!.querySelector('[role="tabpanel"]');
  expect(panel).toBeTruthy();
  expect(panel!.getAttribute('id')).toBe('panel-execution');
});

it('Reports tab shows stub placeholder', async () => {
  await create(fixture());
  const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
  (tabs[1] as HTMLElement).click();
  await (el as any).updateComplete;
  expect(el.shadowRoot!.querySelector('.empty-state')?.textContent).toBe('No report data available');
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/agent-viewer/src/views/perception/`
Expected: FAIL — no tablist rendered

- [ ] **Step 3: Migrate each view**

For each view:
1. Remove duplicated header CSS (`.header`, `.header-label`, `.header-name`,
   `.type-badge`, `.header-desc`) — now in `patternViewStyles`
2. Set `--pattern-badge-color` via host CSS for the perception category colour
3. Rename `render()` to `renderExecution()`
4. Add new `render()` that calls `renderTabBar()` + `renderTabPanel()`

Example for summarisation-view:
```typescript
// Add to styles (host section):
// :host { --pattern-badge-color: rgba(148, 226, 213, 0.2); --pattern-badge-text: #94e2d5; }

// Remove duplicated header CSS

override render() {
  if (!this.spec) return this.renderEmptyState();
  return html`${this.renderTabBar()}${this.renderTabPanel()}`;
}

override renderExecution() {
  const p = this.spec as PerceptionPatternCategory;
  // ... existing render body (header + pipeline rendering)
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/agent-viewer/src/views/perception/`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/agent-viewer/src/views/perception/
git commit -m "feat(agent-viewer): add Execution/Reports tabs to perception views

Migrate summarisation-view, observation-view, narrative-view to tab
architecture. Extract shared header CSS to mixin. Reports tab stubs.

Refs #288"
```

---

### Task 4: Migrate optimisation and memory-hygiene views

**Files:**
- Modify: `packages/agent-viewer/src/views/optimisation/prompt-optimisation-view.ts`
- Modify: `packages/agent-viewer/src/views/optimisation/prompt-optimisation-view.test.ts`
- Modify: `packages/agent-viewer/src/views/social/memory-hygiene-view.ts`
- Modify: `packages/agent-viewer/src/views/social/memory-hygiene-view.test.ts`

**Interfaces:**
- Consumes: `renderTabBar()`, `renderTabPanel()`, `renderExecution()` from PatternViewMixin (Task 2)

- [ ] **Step 1: Add ARIA tab tests to both test files**

Same tab tests as Task 3 Step 1 (tablist present, Execution by default, Reports stub).

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/agent-viewer/src/views/optimisation/ packages/agent-viewer/src/views/social/memory-hygiene-view.test.ts`
Expected: FAIL

- [ ] **Step 3: Migrate both views**

Same migration pattern as Task 3: remove duplicated header CSS, set
`--pattern-badge-color`, rename `render()` → `renderExecution()`, add tab
wrapper `render()`.

Optimisation badge colour: `rgba(250, 179, 135, 0.2)` / `#fab387`
Social badge colour: `rgba(245, 194, 231, 0.2)` / `#f5c2e7`

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/agent-viewer/src/views/optimisation/ packages/agent-viewer/src/views/social/memory-hygiene-view.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/agent-viewer/src/views/optimisation/ packages/agent-viewer/src/views/social/memory-hygiene-view.ts packages/agent-viewer/src/views/social/memory-hygiene-view.test.ts
git commit -m "feat(agent-viewer): add Execution/Reports tabs to optimisation and memory-hygiene views

Refs #288"
```

---

## Batch 3: cognition-view rework

### Task 5: Rework cognition-view with typed config sections

**Files:**
- Modify: `packages/agent-viewer/src/views/social/cognition-view.ts`
- Modify: `packages/agent-viewer/src/views/social/cognition-view.test.ts`

**Interfaces:**
- Consumes: All `*ConfigSpec` types from mosaic-core (Task 1),
  `renderTabBar()`, `renderTabPanel()`, `renderExecution()` from PatternViewMixin (Task 2)

- [ ] **Step 1: Update test fixture with typed config data**

Replace the fabricated fixture in `cognition-view.test.ts` with typed data
matching the new `*ConfigSpec` interfaces:

```typescript
import type { SocialPatternCategory, MoodConfigSpec, DriveConfigSpec } from '@casehubio/mosaic-core';

function fixture(): SocialPatternCategory {
  return {
    category: 'social',
    type: 'cognition',
    name: 'agent-cognition',
    description: 'Full cognitive configuration for autonomous agent',
    cognition: {
      drive: {
        axisWeights: { curiosity: 0.8, affiliation: 0.6 },
        changeThreshold: 0.1,
        maxIntensity: 1.0,
        minIntensity: 0.0,
        moodPleasureModulation: 0.3,
      },
      mood: {
        baseline: { pleasure: 0.0, arousal: 0.0, dominance: 0.0 },
        decayTimeConstant: 'PT4H',
        maxDisplacement: 1.0,
        moodInfluence: 0.5,
        staleStateTimeout: 'PT24H',
      },
      retention: { retentionThreshold: 0.6, confidenceWeight: 0.4 },
      mentalModel: {
        beliefHalfLife: 'PT48H',
        confidenceFloor: 0.3,
        minSignalsForInference: 3,
      },
    },
  };
}
```

- [ ] **Step 2: Write tests for tab structure and typed config rendering**

```typescript
// Tab tests
it('renders tablist with Execution and Reports tabs', async () => {
  await create(fixture());
  const tablist = el.shadowRoot!.querySelector('[role="tablist"]');
  expect(tablist).toBeTruthy();
  const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
  expect(tabs.length).toBe(2);
});

it('Reports tab shows stub placeholder', async () => {
  await create(fixture());
  const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
  (tabs[1] as HTMLElement).click();
  await (el as any).updateComplete;
  expect(el.shadowRoot!.querySelector('.empty-state')?.textContent).toBe('No report data available');
});

// Typed config tests
it('renders all 13 section cards', async () => {
  await create(fixture());
  expect(el.shadowRoot!.querySelectorAll('.section-card').length).toBe(13);
});

it('marks configured sections as active', async () => {
  await create(fixture());
  const active = el.shadowRoot!.querySelectorAll('.section-card.active');
  expect(active.length).toBe(4);
});

it('renders typed config fields for mood section', async () => {
  await create(fixture());
  const moodCard = Array.from(el.shadowRoot!.querySelectorAll('.section-card.active'))
    .find(c => c.querySelector('.section-name')?.textContent === 'Mood');
  expect(moodCard).toBeTruthy();
  const fields = moodCard!.querySelectorAll('.config-field');
  expect(fields.length).toBeGreaterThan(0);
  const labels = Array.from(fields).map(f => f.querySelector('.field-label')?.textContent);
  expect(labels).toContain('decayTimeConstant');
  expect(labels).toContain('maxDisplacement');
});

it('renders drive axis weights', async () => {
  await create(fixture());
  const driveCard = Array.from(el.shadowRoot!.querySelectorAll('.section-card.active'))
    .find(c => c.querySelector('.section-name')?.textContent === 'Drive');
  expect(driveCard).toBeTruthy();
  expect(driveCard!.querySelector('.axis-weights')).toBeTruthy();
});

it('renders summary count', async () => {
  await create(fixture());
  expect(el.shadowRoot!.querySelector('.summary')?.textContent).toBe('4 of 13 cognitive modules configured');
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn vitest run packages/agent-viewer/src/views/social/cognition-view.test.ts`
Expected: FAIL — no tablist, no `.config-field`, no `.axis-weights`

- [ ] **Step 4: Rework cognition-view**

1. Add tab wrapper: `render()` → `renderTabBar()` + `renderTabPanel()`, existing
   body → `renderExecution()`
2. Remove duplicated header CSS, set `--pattern-badge-color`
3. Replace generic `renderSection()` with typed section rendering:
   - Each section card shows typed config fields when expanded/active
   - `renderConfigFields(spec)` — iterates typed fields, renders label + value
   - Duration fields show human-readable format
   - `axisWeights` maps render as horizontal bar chart or key-value list
   - Nested objects (MoodBaseline, NarrativeSynthesisGateSpec) render as sub-sections
4. Group sections into categories: Drives & Mood, Models & Learning, Goals,
   Memory & Norms
5. Keep existing summary count

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn vitest run packages/agent-viewer/src/views/social/cognition-view.test.ts`
Expected: PASS

- [ ] **Step 6: Run full test suite and typecheck**

Run: `yarn test && yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/agent-viewer/src/views/social/cognition-view.ts packages/agent-viewer/src/views/social/cognition-view.test.ts
git commit -m "feat(agent-viewer): rework cognition-view with typed CognitionDefinition sections

Replace generic module card grid with typed config rendering.
Section grouping (Drives & Mood, Models & Learning, Goals, Memory & Norms).
Execution/Reports tabs via PatternViewMixin.

Refs #288, closes #287"
```

---

## Batch 4: Gallery and cross-repo issues

### Task 6: Update gallery demo data and file cross-repo issues

**Files:**
- Modify: `packages/agent-viewer/gallery.html`

**Interfaces:**
- Consumes: All typed `*ConfigSpec` interfaces (Task 1)

- [ ] **Step 1: Update cognition gallery demo data**

In `packages/agent-viewer/gallery.html`, find the cognition-view demo section and
replace the fabricated `Record<string, unknown>` demo data with typed
`CognitionConfig` data using the new `*ConfigSpec` shapes. Include at least 5
sections configured (drive, mood, narrative, retention, mentalModel) to showcase
both configured and unconfigured rendering.

- [ ] **Step 2: Verify all 6 views render tabs in gallery**

Open `gallery.html` in a browser (via Vite dev server) and verify:
- All 6 pattern views show Execution/Reports tab bar
- Execution tab shows existing content
- Reports tab shows "No report data available" placeholder
- cognition-view shows typed config detail with section grouping

- [ ] **Step 3: Run full test suite**

Run: `yarn test && yarn typecheck`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add packages/agent-viewer/gallery.html
git commit -m "feat(agent-viewer): update gallery with typed cognition demo data

All 6 pattern views render Execution/Reports tabs. Cognition demo
uses typed *ConfigSpec data.

Refs #288"
```

- [ ] **Step 5: File cross-repo issues**

File on casehubio/blocks:
```
Title: Expose CognitionDefinition data via REST endpoint for frontend consumption
Body: mosaic cognition-view now has typed *ConfigSpec interfaces matching
CognitionDefinition. Need a REST/GraphQL projection that serves this data
so the frontend can render real agent configurations.
Refs casehubio/casehub-mosaic#288
```

- [ ] **Step 6: Final commit with any remaining changes**

Run `yarn test && yarn typecheck` to verify clean state. Commit any remaining changes.

---

## References

- `specs/issue-288-grounded-pattern-views/2026-10-27-grounded-pattern-views-design.md` — design spec
- `packages/agent-viewer/src/shared-styles.ts` — PatternViewMixin
- `packages/mosaic-core/src/types/patterns/social.ts` — CognitionConfig types
- `packages/agent-viewer/src/views/social/cognition-view.ts` — cognition-view source
- `packages/agent-viewer/src/views/social/cognition-view.test.ts` — cognition-view tests
- `components/detail-pane/src/detail-pane.ts` — TabDefinition ARIA reference
- `agency/agentic-yaml CognitionDefinition.java` — authoritative Java type
- GitHub #288, #287
