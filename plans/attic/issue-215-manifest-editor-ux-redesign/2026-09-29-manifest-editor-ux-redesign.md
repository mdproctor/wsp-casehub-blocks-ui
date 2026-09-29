# Manifest Editor UX Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #215 — agent-manifest-editor: UX redesign — configuration flow, model catalogue, alias resolution

**Goal:** Add pipeline visual cues, add-model capability on built-in provider cards, and alias resolution preview to the manifest editor.

**Architecture:** Three changes to existing files. Pipeline step headers with numbered sections + completion badges on agent-manifest-editor.ts. Add-model rows on built-in provider cards in manifest-provider-card.ts. Alias resolution preview computed from selected models on agent-manifest-editor.ts. No new components. No type changes.

**Tech Stack:** Lit 3.x, TypeScript, vitest with jsdom, pages-ui-tokens CSS custom properties

## Global Constraints

- All components use `--pages-*` CSS custom properties from `pages-ui-tokens`
- ARIA attributes required on all interactive elements
- No new `@customElement` registrations (modifying existing components only)
- Tests run with `npx vitest run --config components/agent-manifest-editor/vitest.config.ts`

---

## Batch 1: Pipeline Visual Cues + Alias Resolution

### Task 1: Pipeline Step Headers on agent-manifest-editor

**Files:**
- Modify: `components/agent-manifest-editor/src/agent-manifest-editor.ts:298-412`
- Test: `components/agent-manifest-editor/src/agent-manifest-editor.test.ts`

**Interfaces:**
- Consumes: `_providerStates: Map<string, ProviderChangedDetail>` (existing), `_aliases: AliasRow[]` (existing)
- Produces: `_getProviderStepStatus(): 'complete' | 'warning' | 'incomplete'`, `_getModelStepStatus(): 'complete' | 'warning' | 'incomplete'`, `_getAliasStepStatus(): 'complete' | 'warning' | 'incomplete'`, `_renderPipelineStep(step: number, title: string, status: string, dimmed: boolean): TemplateResult`

- [ ] **Step 1: Write failing tests for pipeline step rendering**

Add to `agent-manifest-editor.test.ts`:

```typescript
describe('pipeline steps', () => {
  it('renders three pipeline steps with step numbers', async () => {
    await el.updateComplete;
    const steps = el.shadowRoot!.querySelectorAll('.pipeline-step');
    expect(steps.length).toBe(3);
    const numbers = el.shadowRoot!.querySelectorAll('.step-number');
    expect(numbers[0]?.textContent?.trim()).toBe('1');
    expect(numbers[1]?.textContent?.trim()).toBe('2');
    expect(numbers[2]?.textContent?.trim()).toBe('3');
  });

  it('pipeline step has ARIA label with step number and status', async () => {
    await el.updateComplete;
    const steps = el.shadowRoot!.querySelectorAll('.pipeline-step');
    expect(steps[0]?.getAttribute('aria-label')).toMatch(/Step 1.*Providers/);
  });

  it('providers step shows incomplete when no providers configured', async () => {
    await el.updateComplete;
    const steps = el.shadowRoot!.querySelectorAll('.pipeline-step');
    const status = steps[0]?.querySelector('.step-status');
    expect(status?.classList.contains('incomplete')).toBe(true);
  });

  it('providers step shows complete when a provider has credential', async () => {
    el.data = {
      providers: [{ vendor: 'anthropic', credential: 'env:KEY' }],
      models: [{ id: 'claude-opus-4-6', vendor: 'anthropic' }],
    };
    await el.updateComplete;
    await el.updateComplete;
    const steps = el.shadowRoot!.querySelectorAll('.pipeline-step');
    const status = steps[0]?.querySelector('.step-status');
    expect(status?.classList.contains('complete')).toBe(true);
  });

  it('models section is dimmed when no providers configured', async () => {
    await el.updateComplete;
    const steps = el.shadowRoot!.querySelectorAll('.pipeline-step');
    const modelStep = steps[1];
    expect(modelStep?.querySelector('.step-dimmed')).toBeTruthy();
  });

  it('aliases section is dimmed when no models selected', async () => {
    await el.updateComplete;
    const steps = el.shadowRoot!.querySelectorAll('.pipeline-step');
    const aliasStep = steps[2];
    expect(aliasStep?.querySelector('.step-dimmed')).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run --config components/agent-manifest-editor/vitest.config.ts --reporter verbose 2>&1 | tail -30`
Expected: FAIL — `.pipeline-step` not found

- [ ] **Step 3: Add CSS for pipeline steps**

Add to `static styles` in `agent-manifest-editor.ts`:

```css
.pipeline-step { display: flex; align-items: center; gap: var(--pages-space-2, 0.5rem); margin: var(--pages-space-4, 1rem) 0 var(--pages-space-2, 0.5rem); }
.step-number { display: flex; align-items: center; justify-content: center; width: 20px; height: 20px; border-radius: 50%; background: var(--pages-neutral-4, #e5e5e5); color: var(--pages-neutral-11, #404040); font-size: var(--pages-font-size-xs, 11px); font-weight: var(--pages-font-weight-semibold, 600); flex-shrink: 0; }
.step-number.active { background: var(--pages-accent-9, #3b82f6); color: var(--pages-neutral-1, #fff); }
.step-title { font-size: var(--pages-font-size-sm, 12px); font-weight: var(--pages-font-weight-semibold, 600); color: var(--pages-neutral-9, #737373); text-transform: uppercase; letter-spacing: 0.05em; }
.step-status { width: 14px; height: 14px; border-radius: 50%; border: 1.5px solid var(--pages-neutral-6, #d4d4d4); flex-shrink: 0; display: flex; align-items: center; justify-content: center; font-size: 9px; }
.step-status.complete { background: var(--pages-success-9, #16a34a); border-color: var(--pages-success-9, #16a34a); color: white; }
.step-status.warning { background: var(--pages-warning-9, #d97706); border-color: var(--pages-warning-9, #d97706); color: white; }
.step-status.incomplete { background: transparent; }
.step-dimmed { opacity: 0.4; pointer-events: auto; }
.step-dimmed-tooltip { font-size: var(--pages-font-size-xs, 11px); color: var(--pages-neutral-8, #a3a3a3); font-style: italic; margin-left: auto; }
```

- [ ] **Step 4: Add step status computation methods**

Add to `AgentManifestEditor` class:

```typescript
private _getProviderStepStatus(): 'complete' | 'warning' | 'incomplete' {
  let hasCredential = false;
  let hasExpanded = false;
  for (const [, state] of this._providerStates) {
    if (state.credential || state.host) hasCredential = true;
    if (state.selectedModels.length > 0 && !state.credential && !state.host) hasExpanded = true;
  }
  if (hasCredential) return 'complete';
  if (hasExpanded) return 'warning';
  return 'incomplete';
}

private _getModelStepStatus(): 'complete' | 'warning' | 'incomplete' {
  let hasSelectedModels = false;
  let hasOrphanModels = false;
  for (const [, state] of this._providerStates) {
    if (state.selectedModels.length > 0) {
      hasSelectedModels = true;
      if (!state.credential && !state.host) hasOrphanModels = true;
    }
  }
  if (hasSelectedModels && !hasOrphanModels) return 'complete';
  if (hasSelectedModels) return 'warning';
  return 'incomplete';
}

private _getAliasStepStatus(): 'complete' | 'warning' | 'incomplete' {
  if (this._aliases.length === 0) return 'incomplete';
  const allKeysValid = this._aliases.every(a => a.key !== '');
  const noDuplicates = !this._aliases.some((a, i) => this._hasDuplicateAliasKey(a.key, i));
  if (allKeysValid && noDuplicates) return 'complete';
  return 'incomplete';
}

private _renderPipelineStep(step: number, title: string, status: 'complete' | 'warning' | 'incomplete', dimmed: boolean, tooltip?: string) {
  const isActive = status !== 'incomplete';
  return html`
    <div class="pipeline-step" aria-label="Step ${step}: ${title} — ${status}">
      <span class="step-number ${isActive ? 'active' : ''}">${step}</span>
      <span class="step-title">${title}</span>
      <span class="step-status ${status}">${status === 'complete' ? '✓' : status === 'warning' ? '!' : ''}</span>
      ${dimmed && tooltip ? html`<span class="step-dimmed-tooltip">${tooltip}</span>` : nothing}
    </div>
  `;
}
```

- [ ] **Step 5: Replace section-title divs with pipeline steps in render()**

Replace the `Providers` area (before provider-grid), `Models` implicit section, and `Aliases` section title in the `render()` method. The current code has no explicit "Providers" title — add one before the provider-grid. Replace the existing `.section-title` div before `.alias-editor` with a pipeline step. Add a dimmed wrapper around provider-grid and alias-editor when prerequisites aren't met:

```typescript
render() {
  if (this._loading) {
    return html`<div class="loading"><span class="spinner"></span> Loading configuration...</div>`;
  }

  const providerStatus = this._getProviderStepStatus();
  const modelStatus = this._getModelStepStatus();
  const aliasStatus = this._getAliasStepStatus();
  const noProviders = providerStatus === 'incomplete';
  const noModels = modelStatus === 'incomplete';

  return html`
    ${this.devMode ? html`<div class="dev-banner">Dev Mode — inline API keys enabled (not persisted)</div>` : nothing}
    ${this._error ? html`
      <div class="error-banner">
        ${this._error}
        <button class="retry-btn" @click=${this._fetchEndpoint}>Retry</button>
      </div>
    ` : nothing}

    <div class="preset-bar">
      ${PRESETS.map(p => html`
        <div class="preset-card ${classMap({ active: this._activePreset === p.id })}"
             data-preset=${p.id}
             @click=${() => this._onPresetClick(p.id)}>
          ${p.label}
        </div>
      `)}
    </div>

    ${this._renderPipelineStep(1, 'Providers', providerStatus, false)}
    <div class="provider-grid">
      ${BUILT_IN_PROVIDERS.map(bp => html`
        <manifest-provider-card
          .vendor=${bp.vendor}
          .displayName=${bp.displayName}
          .provider=${this._getProviderDecl(bp.vendor)}
          .models=${this._getModelsForProvider(bp.vendor)}
          .selectedModels=${this._getSelectedForProvider(bp.vendor)}
          .detection=${this._getDetection(bp.vendor)}
          .devMode=${this.devMode}
          .testEndpoint=${''}
          .isOther=${false}
          .authPatternOverride=${this._providerAuthPatterns.get(bp.vendor) ?? ''}
          @provider-changed=${this._onProviderChanged}
        ></manifest-provider-card>
      `)}

      ${this._dynamicProviders.map(dp => html`
        <manifest-provider-card
          .vendor=${dp.vendor}
          .displayName=${dp.displayName}
          .provider=${{ vendor: dp.vendor }}
          .models=${dp.models}
          .selectedModels=${this._getSelectedForProvider(dp.vendor)}
          .detection=${dp.detection}
          .devMode=${this.devMode}
          .testEndpoint=${''}
          .isOther=${false}
          @provider-changed=${this._onProviderChanged}
        ></manifest-provider-card>
      `)}

      <manifest-provider-card
        vendor="other"
        displayName="Other"
        .provider=${{ vendor: '' }}
        .models=${[]}
        .selectedModels=${[]}
        detection="none"
        .devMode=${this.devMode}
        testEndpoint=""
        .isOther=${true}
        @provider-changed=${this._onProviderChanged}
      ></manifest-provider-card>
    </div>

    ${this._renderPipelineStep(2, 'Models', modelStatus, noProviders, noProviders ? 'Configure a provider first' : undefined)}

    ${this._renderPipelineStep(3, 'Aliases', aliasStatus, noModels, noModels ? 'Select models first' : undefined)}
    <div class="alias-editor ${noModels ? 'step-dimmed' : ''}" ${noModels ? html`aria-disabled="true"` : nothing}>
      ${this._aliases.map((row, i) => this._renderAliasRow(row, i))}
      <button class="add-alias-btn" @click=${this._addAlias}>+ Add alias</button>
    </div>

    <div class="section-title">System Prompt Preview</div>
    <div class="prompt-label">Generated from personality profile</div>
    <div class="prompt-preview">
      ${this.systemPrompt
        ? this.systemPrompt
        : html`<span class="prompt-empty">No personality profile configured</span>`}
    </div>
  `;
}
```

Note: The "Models" pipeline step sits between the provider-grid and the alias-editor. The models themselves are still rendered inside each provider card — the step header is a visual cue indicating the pipeline stage, not a container for the model lists.

- [ ] **Step 6: Run tests to verify they pass**

Run: `npx vitest run --config components/agent-manifest-editor/vitest.config.ts --reporter verbose 2>&1 | tail -30`
Expected: PASS — all tests including new pipeline step tests

- [ ] **Step 7: Commit**

```bash
git add components/agent-manifest-editor/src/agent-manifest-editor.ts components/agent-manifest-editor/src/agent-manifest-editor.test.ts
git commit -m "feat(#215): pipeline step headers with completion status on manifest editor

Refs #215"
```

### Task 2: Alias Resolution Preview

**Files:**
- Modify: `components/agent-manifest-editor/src/agent-manifest-editor.ts:259-442`
- Test: `components/agent-manifest-editor/src/agent-manifest-editor.test.ts`

**Interfaces:**
- Consumes: `_providerStates: Map<string, ProviderChangedDetail>` (existing), `_aliases: AliasRow[]` (existing), `AliasDeclaration` and `ModelDescriptor` types from `@casehubio/blocks-ui-core`
- Produces: `_resolveAlias(alias: AliasDeclaration): ModelDescriptor | null`, `_getAllSelectedModels(): ModelDescriptor[]`

