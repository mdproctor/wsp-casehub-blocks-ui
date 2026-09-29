# Agent Catalog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #211 — agent-catalog — template browsing, filtering, and from-scratch creation
**Issue group:** #216, #211

**Goal:** Build a browsable template catalog that presents existing
profession/role/archetype variants as selectable agent configurations
with alias-driven manifest defaults.

**Architecture:** Single web component (`<agent-catalog>`) backed by
static template data derived from PROFESSION_PRESETS. Evolves
FullAgentDescriptor in blocks-ui-core. Extracts profile derivation
from avatar-step into a shared function.

**Tech Stack:** LitElement, TypeScript, Vitest, agent-avatar-2d

## Global Constraints

- Dark theme with `--pages-*` CSS custom properties
- ARIA mandatory on all components
- Web Component (`@customElement`)
- TypeScript strict mode
- Tests use Vitest with jsdom

---

## Batch 1: Foundation — types + profile extraction

### Task 1: Evolve FullAgentDescriptor with personality/manifest fields

**Files:**
- Modify: `packages/blocks-ui-core/src/types/agent.ts`
- Modify: `packages/blocks-ui-core/src/types/agent.test.ts`

**Interfaces:**
- Consumes: `PersonalityProfile` from avatar-step (type import only), `Manifest` from `./manifest.ts`
- Produces: Evolved `FullAgentDescriptor` with optional `description`, `personality`, `manifest`, `preferredAlias`, `profession`, `role` fields

- [ ] **Step 1: Write failing test — new fields accepted**

```typescript
// In agent.test.ts
import { describe, it, expect } from 'vitest';
import type { FullAgentDescriptor } from './agent.js';

describe('FullAgentDescriptor', () => {
  it('accepts optional personality and manifest fields', () => {
    const descriptor: FullAgentDescriptor = {
      agentId: 'test-1',
      name: 'Test Agent',
      tenancyId: 'tenant-1',
      archetypeFamily: 'Sage',
      subArchetype: 'Mentor',
      description: 'A test agent',
      profession: 'Software',
      role: 'Architect',
      preferredAlias: 'reasoning-heavy',
    };
    expect(descriptor.description).toBe('A test agent');
    expect(descriptor.profession).toBe('Software');
    expect(descriptor.role).toBe('Architect');
    expect(descriptor.preferredAlias).toBe('reasoning-heavy');
  });

  it('remains backward compatible — existing minimal shape works', () => {
    const descriptor: FullAgentDescriptor = {
      agentId: 'test-2',
      name: 'Minimal Agent',
      tenancyId: 'tenant-1',
    };
    expect(descriptor.agentId).toBe('test-2');
    expect(descriptor.description).toBeUndefined();
    expect(descriptor.personality).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/blocks-ui-core test`
Expected: FAIL — `personality` and new fields not on the type

- [ ] **Step 3: Add new optional fields to FullAgentDescriptor**

```typescript
// In agent.ts — add imports and new fields
import type { Manifest } from './manifest.js';

export interface PersonalityProfile {
  profession?: string;
  role?: string;
  mbti?: string;
  enneagram?: string;
  disc?: string;
  belbin?: { primary: string; secondaries: string[] };
  sdi?: string;
  bigFive?: Partial<Record<'O' | 'C' | 'E' | 'A' | 'N', 'high' | 'low'>>;
}

export interface FullAgentDescriptor {
  readonly agentId: string;
  readonly name: string;
  readonly tenancyId: string;
  readonly archetypeFamily?: string;
  readonly subArchetype?: string;
  readonly archetypeAdjectives?: readonly string[];
  readonly avatar?: string;
  readonly description?: string;
  readonly personality?: PersonalityProfile;
  readonly manifest?: Manifest;
  readonly preferredAlias?: string;
  readonly profession?: string;
  readonly role?: string;
}
```

Note: `PersonalityProfile` is defined here in blocks-ui-core (canonical
location for shared types) rather than importing from avatar-step, to
avoid a circular dependency. avatar-step's local `PersonalityProfile`
interface has the same shape — it can be replaced with a re-export from
blocks-ui-core in a later cleanup.

- [ ] **Step 4: Update index.ts to export PersonalityProfile**

Ensure `packages/blocks-ui-core/src/types/index.ts` exports the new
`PersonalityProfile` type from `agent.ts`.

- [ ] **Step 5: Run test to verify it passes**

Run: `yarn workspace @casehubio/blocks-ui-core test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/blocks-ui-core/src/types/agent.ts packages/blocks-ui-core/src/types/agent.test.ts packages/blocks-ui-core/src/types/index.ts
git commit -m "feat(#211): evolve FullAgentDescriptor with personality/manifest fields"
```

### Task 2: Extract initProfile into shared function

**Files:**
- Create: `components/avatar-step/src/data/profile-derivation.ts`
- Create: `components/avatar-step/src/data/profile-derivation.test.ts`
- Modify: `components/avatar-step/src/avatar-step.ts` (replace `_initProfile` with call to shared fn)
- Modify: `components/avatar-step/src/index.ts` (add export)

**Interfaces:**
- Consumes: `SUB_ARCHETYPE_RULES`, `FRAMEWORK_FAMILY_MAP` from `compatibility-matrix.ts`, `PersonalityProfile` from avatar-step
- Produces: `initProfile(archetypeKey: string): PersonalityProfile` — exported function

- [ ] **Step 1: Write failing test for initProfile**

```typescript
// profile-derivation.test.ts
import { describe, it, expect } from 'vitest';
import { initProfile } from './profile-derivation.js';

describe('initProfile', () => {
  it('derives full personality profile for Caregiver/Angel', () => {
    const profile = initProfile('Caregiver/Angel');
    expect(profile.mbti).toBe('INFJ');
    expect(profile.enneagram).toBe('Type 2');
    expect(profile.disc).toBe('S');
    expect(profile.sdi).toBe('Blue');
    expect(profile.belbin?.primary).toBe('Co-ordinator');
    expect(profile.belbin?.secondaries).toContain('Teamworker');
    expect(profile.bigFive?.O).toBe('low');
    expect(profile.bigFive?.A).toBe('high');
  });

  it('derives profile for Hero/Warrior', () => {
    const profile = initProfile('Hero/Warrior');
    expect(profile.mbti).toBe('ENTJ');
    expect(profile.enneagram).toBe('Type 8');
    expect(profile.disc).toBe('D');
    expect(profile.sdi).toBe('Red');
  });

  it('returns empty profile for unknown archetype', () => {
    const profile = initProfile('Unknown/None');
    expect(profile.mbti).toBeUndefined();
    expect(profile.sdi).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/avatar-step test -- src/data/profile-derivation.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Create profile-derivation.ts**

Extract `_initProfile` from `avatar-step.ts` into a standalone exported
function. Move the `BIG_FIVE_DIMS` const along with it.

```typescript
// profile-derivation.ts
import type { ArchetypeFamily } from '@casehubio/agent-avatar-2d';
import { SUB_ARCHETYPE_RULES, FRAMEWORK_FAMILY_MAP } from './compatibility-matrix.js';
import type { BigFiveDimension, BigFivePole } from './compatibility-matrix.js';
import type { PersonalityProfile } from '../avatar-step.js';

const BIG_FIVE_DIMS: BigFiveDimension[] = ['O', 'C', 'E', 'A', 'N'];