- [ ] **Step 1: Write failing tests for alias resolution**

Add to `agent-manifest-editor.test.ts`:

```typescript
describe('alias resolution preview', () => {
  it('shows resolved model name when alias matches', async () => {
    el.data = {
      providers: [{ vendor: 'anthropic', credential: 'env:KEY' }],
      models: [
        { id: 'claude-opus-4-6', displayName: 'Claude Opus 4.6', vendor: 'anthropic', tier: 'FLAGSHIP', capabilities: ['vision', 'tool_use'], contextWindow: 1000000 },
        { id: 'claude-haiku-4-5', displayName: 'Claude Haiku 4.5', vendor: 'anthropic', tier: 'FAST', capabilities: ['tool_use'], contextWindow: 200000 },
      ],
      aliases: { 'reasoning-heavy': { tier: 'FLAGSHIP', capabilities: ['tool_use'] } },
    };
    await el.updateComplete;
    await el.updateComplete;

    const resolution = el.shadowRoot!.querySelector('.alias-resolution');
    expect(resolution?.textContent).toContain('Claude Opus 4.6');
  });

  it('shows no-match warning when alias cannot resolve', async () => {
    el.data = {
      providers: [{ vendor: 'anthropic', credential: 'env:KEY' }],
      models: [{ id: 'claude-haiku-4-5', vendor: 'anthropic', tier: 'FAST' }],
      aliases: { 'embedding': { tier: 'EMBEDDING' } },
    };
    await el.updateComplete;
    await el.updateComplete;

    const resolution = el.shadowRoot!.querySelector('.alias-resolution');
    expect(resolution?.textContent).toContain('(no match)');
    expect(resolution?.classList.contains('no-match')).toBe(true);
  });

  it('resolution has aria-live="polite"', async () => {
    el.data = {
      providers: [{ vendor: 'anthropic', credential: 'env:KEY' }],
      models: [{ id: 'claude-opus-4-6', vendor: 'anthropic', tier: 'FLAGSHIP' }],
      aliases: { fast: { tier: 'FLAGSHIP' } },
    };
    await el.updateComplete;
    await el.updateComplete;

    const resolution = el.shadowRoot!.querySelector('.alias-resolution');
    expect(resolution?.getAttribute('aria-live')).toBe('polite');
  });

  it('preferVendor is a soft preference in resolution', async () => {
    el.data = {
      providers: [
        { vendor: 'openai', credential: 'env:KEY' },
      ],
      models: [
        { id: 'gpt-4o', displayName: 'GPT-4o', vendor: 'openai', tier: 'FLAGSHIP', contextWindow: 128000 },
      ],
      aliases: { 'reasoning-heavy': { tier: 'FLAGSHIP', preferVendor: 'anthropic' } },
    };
    await el.updateComplete;
    await el.updateComplete;

    const resolution = el.shadowRoot!.querySelector('.alias-resolution');
    expect(resolution?.textContent).toContain('GPT-4o');
  });

  it('stale-vendor warning is replaced by resolution preview', async () => {
    el.data = {
      providers: [{ vendor: 'openai', credential: 'env:KEY' }],
      models: [{ id: 'gpt-4o', vendor: 'openai', tier: 'FLAGSHIP' }],
      aliases: { test: { preferVendor: 'anthropic', tier: 'FLAGSHIP' } },
    };
    await el.updateComplete;
    await el.updateComplete;

    expect(el.shadowRoot!.querySelector('.alias-stale-warning')).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run --config components/agent-manifest-editor/vitest.config.ts --reporter verbose 2>&1 | tail -30`
Expected: FAIL — `.alias-resolution` not found

- [ ] **Step 3: Add resolution CSS and helper methods**

Add to `static styles`:

```css
.alias-resolution { font-size: var(--pages-font-size-sm, 12px); font-family: 'SF Mono', 'Fira Code', monospace; white-space: nowrap; }
.alias-resolution.match { color: var(--pages-success-11, #15803d); }
.alias-resolution.no-match { color: var(--pages-warning-9, #d97706); }
```

Add methods to `AgentManifestEditor` class:

```typescript
private _getAllSelectedModels(): ModelDescriptor[] {
  const models: ModelDescriptor[] = [];
  for (const [, state] of this._providerStates) {
    models.push(...state.selectedModels);
  }
  return models;
}

private _resolveAlias(alias: AliasDeclaration): ModelDescriptor | null {
  const allModels = this._getAllSelectedModels();
  const candidates = allModels.filter(m => {
    if (alias.tier && m.tier !== alias.tier) return false;
    if (alias.capabilities?.length) {
      if (!alias.capabilities.every(c => m.capabilities?.includes(c))) return false;
    }
    if (alias.locality && m.locality !== alias.locality) return false;
    if (alias.maxCost) {
      const costOrder = ['FREE', 'LOW', 'MEDIUM', 'HIGH', 'PREMIUM'];
      if (costOrder.indexOf(m.costTier ?? 'MEDIUM') > costOrder.indexOf(alias.maxCost)) return false;
    }
    if (alias.minContext && (m.contextWindow ?? 0) < alias.minContext) return false;
    if (alias.minOutput && (m.maxOutput ?? 0) < alias.minOutput) return false;
    return true;
  });
  if (candidates.length === 0) return null;
  const tierRank: Record<string, number> = { FLAGSHIP: 0, STANDARD: 1, FAST: 2, EMBEDDING: 3 };
  candidates.sort((a, b) => {
    const aPreferred = alias.preferVendor && a.vendor === alias.preferVendor ? 0 : 1;
    const bPreferred = alias.preferVendor && b.vendor === alias.preferVendor ? 0 : 1;
    if (aPreferred !== bPreferred) return aPreferred - bPreferred;
    const aTier = tierRank[a.tier ?? 'STANDARD'] ?? 1;
    const bTier = tierRank[b.tier ?? 'STANDARD'] ?? 1;
    if (aTier !== bTier) return aTier - bTier;
    return (b.contextWindow ?? 0) - (a.contextWindow ?? 0);
  });
  return candidates[0]!;
}
```

- [ ] **Step 4: Update _renderAliasRow to include resolution preview**

Replace the `_renderAliasRow` method:

```typescript
private _renderAliasRow(row: AliasRow, idx: number) {
  const duplicate = this._hasDuplicateAliasKey(row.key, idx);
  const resolved = this._resolveAlias(row.declaration);

  return html`
    <div class="alias-row">
      <input class="alias-key-input" .value=${row.key} placeholder="Alias name"
             @input=${(e: Event) => this._updateAliasKey(idx, (e.target as HTMLInputElement).value)}>
      <select class="alias-field"
              .value=${row.declaration.tier ?? ''}
              @change=${(e: Event) => this._updateAliasField(idx, 'tier', (e.target as HTMLSelectElement).value || undefined)}>
        <option value="">Tier</option>
        <option value="FLAGSHIP">Flagship</option>
        <option value="STANDARD">Standard</option>
        <option value="FAST">Fast</option>
        <option value="EMBEDDING">Embedding</option>
      </select>
      <input class="alias-field" type="number" placeholder="Min ctx" style="width:70px"
             .value=${String(row.declaration.minContext ?? '')}
             @input=${(e: Event) => this._updateAliasField(idx, 'minContext', Number((e.target as HTMLInputElement).value) || undefined)}>
      <input class="alias-field" placeholder="Prefer vendor" style="width:100px"
             .value=${row.declaration.preferVendor ?? ''}
             @input=${(e: Event) => this._updateAliasField(idx, 'preferVendor', (e.target as HTMLInputElement).value || undefined)}>
      <button class="delete-btn" @click=${() => this._removeAlias(idx)}>✗</button>
      <span class="alias-resolution ${resolved ? 'match' : 'no-match'}" aria-live="polite">
        → ${resolved ? (resolved.displayName ?? resolved.id) : '(no match)'}
      </span>
      ${duplicate ? html`<span class="alias-duplicate-error">Duplicate key</span>` : nothing}
    </div>
  `;
}
```

- [ ] **Step 5: Remove `_isStalePreferVendor` and enhance `_getAliasStepStatus`**

Delete the `_isStalePreferVendor` method (lines 259-265 in current file). Remove the `.alias-stale-warning` CSS rule from `static styles`.

Update `_getAliasStepStatus` (added in Task 1) to use resolution checking now that `_resolveAlias` exists:

```typescript
private _getAliasStepStatus(): 'complete' | 'warning' | 'incomplete' {
  if (this._aliases.length === 0) return 'incomplete';
  const allKeysValid = this._aliases.every(a => a.key !== '');
  const noDuplicates = !this._aliases.some((a, i) => this._hasDuplicateAliasKey(a.key, i));
  const allResolve = this._aliases.every(a => this._resolveAlias(a.declaration) !== null);
  if (allKeysValid && noDuplicates && allResolve) return 'complete';
  if (allKeysValid && noDuplicates) return 'warning';
  return 'incomplete';
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `npx vitest run --config components/agent-manifest-editor/vitest.config.ts --reporter verbose 2>&1 | tail -30`
Expected: PASS — all tests including resolution preview tests

- [ ] **Step 7: Commit**

```bash
git add components/agent-manifest-editor/src/agent-manifest-editor.ts components/agent-manifest-editor/src/agent-manifest-editor.test.ts
git commit -m "feat(#215): alias resolution preview with live model matching