export function initProfile(archetypeKey: string): PersonalityProfile {
  const [family, sub] = archetypeKey.split('/');
  const profile: PersonalityProfile = {};

  const rules = SUB_ARCHETYPE_RULES[family as ArchetypeFamily]?.find(r => r.subArchetype === sub);
  if (rules?.mbtiAffinity.length) profile.mbti = rules.mbtiAffinity[0];
  if (rules?.enneagramAffinity.length) profile.enneagram = `Type ${rules.enneagramAffinity[0]}`;

  for (const fw of ['disc', 'sdi'] as const) {
    const map = FRAMEWORK_FAMILY_MAP[fw];
    for (const [val, families] of Object.entries(map)) {
      if ((families as string[]).includes(family!)) { profile[fw] = val; break; }
    }
  }

  const belbinMatches: string[] = [];
  for (const [val, families] of Object.entries(FRAMEWORK_FAMILY_MAP.belbin)) {
    if ((families as string[]).includes(family!)) belbinMatches.push(val);
  }
  if (belbinMatches.length > 0) {
    profile.belbin = { primary: belbinMatches[0]!, secondaries: belbinMatches.slice(1, 3) };
  }

  const bigFive: Partial<Record<BigFiveDimension, BigFivePole>> = {};
  for (const dim of BIG_FIVE_DIMS) {
    const highFamilies = FRAMEWORK_FAMILY_MAP.bigFive[`High ${dim}`] as string[] | undefined;
    const lowFamilies = FRAMEWORK_FAMILY_MAP.bigFive[`Low ${dim}`] as string[] | undefined;
    const inHigh = highFamilies?.includes(family!) ?? false;
    const inLow = lowFamilies?.includes(family!) ?? false;
    if (inHigh && !inLow) bigFive[dim] = 'high';
    else if (inLow && !inHigh) bigFive[dim] = 'low';
  }
  if (Object.keys(bigFive).length > 0) profile.bigFive = bigFive;

  return profile;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @casehubio/avatar-step test -- src/data/profile-derivation.test.ts`
Expected: PASS

- [ ] **Step 5: Replace _initProfile in avatar-step.ts**

In `avatar-step.ts`:
1. Add import: `import { initProfile } from './data/profile-derivation.js';`
2. Replace the `_initProfile` method body with a delegation:
```typescript
private _initProfile(archetypeKey: string): PersonalityProfile {
  return initProfile(archetypeKey);
}
```
3. Remove the `BIG_FIVE_DIMS` const from the top of the file (it's now
   in profile-derivation.ts). Note: `BIG_FIVE_DIMS` is also used in
   `_renderProfileSection` and `_renderPersonalityPanel` — keep a second
   declaration in avatar-step.ts, OR import it. Since it's a simple
   const, keep a local copy to avoid export clutter.

- [ ] **Step 6: Add export to index.ts**

In `components/avatar-step/src/index.ts`, add:
```typescript
export { initProfile } from './data/profile-derivation.js';
```

- [ ] **Step 7: Run all avatar-step tests**

Run: `yarn workspace @casehubio/avatar-step test`
Expected: ALL PASS (including the existing 46 tests)

- [ ] **Step 8: Commit**

```bash
git add components/avatar-step/src/data/profile-derivation.ts components/avatar-step/src/data/profile-derivation.test.ts components/avatar-step/src/avatar-step.ts components/avatar-step/src/index.ts
git commit -m "refactor(#211): extract initProfile into shared function"
```

## Batch 2: Template data + catalog component

### Task 3: Create template data and catalog scaffold

**Files:**
- Create: `components/agent-catalog/package.json`
- Create: `components/agent-catalog/vitest.config.ts`
- Create: `components/agent-catalog/src/index.ts`
- Create: `components/agent-catalog/src/data/catalog-templates.ts`
- Create: `components/agent-catalog/src/data/catalog-templates.test.ts`

**Interfaces:**
- Consumes: `PROFESSION_PRESETS`, `PROFESSION_LIST`, `RoleVariant` from `@casehubio/avatar-step`; `initProfile` from `@casehubio/avatar-step`; `FullAgentDescriptor` from `@casehubio/blocks-ui-core`
- Produces: `CatalogTemplate` interface, `CATALOG_TEMPLATES` array, `buildDescriptor(template: CatalogTemplate): FullAgentDescriptor`, `FEATURED_TEMPLATES` filtered array

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/agent-catalog",
  "version": "0.0.1",
  "type": "module",
  "main": "src/index.ts",
  "dependencies": {
    "@casehubio/agent-avatar-2d": "workspace:*",
    "@casehubio/avatar-step": "workspace:*",
    "@casehubio/blocks-ui-core": "workspace:*",
    "lit": "^3.0.0"
  },
  "devDependencies": {
    "vitest": "^3.0.0"
  },
  "scripts": {
    "test": "vitest run"
  }
}
```

- [ ] **Step 2: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@casehubio/agent-avatar-2d': resolve(__dirname, '../../packages/agent-avatar-2d/src/index.ts'),
      '@casehubio/avatar-step': resolve(__dirname, '../avatar-step/src/index.ts'),
      '@casehubio/blocks-ui-core': resolve(__dirname, '../../packages/blocks-ui-core/src/index.ts'),
    },
  },
  esbuild: {
    target: 'es2022',
    tsconfigRaw: {
      compilerOptions: {
        experimentalDecorators: true,
        useDefineForClassFields: false,
      },
    },
  },
  test: {
    environment: 'jsdom',
    globals: true,
  },
});
```

- [ ] **Step 3: Write failing test for catalog templates**

```typescript
// catalog-templates.test.ts
import { describe, it, expect } from 'vitest';
import { CATALOG_TEMPLATES, FEATURED_TEMPLATES, buildDescriptor } from './catalog-templates.js';
import type { CatalogTemplate } from './catalog-templates.js';