Refs #215"
```

## Batch 2: Add-Model on Built-In Provider Cards + Integration

### Task 3: Add-Model on Built-In Provider Cards

**Files:**
- Modify: `components/agent-manifest-editor/src/manifest-provider-card.ts:294-616`
- Test: `components/agent-manifest-editor/src/manifest-provider-card.test.ts`

**Interfaces:**
- Consumes: `ModelDescriptor` from `@casehubio/blocks-ui-core`, `_hasDuplicateModelId(id, idx)` (existing, currently only used by Other card)
- Produces: `_addedModels: ModelDescriptor[]` (new state), `_addModel(): void`, `_removeAddedModel(idx): void`, `_updateAddedModel(idx, field, value): void`. The `_getSelectedModelDescriptors()` method (existing) and `_groupModelsByTier()` (existing) are updated to include `_addedModels`.

- [ ] **Step 1: Write failing tests for add-model on built-in cards**

Add to `manifest-provider-card.test.ts`:

```typescript
describe('add model on built-in card', () => {
  it('shows "Add model" button when expanded', async () => {
    await el.updateComplete;
    (el.shadowRoot!.querySelector('.card-header') as HTMLElement).click();
    await el.updateComplete;

    const addBtn = el.shadowRoot!.querySelector('.add-model-btn');
    expect(addBtn).toBeTruthy();
    expect(addBtn?.getAttribute('aria-label')).toBe('Add a model to Anthropic configuration');
  });

  it('clicking "Add model" creates an editable row', async () => {
    await el.updateComplete;
    (el.shadowRoot!.querySelector('.card-header') as HTMLElement).click();
    await el.updateComplete;

    const addBtn = el.shadowRoot!.querySelector('.add-model-btn') as HTMLElement;
    addBtn.click();
    await el.updateComplete;

    const addedRows = el.shadowRoot!.querySelectorAll('.added-model-row');
    expect(addedRows.length).toBe(1);
  });

  it('added model appears in tier group and can be selected', async () => {
    await el.updateComplete;
    (el.shadowRoot!.querySelector('.card-header') as HTMLElement).click();
    await el.updateComplete;

    const addBtn = el.shadowRoot!.querySelector('.add-model-btn') as HTMLElement;
    addBtn.click();
    await el.updateComplete;

    const idInput = el.shadowRoot!.querySelector('.added-model-row input[placeholder="Model ID"]') as HTMLInputElement;
    idInput.value = 'claude-sonnet-5-5';
    idInput.dispatchEvent(new Event('input', { bubbles: true }));
    await el.updateComplete;

    const events: any[] = [];
    el.addEventListener('provider-changed', ((e: CustomEvent) => events.push(e.detail)) as EventListener);

    const checkboxes = el.shadowRoot!.querySelectorAll('input[type="checkbox"]');
    const lastCheckbox = checkboxes[checkboxes.length - 1] as HTMLInputElement;
    lastCheckbox.click();
    await el.updateComplete;

    expect(events.length).toBeGreaterThan(0);
    expect(events[0].selectedModels.some((m: any) => m.id === 'claude-sonnet-5-5')).toBe(true);
  });

  it('duplicate model ID shows validation error', async () => {
    await el.updateComplete;
    (el.shadowRoot!.querySelector('.card-header') as HTMLElement).click();
    await el.updateComplete;

    const addBtn = el.shadowRoot!.querySelector('.add-model-btn') as HTMLElement;
    addBtn.click();
    await el.updateComplete;

    const idInput = el.shadowRoot!.querySelector('.added-model-row input[placeholder="Model ID"]') as HTMLInputElement;
    idInput.value = 'claude-opus-4-6';
    idInput.dispatchEvent(new Event('input', { bubbles: true }));
    await el.updateComplete;

    const error = el.shadowRoot!.querySelector('.added-model-row .validation-error');
    expect(error).toBeTruthy();
  });

  it('removing an added model removes it from the list', async () => {
    await el.updateComplete;
    (el.shadowRoot!.querySelector('.card-header') as HTMLElement).click();
    await el.updateComplete;

    const addBtn = el.shadowRoot!.querySelector('.add-model-btn') as HTMLElement;
    addBtn.click();
    await el.updateComplete;

    const deleteBtn = el.shadowRoot!.querySelector('.added-model-row .delete-btn') as HTMLElement;
    deleteBtn.click();
    await el.updateComplete;

    expect(el.shadowRoot!.querySelectorAll('.added-model-row').length).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run --config components/agent-manifest-editor/vitest.config.ts --reporter verbose 2>&1 | tail -30`
Expected: FAIL — `.add-model-btn` not found

- [ ] **Step 3: Add `_addedModels` state and mutation methods**

Add state and methods to `ManifestProviderCard`:

```typescript
@state() private _addedModels: ModelDescriptor[] = [];

private _addModel(): void {
  const id = `custom-${this._addedModels.length + 1}`;
  this._addedModels = [...this._addedModels, { id, vendor: this.vendor }];
}

private _removeAddedModel(idx: number): void {
  const removed = this._addedModels[idx]!;
  this._addedModels = this._addedModels.filter((_, i) => i !== idx);
  this.selectedModels = this.selectedModels.filter(id => id !== removed.id);
  this._emitChanged();
}

private _updateAddedModel(idx: number, field: string, value: string | number): void {
  const models = [...this._addedModels];
  const model = { ...models[idx]! };
  if (field === 'id') model.id = value as string;
  else if (field === 'displayName') model.displayName = value as string;
  else if (field === 'contextWindow') model.contextWindow = value as number;
  model.vendor = this.vendor;
  models[idx] = model;
  this._addedModels = models;
  this._emitChanged();
}

private _hasAddedDuplicateModelId(id: string, idx: number): boolean {
  const allModels = [...this.models, ...this._addedModels];
  return allModels.some((m, i) => {
    if (i === this.models.length + idx) return false;
    return m.id === id;
  });
}
```

- [ ] **Step 4: Update `_groupModelsByTier` and `_getSelectedModelDescriptors` to include added models**

Update `_groupModelsByTier()`:

```typescript
private _groupModelsByTier(): Record<string, ModelDescriptor[]> {
  const groups: Record<string, ModelDescriptor[]> = {};
  const tiers = ['FLAGSHIP', 'STANDARD', 'FAST', 'EMBEDDING'];
  for (const tier of tiers) groups[tier] = [];
  for (const model of [...this.models, ...this._addedModels]) {
    const tier = model.tier ?? 'STANDARD';
    if (!groups[tier]) groups[tier] = [];
    groups[tier]!.push(model);
  }
  return groups;
}
```

Update `_getSelectedModelDescriptors()`:

```typescript
private _getSelectedModelDescriptors(): ModelDescriptor[] {
  const allModels = this.isOther ? this._otherModels : [...this.models, ...this._addedModels];
  return allModels.filter(m => this.selectedModels.includes(m.id));
}
```

- [ ] **Step 5: Add `_renderAddedModelRow` and update `_renderModelList`**

Add render method:

```typescript
private _renderAddedModelRow(model: ModelDescriptor, idx: number) {
  const duplicate = this._hasAddedDuplicateModelId(model.id, idx);
  return html`
    <div class="added-model-row other-model-row">
      <input class="cred-input" .value=${model.id} placeholder="Model ID"
             @input=${(e: Event) => this._updateAddedModel(idx, 'id', (e.target as HTMLInputElement).value)}>
      <input class="cred-input" .value=${model.displayName ?? ''} placeholder="Display name"
             @input=${(e: Event) => this._updateAddedModel(idx, 'displayName', (e.target as HTMLInputElement).value)}>
      <input class="cred-input" type="number" .value=${String(model.contextWindow ?? '')} placeholder="Context"
             style="width: 80px"
             @input=${(e: Event) => this._updateAddedModel(idx, 'contextWindow', Number((e.target as HTMLInputElement).value))}>
      <button class="delete-btn" @click=${() => this._removeAddedModel(idx)}>✗</button>
      ${duplicate ? html`<span class="validation-error">Duplicate ID</span>` : nothing}
    </div>
  `;
}
```

Update `_renderModelList()` — add after the tier groups and before batch-set:

```typescript
private _renderModelList() {
  const groups = this._groupModelsByTier();
  return html`
    <div class="section-label">Models</div>
    ${this._renderBatchSet()}
    ${Object.entries(groups).filter(([, models]) => models.length > 0).map(([tier, models]) => html`
      <div class="tier-group">
        <div class="tier-label">${tier}</div>
        ${models.map(m => this._renderModelRow(m))}
      </div>
    `)}
    ${this._addedModels.map((m, i) => this._renderAddedModelRow(m, i))}
    <button class="add-model-btn add-btn" aria-label="Add a model to ${this.displayName} configuration" @click=${this._addModel}>+ Add model</button>
  `;
}
```

Add CSS for `.added-model-row` (reuses `.other-model-row` styles already defined):

```css
.add-model-btn { margin-top: var(--pages-space-2, 0.5rem); }
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `npx vitest run --config components/agent-manifest-editor/vitest.config.ts --reporter verbose 2>&1 | tail -30`
Expected: PASS — all tests including add-model tests

- [ ] **Step 7: Commit**

```bash
git add components/agent-manifest-editor/src/manifest-provider-card.ts components/agent-manifest-editor/src/manifest-provider-card.test.ts
git commit -m "feat(#215): add-model capability on built-in provider cards

Refs #215"
```

### Task 4: Integration Test — Preset → Add Model → Alias Resolves

**Files:**
- Modify: `components/agent-manifest-editor/src/agent-manifest-editor.integration.test.ts`

**Interfaces:**
- Consumes: All interfaces from Tasks 1-3 (pipeline steps, resolution preview, add-model)
- Produces: Integration test coverage for the full configuration pipeline

- [ ] **Step 1: Write integration test**

Add to `agent-manifest-editor.integration.test.ts`:

```typescript
describe('pipeline flow', () => {
  it('pipeline steps reflect configuration state', async () => {
    await el.updateComplete;

    // Initially all incomplete
    const steps = el.shadowRoot!.querySelectorAll('.pipeline-step');
    expect(steps.length).toBe(3);
    expect(steps[0]?.querySelector('.step-status.incomplete')).toBeTruthy();

    // Select preset → providers + models configured
    const preset = el.shadowRoot!.querySelector('[data-preset="anthropic-direct"]') as HTMLElement;
    preset.click();
    await el.updateComplete;
    await el.updateComplete;

    const updatedSteps = el.shadowRoot!.querySelectorAll('.pipeline-step');
    expect(updatedSteps[0]?.querySelector('.step-status.complete')).toBeTruthy();
    expect(updatedSteps[1]?.querySelector('.step-status.complete')).toBeTruthy();
  });

  it('end-to-end: preset → alias resolves to correct model', async () => {
    el.data = {
      providers: [{ vendor: 'anthropic', credential: 'env:KEY' }],
      models: [
        { id: 'claude-opus-4-6', displayName: 'Claude Opus 4.6', vendor: 'anthropic', tier: 'FLAGSHIP', capabilities: ['vision', 'tool_use'], contextWindow: 1000000 },
        { id: 'claude-haiku-4-5', displayName: 'Claude Haiku 4.5', vendor: 'anthropic', tier: 'FAST', capabilities: ['tool_use'], contextWindow: 200000 },
      ],
      aliases: { 'reasoning-heavy': { tier: 'FLAGSHIP', capabilities: ['tool_use'] } },
    };
    await el.updateComplete;
    await el.updateComplete;

    const resolution = el.shadowRoot!.querySelector('.alias-resolution');
    expect(resolution?.textContent).toContain('Claude Opus 4.6');
    expect(resolution?.classList.contains('match')).toBe(true);
  });

  it('models step dimmed message appears when no providers', async () => {
    await el.updateComplete;
    const tooltip = el.shadowRoot!.querySelector('.step-dimmed-tooltip');
    expect(tooltip?.textContent).toContain('Configure a provider first');
  });
});
```

- [ ] **Step 2: Run full test suite**

Run: `npx vitest run --config components/agent-manifest-editor/vitest.config.ts --reporter verbose 2>&1 | tail -40`
Expected: PASS — all unit and integration tests

- [ ] **Step 3: Commit**

```bash
git add components/agent-manifest-editor/src/agent-manifest-editor.integration.test.ts
git commit -m "test(#215): integration tests for pipeline flow + alias resolution

Refs #215"
```

## References

- [2026-09-29-manifest-editor-ux-redesign-design.md] — design spec this plan implements
- `components/agent-manifest-editor/src/agent-manifest-editor.ts` — main editor component (pipeline steps, alias resolution)
- `components/agent-manifest-editor/src/manifest-provider-card.ts:260-616` — provider card (add-model, model list rendering)
- `components/agent-manifest-editor/src/agent-manifest-editor.test.ts` — existing unit tests
- `components/agent-manifest-editor/src/manifest-provider-card.test.ts` — existing provider card tests
- `components/agent-manifest-editor/src/agent-manifest-editor.integration.test.ts` — existing integration tests
- `packages/blocks-ui-core/src/types/manifest.ts` — Manifest, ModelDescriptor, AliasDeclaration types
- `docs/protocols/blocks-ui/component-customisation-pattern.md` — typed config + render callback pattern
- GitHub #215 — focal issue
- GitHub #166 — parent epic (agent setup wizard)