describe('catalog-templates', () => {
  it('generates templates from all profession preset variants', () => {
    expect(CATALOG_TEMPLATES.length).toBeGreaterThan(100);
    for (const t of CATALOG_TEMPLATES) {
      expect(t.id).toBeTruthy();
      expect(t.profession).toBeTruthy();
      expect(t.role).toBeTruthy();
      expect(t.variant.archetype).toMatch(/\w+\/\w+/);
    }
  });

  it('assigns unique IDs', () => {
    const ids = CATALOG_TEMPLATES.map(t => t.id);
    expect(new Set(ids).size).toBe(ids.length);
  });

  it('featured templates is a subset of all templates', () => {
    expect(FEATURED_TEMPLATES.length).toBeGreaterThanOrEqual(3);
    expect(FEATURED_TEMPLATES.length).toBeLessThanOrEqual(5);
    for (const ft of FEATURED_TEMPLATES) {
      expect(ft.featured).toBe(true);
      expect(CATALOG_TEMPLATES).toContain(ft);
    }
  });

  it('buildDescriptor produces valid FullAgentDescriptor', () => {
    const template = CATALOG_TEMPLATES.find(t => t.profession === 'Software' && t.role === 'Architect')!;
    expect(template).toBeTruthy();
    const desc = buildDescriptor(template);
    expect(desc.agentId).toBe('');
    expect(desc.name).toBe(template.variant.label);
    expect(desc.tenancyId).toBe('');
    expect(desc.archetypeFamily).toBeTruthy();
    expect(desc.subArchetype).toBeTruthy();
    expect(desc.personality?.mbti).toBeTruthy();
    expect(desc.profession).toBe('Software');
    expect(desc.role).toBe('Architect');
    expect(desc.preferredAlias).toBeTruthy();
  });

  it('every template has a preferredAlias', () => {
    for (const t of CATALOG_TEMPLATES) {
      expect(t.preferredAlias, `${t.id} missing alias`).toBeTruthy();
    }
  });
});
```

- [ ] **Step 4: Run test to verify it fails**

Run: `yarn workspace @casehubio/agent-catalog test`
Expected: FAIL — module not found

- [ ] **Step 5: Implement catalog-templates.ts**

```typescript
// catalog-templates.ts
import { PROFESSION_PRESETS, PROFESSION_LIST, initProfile } from '@casehubio/avatar-step';
import type { RoleVariant } from '@casehubio/avatar-step';
import type { FullAgentDescriptor, PersonalityProfile } from '@casehubio/blocks-ui-core';

export interface CatalogTemplate {
  readonly id: string;
  readonly profession: string;
  readonly role: string;
  readonly variant: RoleVariant;
  readonly preferredAlias: string;
  readonly featured?: boolean;
}

const ALIAS_BY_ROLE: Record<string, string> = {
  'Architect': 'reasoning-heavy',
  'QA Lead': 'reasoning-heavy',
  'Data Scientist': 'reasoning-heavy',
  'Analyst': 'reasoning-heavy',
  'Researcher': 'reasoning-heavy',
  'Auditor': 'reasoning-heavy',
  'Litigator': 'reasoning-heavy',
  'Compliance Officer': 'reasoning-heavy',
  'Defence Attorney': 'reasoning-heavy',
  'Contracts Specialist': 'reasoning-heavy',
  'Surgeon': 'reasoning-heavy',
  'Psychiatrist': 'reasoning-heavy',
  'Quant': 'reasoning-heavy',
  'Trader': 'fast-response',
  'Sales Development': 'fast-response',
  'Tutor': 'fast-response',
  'Nurse': 'fast-response',
  'GP': 'fast-response',
  'Customer Success': 'fast-response',
  'Community Manager': 'fast-response',
};
const DEFAULT_ALIAS = 'reasoning-heavy';

function slugify(s: string): string {
  return s.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/(^-|-$)/g, '');
}

const FEATURED_IDS = new Set([
  'software-architect-the-systems-architect',
  'legal-compliance-officer-the-standards-enforcer',
  'medical-gp-the-holistic-family-doctor',
  'finance-analyst-the-evidence-driven-analyst',
  'coaching-life-coach-the-wisdom-guide',
]);

function buildTemplates(): CatalogTemplate[] {
  const templates: CatalogTemplate[] = [];
  for (const profession of PROFESSION_LIST) {
    const roles = PROFESSION_PRESETS[profession];
    if (!roles) continue;
    for (const { role, variants } of roles) {
      for (const variant of variants) {
        const id = `${slugify(profession)}-${slugify(role)}-${slugify(variant.label)}`;
        templates.push({
          id,
          profession,
          role,
          variant,
          preferredAlias: ALIAS_BY_ROLE[role] ?? DEFAULT_ALIAS,
          featured: FEATURED_IDS.has(id) || undefined,
        });
      }
    }
  }
  return templates;
}

export const CATALOG_TEMPLATES: readonly CatalogTemplate[] = buildTemplates();
export const FEATURED_TEMPLATES: readonly CatalogTemplate[] = CATALOG_TEMPLATES.filter(t => t.featured);

export function buildDescriptor(template: CatalogTemplate): FullAgentDescriptor {
  const [family, sub] = template.variant.archetype.split('/');
  const personality = initProfile(template.variant.archetype);
  return {
    agentId: '',
    name: template.variant.label,
    tenancyId: '',
    archetypeFamily: family,
    subArchetype: sub,
    description: template.variant.description,
    personality,
    preferredAlias: template.preferredAlias,
    profession: template.profession,
    role: template.role,
  };
}
```

- [ ] **Step 6: Create index.ts**

```typescript
// index.ts — placeholder, component export added in Task 4
export { CATALOG_TEMPLATES, FEATURED_TEMPLATES, buildDescriptor } from './data/catalog-templates.js';
export type { CatalogTemplate } from './data/catalog-templates.js';
```

- [ ] **Step 7: Install dependencies and run tests**

Run: `yarn install && yarn workspace @casehubio/agent-catalog test`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add components/agent-catalog/
git commit -m "feat(#211): catalog template data derived from PROFESSION_PRESETS"
```

### Task 4: Agent catalog web component

**Files:**
- Create: `components/agent-catalog/src/agent-catalog.ts`
- Create: `components/agent-catalog/src/agent-catalog.test.ts`
- Modify: `components/agent-catalog/src/index.ts` (add component export)

**Interfaces:**
- Consumes: `CATALOG_TEMPLATES`, `FEATURED_TEMPLATES`, `buildDescriptor`, `CatalogTemplate` from `./data/catalog-templates.js`; `AgentAvatar` from `@casehubio/agent-avatar-2d`; `buildSummaryText` from `@casehubio/avatar-step`
- Produces: `<agent-catalog>` custom element; emits `catalog:template:selected` with `{ template: FullAgentDescriptor }`

- [ ] **Step 1: Write failing tests**

```typescript
// agent-catalog.test.ts
import { describe, it, expect, beforeAll, beforeEach, afterEach } from 'vitest';
import { registerCollection, mythicCollection } from '@casehubio/agent-avatar-2d';
import './agent-catalog.js';

type CatalogEl = HTMLElement & { updateComplete: Promise<boolean> };

beforeAll(() => {
  registerCollection(mythicCollection);
});

describe('agent-catalog', () => {
  let el: CatalogEl;

  beforeEach(() => {
    el = document.createElement('agent-catalog') as CatalogEl;
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
  });

  it('renders with role="region" and aria-label', async () => {
    await el.updateComplete;
    expect(el.getAttribute('role')).toBe('region');
    expect(el.getAttribute('aria-label')).toBe('Agent template catalog');
  });

  it('renders featured section with curated templates', async () => {
    await el.updateComplete;
    const featured = el.shadowRoot!.querySelectorAll('.featured-card');
    expect(featured.length).toBeGreaterThanOrEqual(3);
    expect(featured.length).toBeLessThanOrEqual(5);
  });

  it('renders profession filter pills', async () => {
    await el.updateComplete;
    const pills = el.shadowRoot!.querySelectorAll('[data-filter="profession"]');
    expect(pills.length).toBeGreaterThan(0);
  });

  it('renders search input', async () => {
    await el.updateComplete;
    const search = el.shadowRoot!.querySelector('[role="searchbox"]');
    expect(search).toBeTruthy();
  });

  it('renders template grid with cards', async () => {
    await el.updateComplete;
    const cards = el.shadowRoot!.querySelectorAll('.template-card');
    expect(cards.length).toBeGreaterThan(0);
  });

  it('profession filter reduces grid to matching templates', async () => {
    await el.updateComplete;
    const allCards = el.shadowRoot!.querySelectorAll('.template-card').length;
    const pill = el.shadowRoot!.querySelector('[data-filter="profession"][data-value="Legal"]') as HTMLElement;
    expect(pill).toBeTruthy();
    pill.click();
    await el.updateComplete;
    const filteredCards = el.shadowRoot!.querySelectorAll('.template-card').length;
    expect(filteredCards).toBeLessThan(allCards);
    expect(filteredCards).toBeGreaterThan(0);
  });

  it('hides featured section when profession filter active', async () => {
    await el.updateComplete;
    const pill = el.shadowRoot!.querySelector('[data-filter="profession"]') as HTMLElement;
    pill.click();
    await el.updateComplete;
    const featured = el.shadowRoot!.querySelector('.featured-section');
    expect(featured).toBeNull();
  });

  it('card click expands inline detail', async () => {
    await el.updateComplete;
    const card = el.shadowRoot!.querySelector('.template-card') as HTMLElement;
    card.click();
    await el.updateComplete;
    const detail = el.shadowRoot!.querySelector('.detail-expansion');
    expect(detail).toBeTruthy();
    expect(detail!.getAttribute('role')).toBe('region');
  });

  it('second card click collapses previous and expands new', async () => {
    await el.updateComplete;
    const cards = el.shadowRoot!.querySelectorAll('.template-card');
    (cards[0] as HTMLElement).click();
    await el.updateComplete;
    const firstId = el.shadowRoot!.querySelector('.detail-expansion')?.getAttribute('data-template-id');
    (cards[1] as HTMLElement).click();
    await el.updateComplete;
    const details = el.shadowRoot!.querySelectorAll('.detail-expansion');
    expect(details.length).toBe(1);
    expect(details[0]!.getAttribute('data-template-id')).not.toBe(firstId);
  });

  it('select button emits catalog:template:selected', async () => {
    await el.updateComplete;
    const card = el.shadowRoot!.querySelector('.template-card') as HTMLElement;
    card.click();
    await el.updateComplete;
    let detail: Record<string, unknown> | undefined;
    el.addEventListener('catalog:template:selected', ((e: CustomEvent) => {
      detail = e.detail;
    }) as EventListener);
    const selectBtn = el.shadowRoot!.querySelector('.select-btn') as HTMLElement;
    selectBtn.click();
    await el.updateComplete;
    expect(detail).toBeDefined();
    expect(detail!.template).toBeDefined();
    const tmpl = detail!.template as Record<string, unknown>;
    expect(tmpl.archetypeFamily).toBeTruthy();
    expect(tmpl.personality).toBeDefined();
  });

  it('from-scratch button emits catalog:template:selected with empty descriptor', async () => {
    await el.updateComplete;
    let detail: Record<string, unknown> | undefined;
    el.addEventListener('catalog:template:selected', ((e: CustomEvent) => {
      detail = e.detail;
    }) as EventListener);
    const btn = el.shadowRoot!.querySelector('.from-scratch-btn') as HTMLElement;
    expect(btn).toBeTruthy();
    btn.click();
    await el.updateComplete;
    expect(detail).toBeDefined();
    const tmpl = detail!.template as Record<string, unknown>;
    expect(tmpl.agentId).toBe('');
    expect(tmpl.archetypeFamily).toBeUndefined();
  });

  it('search filters templates by label', async () => {
    await el.updateComplete;
    const allCards = el.shadowRoot!.querySelectorAll('.template-card').length;
    const input = el.shadowRoot!.querySelector('[role="searchbox"]') as HTMLInputElement;
    input.value = 'investigator';
    input.dispatchEvent(new Event('input', { bubbles: true }));
    await el.updateComplete;
    const filteredCards = el.shadowRoot!.querySelectorAll('.template-card').length;
    expect(filteredCards).toBeLessThan(allCards);
    expect(filteredCards).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/agent-catalog test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement agent-catalog.ts**

Full component implementation with:
- `_search`, `_profession`, `_expandedId` state
- `connectedCallback` setting ARIA attrs
- `_filteredTemplates()` computed from state
- `_renderFeatured()` — featured cards section (hidden when filters active)
- `_renderSearch()` — search input
- `_renderProfessionPills()` — profession filter pills
- `_renderGrid()` — card grid with inline detail expansion
- `_renderCard(template)` — individual card with avatar, label, profession>role
- `_renderDetail(template)` — expanded detail with personality summary, alias, select button
- `_renderFromScratch()` — from-scratch button
- `_select(template)` — emits event with `buildDescriptor(template)`
- `_selectFromScratch()` — emits event with empty descriptor

CSS follows existing dark-theme patterns from avatar-step (`.pill`,
`.panel`, card styling). Responsive grid: 3 columns desktop, 2 tablet,
1 mobile.

The component is ~300 lines of LitElement template code. Write it as
a single file — no sub-components needed at this scale.

- [ ] **Step 4: Update index.ts**

```typescript
export { AgentCatalog } from './agent-catalog.js';
export { CATALOG_TEMPLATES, FEATURED_TEMPLATES, buildDescriptor } from './data/catalog-templates.js';
export type { CatalogTemplate } from './data/catalog-templates.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/agent-catalog test`
Expected: ALL PASS

- [ ] **Step 6: Run avatar-step tests for regression**

Run: `yarn workspace @casehubio/avatar-step test`
Expected: ALL PASS (46 tests)

- [ ] **Step 7: Commit**

```bash
git add components/agent-catalog/
git commit -m "feat(#211): agent-catalog component — template browsing, filtering, selection"
```

## Batch 3: CLAUDE.md + schema registry

### Task 5: Update CLAUDE.md and schema registry

**Files:**
- Modify: `CLAUDE.md` (add agent-catalog entry to Key Directories)
- Modify: `packages/blocks-ui-schema/src/registry.ts` (add AgentCatalog if registry exists)

**Interfaces:**
- Consumes: `AgentCatalog` component
- Produces: Updated documentation and schema registry entry

- [ ] **Step 1: Add agent-catalog to CLAUDE.md Key Directories table**

Add entry under `components/agent-catalog/`:
```
| `components/agent-catalog/` | Agent template catalog — browsable grid of pre-built agent configurations derived from PROFESSION_PRESETS. Profession pill filters, search, featured picks, inline detail expansion with personality preview. Emits `catalog:template:selected` with FullAgentDescriptor. |
```

- [ ] **Step 2: Add to schema registry if applicable**

Check if `packages/blocks-ui-schema/src/registry.ts` has entries for
other agent-* components. If so, add `AgentCatalog`. If the registry
uses auto-detection, this may not be needed.

- [ ] **Step 3: Run schema staleness test**

Run: `yarn workspace @casehubio/blocks-ui-schema test`
Expected: PASS (or expected failure if registry needs regeneration —
follow the regeneration pattern used by other components)

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md packages/blocks-ui-schema/
git commit -m "docs(#211): add agent-catalog to CLAUDE.md and schema registry"
```

## References

- [specs/issue-216-211-avatar-catalog/2026-09-29-agent-catalog-design.md] — design spec
- [packages/blocks-ui-core/src/types/agent.ts] — FullAgentDescriptor
- [components/avatar-step/src/avatar-step.ts:342-377] — _initProfile to extract
- [components/avatar-step/src/data/profession-presets.ts] — PROFESSION_PRESETS
- [components/avatar-step/src/data/framework-descriptors.ts] — buildSummaryText
- [components/agent-manifest-editor/src/presets.ts] — STANDARD_ALIASES
- [GitHub #211] — focal issue
- [GitHub #166] — parent epic
