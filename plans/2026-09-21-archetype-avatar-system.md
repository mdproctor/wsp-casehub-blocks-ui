# Archetype Avatar System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #167 — Archetype-driven avatar system
**Issue group:** #167

**Goal:** Replace DiceBear Avataaars with a configurable avatar platform driven by 48 archetype presets, mechanical SVG composition, and themed collections.

**Architecture:** Composable SVG parts stored in sprite sheets (`mythic.parts.svg`), assembled at runtime by a layered builder. Each archetype is a config row mapping to abstract part IDs. Collections provide concrete SVG renderers. Compact identity codes (`mythic:P1B`) enable deterministic reproduction.

**Tech Stack:** TypeScript, Lit 3, Vitest, SVG

## Global Constraints

- Package: `@casehubio/agent-avatar-2d` — retain existing name, `agent-avatar` element tag
- ESM only, `"type": "module"` in package.json
- `tsconfig.json` extends `../../tsconfig.base.json`, references `blocks-ui-core`
- Vitest with jsdom environment, esbuild target es2022
- ARIA mandatory: `role="img"`, reactive `aria-label`
- SVG viewBox: `0 0 200 240` for all parts
- CSS custom property palette: `var(--skin)`, `var(--primary)`, `var(--secondary)`, `var(--accent)`, `var(--hair-color)`
- Part ID convention: `{category}:{part-id}` (e.g., `head:oval`, `prop:magnifying-glass`)
- Source on issue-166 branch has existing package scaffold (package.json, tsconfig, vitest.config) — cherry-pick or recreate

---

## Batch 1: Foundation — Types, Config Table, Palettes, Code Encoding

After this batch: all data structures defined, 48 archetype configs mapped, 12 palettes declared, compact codes encode/decode correctly. Zero rendering — pure data and logic.

### Task 1: Package scaffold + types

**Files:**
- Create: `packages/agent-avatar-2d/package.json`
- Create: `packages/agent-avatar-2d/tsconfig.json`
- Create: `packages/agent-avatar-2d/tsconfig.build.json`
- Create: `packages/agent-avatar-2d/vitest.config.ts`
- Create: `packages/agent-avatar-2d/src/types.ts`
- Create: `packages/agent-avatar-2d/src/index.ts`
- Test: `packages/agent-avatar-2d/src/__tests__/types.test.ts`

**Interfaces:**
- Produces: `AvatarPayload`, `PartAssignment`, `FamilyPalette`, `AvatarModifiers`, `PartModifiers`, `AxisExpression`, `AvatarSize`, `DetailLevel`, `ArchetypeFamily`, `AvatarCollection`

- [ ] **Step 1: Create package.json**

Based on issue-166 branch version but with DiceBear removed:

```json
{
  "name": "@casehubio/agent-avatar-2d",
  "version": "0.2.0",
  "description": "Archetype-driven 2D avatar system with composable SVG parts",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist",
    "extract-parts": "tsx src/scripts/extract-parts.ts"
  },
  "dependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "lit": "^3.2.1"
  },
  "devDependencies": {
    "rimraf": "^6.1.0",
    "tsx": "^4.0.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  }
}
```

- [ ] **Step 2: Create tsconfig.json, tsconfig.build.json, vitest.config.ts**

Copy from issue-166 branch — same structure, same aliases.

- [ ] **Step 3: Write the failing test for types**

```typescript
// src/__tests__/types.test.ts
import { describe, it, expect } from 'vitest';
import type {
  AvatarPayload, PartAssignment, FamilyPalette, AvatarModifiers,
  PartModifiers, AxisExpression, AvatarCollection,
} from '../types.js';
import { ARCHETYPE_FAMILIES, AVATAR_SIZES, DetailLevel } from '../types.js';

describe('types', () => {
  it('ARCHETYPE_FAMILIES contains all 12 families', () => {
    expect(ARCHETYPE_FAMILIES).toHaveLength(12);
    expect(ARCHETYPE_FAMILIES).toContain('Sage');
    expect(ARCHETYPE_FAMILIES).toContain('Hero');
    expect(ARCHETYPE_FAMILIES).toContain('Magician');
  });

  it('AVATAR_SIZES maps to pixel values', () => {
    expect(AVATAR_SIZES.xs).toBe(24);
    expect(AVATAR_SIZES.sm).toBe(40);
    expect(AVATAR_SIZES.md).toBe(64);
    expect(AVATAR_SIZES.lg).toBe(128);
  });

  it('DetailLevel enum orders correctly', () => {
    expect(DetailLevel.XS).toBeLessThan(DetailLevel.SM);
    expect(DetailLevel.SM).toBeLessThan(DetailLevel.MD);
    expect(DetailLevel.MD).toBeLessThan(DetailLevel.LG);
  });
});
```

- [ ] **Step 4: Run test to verify it fails**

Run: `yarn workspace @casehubio/agent-avatar-2d test -- src/__tests__/types.test.ts`
Expected: FAIL — modules not found

- [ ] **Step 5: Implement types.ts**

All interfaces from the spec: `AvatarPayload`, `PartAssignment`, `FamilyPalette`, `AvatarModifiers`, `PartModifiers`, `AxisExpression`, `AvatarCollection`. Plus constants `ARCHETYPE_FAMILIES`, `AVATAR_SIZES`, `DetailLevel` enum, `DETAIL_TIERS` mapping, `AvatarSize` type.

- [ ] **Step 6: Create index.ts with type exports**

- [ ] **Step 7: Run test to verify it passes**

- [ ] **Step 8: Run yarn install to wire workspace dependency**

- [ ] **Step 9: Commit**

```
feat(agent-avatar-2d): package scaffold and type definitions Refs #167
```

### Task 2: Palettes + Config Table

**Files:**
- Create: `packages/agent-avatar-2d/src/palettes.ts`
- Create: `packages/agent-avatar-2d/src/config-table.ts`
- Test: `packages/agent-avatar-2d/src/__tests__/config-table.test.ts`

**Interfaces:**
- Consumes: `FamilyPalette`, `PartAssignment`, `ArchetypeFamily` from types.ts
- Produces: `FAMILY_PALETTES`, `ARCHETYPE_CONFIGS`, `ARCHETYPE_INDEX` (sorted canonical list for code encoding)

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/config-table.test.ts
import { describe, it, expect } from 'vitest';
import { FAMILY_PALETTES } from '../palettes.js';
import { ARCHETYPE_CONFIGS, ARCHETYPE_INDEX } from '../config-table.js';
import { ARCHETYPE_FAMILIES } from '../types.js';

describe('FAMILY_PALETTES', () => {
  it('has a palette for every family', () => {
    for (const family of ARCHETYPE_FAMILIES) {
      expect(FAMILY_PALETTES[family]).toBeDefined();
      expect(FAMILY_PALETTES[family].primary).toMatch(/^#[0-9a-f]{6}$/i);
      expect(FAMILY_PALETTES[family].skin).toMatch(/^#[0-9a-f]{6}$/i);
    }
  });
});

describe('ARCHETYPE_CONFIGS', () => {
  it('has exactly 48 entries', () => {
    expect(Object.keys(ARCHETYPE_CONFIGS)).toHaveLength(48);
  });

  it('every entry has all required fields including hat and expression', () => {
    for (const [key, config] of Object.entries(ARCHETYPE_CONFIGS)) {
      expect(config.head, `${key}.head`).toBeTruthy();
      expect(config.hair, `${key}.hair`).toBeTruthy();
      expect(config.costume, `${key}.costume`).toBeTruthy();
      expect(config.eyebrows, `${key}.eyebrows`).toBeTruthy();
      expect(config.props, `${key}.props`).toBeInstanceOf(Array);
      expect(config.props.length, `${key}.props`).toBeGreaterThanOrEqual(1);
      // hat and expression are nullable — verify field exists (even if null)
      expect('hat' in config, `${key} missing hat field`).toBe(true);
      expect('expression' in config, `${key} missing expression field`).toBe(true);
    }
  });

  it('within each family, sub-archetypes differ by at least 2 parts', () => {
    for (const family of ARCHETYPE_FAMILIES) {
      const entries = Object.entries(ARCHETYPE_CONFIGS)
        .filter(([k]) => k.startsWith(`${family}/`));
      for (let i = 0; i < entries.length; i++) {
        for (let j = i + 1; j < entries.length; j++) {
          const [keyA, a] = entries[i]!;
          const [keyB, b] = entries[j]!;
          let diffs = 0;
          if (a.hair !== b.hair) diffs++;
          if (a.facialHair !== b.facialHair) diffs++;
          if (a.costume !== b.costume) diffs++;
          if (a.glasses !== b.glasses) diffs++;
          if (a.eyebrows !== b.eyebrows) diffs++;
          if (JSON.stringify(a.props) !== JSON.stringify(b.props)) diffs++;
          if (JSON.stringify(a.accessories) !== JSON.stringify(b.accessories)) diffs++;
          expect(diffs, `${keyA} vs ${keyB}`).toBeGreaterThanOrEqual(2);
        }
      }
    }
  });
});

describe('ARCHETYPE_INDEX', () => {
  it('is sorted alphabetically', () => {
    const sorted = [...ARCHETYPE_INDEX].sort();
    expect(ARCHETYPE_INDEX).toEqual(sorted);
  });

  it('matches ARCHETYPE_CONFIGS keys', () => {
    expect(ARCHETYPE_INDEX).toEqual(Object.keys(ARCHETYPE_CONFIGS).sort());
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement palettes.ts**

12 `FamilyPalette` entries from `part-catalogue.md` colour table.

- [ ] **Step 4: Implement config-table.ts**

48 `PartAssignment` entries from `part-catalogue.md` assignment tables. `ARCHETYPE_INDEX` is `Object.keys(ARCHETYPE_CONFIGS).sort()`.

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Commit**

```
feat(agent-avatar-2d): 12 family palettes and 48 archetype config presets Refs #167
```

### Task 3: Compact code encode/decode

**Files:**
- Create: `packages/agent-avatar-2d/src/code.ts`
- Test: `packages/agent-avatar-2d/src/__tests__/code.test.ts`

**Interfaces:**
- Consumes: `PartAssignment`, `ARCHETYPE_INDEX`, `ARCHETYPE_CONFIGS` from config-table.ts
- Produces: `encodePreset(archetypeKey: string, collection?: string): string`, `encodeCustom(assignment: PartAssignment, collection?: string): string`, `decodeCode(code: string): { collection: string; type: 'preset' | 'custom'; archetypeKey?: string; assignment: PartAssignment }`

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/code.test.ts
import { describe, it, expect } from 'vitest';
import { encodePreset, encodeCustom, decodeCode } from '../code.js';
import { ARCHETYPE_CONFIGS } from '../config-table.js';

describe('encodePreset', () => {
  it('encodes Sage/Detective as mythic:P with base36 index', () => {
    const code = encodePreset('Sage/Detective');
    expect(code).toMatch(/^mythic:P[0-9a-z]+$/i);
  });

  it('preset code length is compact', () => {
    const code = encodePreset('Sage/Detective');
    expect(code.length).toBeLessThanOrEqual(12);
  });

  it('accepts custom collection', () => {
    const code = encodePreset('Sage/Detective', 'pixel');
    expect(code).toStartWith('pixel:P');
  });
});

describe('encodeCustom', () => {
  it('produces a C-prefixed code with 10 base64 chars (60-bit encoding)', () => {
    const assignment = ARCHETYPE_CONFIGS['Sage/Detective']!;
    const code = encodeCustom({ ...assignment, hair: 'afro-short' });
    expect(code).toMatch(/^mythic:C.{10}$/);
  });
});

describe('decodeCode', () => {
  it('roundtrips preset codes', () => {
    for (const key of Object.keys(ARCHETYPE_CONFIGS)) {
      const code = encodePreset(key);
      const decoded = decodeCode(code);
      expect(decoded.type).toBe('preset');
      expect(decoded.archetypeKey).toBe(key);
      expect(decoded.assignment).toEqual(ARCHETYPE_CONFIGS[key]);
    }
  });

  it('roundtrips custom codes', () => {
    const original = { ...ARCHETYPE_CONFIGS['Sage/Detective']!, hair: 'afro-short' };
    const code = encodeCustom(original);
    const decoded = decodeCode(code);
    expect(decoded.type).toBe('custom');
    expect(decoded.assignment.hair).toBe('afro-short');
    expect(decoded.assignment.head).toBe(original.head);
  });

  it('detects preset vs custom correctly', () => {
    const presetCode = encodePreset('Hero/Warrior');
    const customCode = encodeCustom({ ...ARCHETYPE_CONFIGS['Hero/Warrior']!, glasses: 'aviator' });
    expect(decodeCode(presetCode).type).toBe('preset');
    expect(decodeCode(customCode).type).toBe('custom');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement code.ts**

Part index registries (ordered arrays for each category — head shapes, hairs, hats, expressions, etc.) for bit-packing. `encodePreset` produces `{collection}:P{base36(index)}`. `encodeCustom` packs 60 bits into 10 base64 chars with 2-bit version prefix (expanded from original 48-bit to accommodate hat and expression fields). `decodeCode` reverses both.

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```
feat(agent-avatar-2d): compact avatar code encode/decode with roundtrip tests Refs #167
```

---

## Batch 2: Builder + Collection Infrastructure

After this batch: the composition engine works end-to-end with stub SVG parts. You can call `buildAvatar()` and get a valid SVG string with correct layer ordering and size-tier filtering.

### Task 4: Collection registry + builder

**Files:**
- Create: `packages/agent-avatar-2d/src/collections/registry.ts`
- Create: `packages/agent-avatar-2d/src/builder.ts`
- Create: `packages/agent-avatar-2d/src/modifiers.ts`
- Test: `packages/agent-avatar-2d/src/__tests__/builder.test.ts`
- Test: `packages/agent-avatar-2d/src/__tests__/modifiers.test.ts`

**Interfaces:**
- Consumes: `PartAssignment`, `FamilyPalette`, `AvatarModifiers`, `PartModifiers`, `DetailLevel`, `AvatarCollection` from types.ts; `FAMILY_PALETTES` from palettes.ts
- Produces: `registerCollection(collection)`, `getCollection(id)`, `buildAvatar(config, palette, size, modifiers, registry)`, `renderAvatar(code, options?)`, `resolveModifiers(input)`, `applyPalette(svg, palette)`

- [ ] **Step 1: Write failing builder tests**

```typescript
// src/__tests__/builder.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { buildAvatar } from '../builder.js';
import { registerCollection, getCollection } from '../collections/registry.js';
import { ARCHETYPE_CONFIGS } from '../config-table.js';
import { FAMILY_PALETTES } from '../palettes.js';
import { DetailLevel } from '../types.js';
import type { AvatarCollection } from '../types.js';

function stubCollection(): AvatarCollection {
  const parts = new Map<string, string>();
  parts.set('head:oval', '<ellipse cx="100" cy="95" rx="38" ry="42" fill="var(--skin)"/>');
  parts.set('hair:bald-sides', '<path d="M62,88 Q62,55 80,50" fill="var(--hair-color)"/>');
  parts.set('costume:blazer-tie', '<path d="M60,240 L60,160" fill="var(--primary)"/>');
  parts.set('brow:thin-arched', '<path d="M74,81 Q80,77 96,80" fill="none" stroke="var(--hair-color)"/>');
  parts.set('glasses:round-wire', '<circle cx="85" cy="92" r="12" fill="none" stroke="#333"/>');
  parts.set('prop:magnifying-glass', '<circle cx="158" cy="175" r="18" stroke="var(--accent)"/>');
  parts.set('prop:notebook', '<rect x="150" y="170" width="20" height="28" fill="var(--accent)"/>');
  return { id: 'stub', partsUrl: '', previewUrl: '', parts };
}

describe('buildAvatar', () => {
  beforeAll(() => {
    registerCollection(stubCollection());
  });

  it('produces valid SVG string', () => {
    const config = ARCHETYPE_CONFIGS['Sage/Detective']!;
    const palette = FAMILY_PALETTES['Sage']!;
    const registry = getCollection('stub')!;
    const svg = buildAvatar(config, palette, 'lg', {}, registry);
    expect(svg).toContain('<svg');
    expect(svg).toContain('</svg>');
    expect(svg).toContain('viewBox');
  });

  it('applies palette — no var() references remain', () => {
    const config = ARCHETYPE_CONFIGS['Sage/Detective']!;
    const palette = FAMILY_PALETTES['Sage']!;
    const registry = getCollection('stub')!;
    const svg = buildAvatar(config, palette, 'lg', {}, registry);
    expect(svg).not.toContain('var(--');
  });

  it('xs size omits props and glasses', () => {
    const config = ARCHETYPE_CONFIGS['Sage/Detective']!;
    const palette = FAMILY_PALETTES['Sage']!;
    const registry = getCollection('stub')!;
    const svg = buildAvatar(config, palette, 'xs', {}, registry);
    expect(svg).not.toContain('magnifying');
    expect(svg).not.toContain('round-wire');
    expect(svg).toContain('ellipse'); // head still present
  });

  it('md size includes props but omits accessories', () => {
    const config = ARCHETYPE_CONFIGS['Sage/Detective']!;
    const palette = FAMILY_PALETTES['Sage']!;
    const registry = getCollection('stub')!;
    const svg = buildAvatar(config, palette, 'md', {}, registry);
    expect(svg).toContain('magnifying');
  });

  it('layers are in correct z-order: costume → head → hair → hat → beard → expression → brow → glasses → props → acc', () => {
    const config = ARCHETYPE_CONFIGS['Sage/Detective']!;
    const palette = FAMILY_PALETTES['Sage']!;
    const registry = getCollection('stub')!;
    const svg = buildAvatar(config, palette, 'lg', {}, registry);
    const costumeIdx = svg.indexOf('M60,240');
    const headIdx = svg.indexOf('cx="100" cy="95"');
    const hairIdx = svg.indexOf('M62,88');
    expect(costumeIdx).toBeLessThan(headIdx);
    expect(headIdx).toBeLessThan(hairIdx);
  });
});
```

- [ ] **Step 2: Write failing modifiers test**

```typescript
// src/__tests__/modifiers.test.ts
import { describe, it, expect } from 'vitest';
import { resolveModifiers } from '../modifiers.js';

describe('resolveModifiers', () => {
  it('returns neutral modifiers for empty input', () => {
    const mods = resolveModifiers({});
    expect(mods.intensity).toBe(0);
    expect(mods.temperament).toBe(0);
    expect(mods.energy).toBe(0);
    expect(mods.precision).toBe(0);
    expect(mods.organic).toBe(0);
  });

  it('maps "meticulous" to positive precision', () => {
    const mods = resolveModifiers({ adjectives: ['meticulous'] });
    expect(mods.precision).toBeGreaterThan(0);
  });

  it('maps "gentle" to negative intensity', () => {
    const mods = resolveModifiers({ adjectives: ['gentle'] });
    expect(mods.intensity).toBeLessThan(0);
  });

  it('maps canonical axes to expression', () => {
    const mods = resolveModifiers({
      canonicalAxes: { ruleFollowing: { term: 'strict', weight: 1.0 } },
    });
    expect(mods.expression.ruleFollowing).toBe('strict');
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

- [ ] **Step 4: Implement collections/registry.ts**

`registerCollection`, `getCollection`, default collection tracking.

- [ ] **Step 5: Implement modifiers.ts**

Adjective→category mapping table. `resolveModifiers()` function. `applyPalette()` string substitution.

- [ ] **Step 6: Implement builder.ts**

`buildAvatar()` with layer stacking, size-tier filtering, `resolvePartKey()` for variant selection, `assembleSvg()` wrapper. `renderAvatar(code, options?)` as the public API entry point.

- [ ] **Step 7: Update index.ts with new exports**

- [ ] **Step 8: Run all tests to verify they pass**

- [ ] **Step 9: Commit**

```
feat(agent-avatar-2d): builder, collection registry, modifier resolution Refs #167
```

---

## Batch 3: Mythic Collection — Starter SVG Parts

After this batch: `renderAvatar('mythic:P...')` produces real visual output for all 12 family roots. The extract script generates TypeScript from the SVG source of truth.

### Task 5: Extract script + mythic parts SVG (12 family roots)

**Files:**
- Create: `packages/agent-avatar-2d/src/scripts/extract-parts.ts`
- Create: `packages/agent-avatar-2d/src/collections/mythic/mythic.parts.svg`
- Create: `packages/agent-avatar-2d/src/collections/mythic/mythic.preview.svg`
- Create: `packages/agent-avatar-2d/src/collections/mythic/mythic-parts.ts` (generated)
- Create: `packages/agent-avatar-2d/src/collections/mythic/index.ts`
- Test: `packages/agent-avatar-2d/src/__tests__/mythic-collection.test.ts`

**Interfaces:**
- Consumes: `AvatarCollection` from types.ts; `registerCollection` from registry.ts; `ARCHETYPE_CONFIGS` from config-table.ts
- Produces: `mythicCollection` (default registered collection), `mythic.parts.svg` (source of truth)

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/mythic-collection.test.ts
import { describe, it, expect } from 'vitest';
import { mythicCollection } from '../collections/mythic/index.js';
import { ARCHETYPE_CONFIGS } from '../config-table.js';

describe('mythic collection', () => {
  it('has id "mythic"', () => {
    expect(mythicCollection.id).toBe('mythic');
  });

  it('provides parts for all config-referenced part IDs including hat and expression', () => {
    const neededIds = new Set<string>();
    for (const config of Object.values(ARCHETYPE_CONFIGS)) {
      neededIds.add(`head:${config.head}`);
      neededIds.add(`hair:${config.hair}`);
      neededIds.add(`costume:${config.costume}`);
      neededIds.add(`brow:${config.eyebrows}`);
      if (config.facialHair !== 'none') neededIds.add(`beard:${config.facialHair}`);
      if (config.glasses) neededIds.add(`glasses:${config.glasses}`);
      if (config.hat) neededIds.add(`hat:${config.hat}`);
      if (config.expression) neededIds.add(`expression:${config.expression}`);
      for (const p of config.props) neededIds.add(`prop:${p}`);
      for (const a of config.accessories) neededIds.add(`acc:${a}`);
    }
    const missing = [...neededIds].filter(id => !mythicCollection.parts.has(id));
    expect(missing, `Missing parts: ${missing.join(', ')}`).toEqual([]);
  });

  it('every part contains valid SVG content', () => {
    for (const [id, content] of mythicCollection.parts.entries()) {
      expect(content, `${id} is empty`).toBeTruthy();
      expect(content, `${id} has no SVG elements`).toMatch(/<[a-z]/);
    }
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Create extract-parts.ts**

Script that reads `mythic.parts.svg`, parses `<symbol>` elements by ID, extracts inner content, writes `mythic-parts.ts` as a `Record<string, string>`.

- [ ] **Step 4: Create mythic.parts.svg**

Convert the avatar-preview.html SVG parts into the `<symbol>` format. Start with the parts needed for the 12 family roots (the existing preview avatars). Each part gets a `<symbol id="category:part-id" viewBox="0 0 200 240">`. Use `var(--skin)`, `var(--primary)`, etc. for palette colours.

This is the largest creative step — extract and convert ~60-80 unique parts from the HTML preview into clean, palette-driven SVG symbols.

- [ ] **Step 5: Create mythic.preview.svg**

12 pre-composed family root avatars as `<symbol id="preview:Family">` elements. These are the fully rendered versions from avatar-preview.html converted to symbol format.

- [ ] **Step 6: Run extract script**

Run: `yarn workspace @casehubio/agent-avatar-2d run extract-parts`
Verify: `mythic-parts.ts` generated with expected entries.

- [ ] **Step 7: Create collections/mythic/index.ts**

Wrap generated parts map into an `AvatarCollection` export. Register as default collection.

- [ ] **Step 8: Run tests to verify they pass**

The `mythic-collection.test.ts` validates that every part referenced by ARCHETYPE_CONFIGS exists in the collection.

- [ ] **Step 9: Visual verification**

Create a simple test HTML that renders all 12 family root avatars via `renderAvatar()`. Open in browser, compare with avatar-preview.html. Verify palette application, layer ordering, and visual fidelity.

- [ ] **Step 10: Commit**

```
feat(agent-avatar-2d): mythic collection with 12 family root SVG parts Refs #167
```

---

## Batch 4: Component + DiceBear Removal

After this batch: `<agent-avatar>` web component works with both payload and code input modes. DiceBear is gone. BlocksComponentRegistry updated.

### Task 6: `<agent-avatar>` component

**Files:**
- Create: `packages/agent-avatar-2d/src/agent-avatar.ts`
- Modify: `packages/agent-avatar-2d/src/index.ts`
- Test: `packages/agent-avatar-2d/src/__tests__/agent-avatar.test.ts`

**Interfaces:**
- Consumes: `renderAvatar` from builder.ts; `decodeCode` from code.ts; `AvatarPayload`, `AvatarSize` from types.ts
- Produces: `<agent-avatar>` custom element with `archetype`, `code`, `size`, `collection` properties

- [ ] **Step 1: Write failing tests**

```typescript
// src/__tests__/agent-avatar.test.ts
import { describe, it, expect, beforeAll, beforeEach } from 'vitest';
import { fixture, html } from '@open-wc/testing-helpers';
import '../agent-avatar.js';
import { registerCollection } from '../collections/registry.js';
import { mythicCollection } from '../collections/mythic/index.js';
import type { AgentAvatar } from '../agent-avatar.js';

beforeAll(() => {
  registerCollection(mythicCollection);
});

describe('agent-avatar', () => {
  it('has role="img"', async () => {
    const el = await fixture<AgentAvatar>(html`<agent-avatar></agent-avatar>`);
    expect(el.getAttribute('role')).toBe('img');
  });

  it('has aria-label', async () => {
    const el = await fixture<AgentAvatar>(html`
      <agent-avatar .archetype=${{ family: 'Sage', subArchetype: 'Detective' }}></agent-avatar>
    `);
    expect(el.getAttribute('aria-label')).toContain('Sage');
    expect(el.getAttribute('aria-label')).toContain('Detective');
  });

  it('renders SVG from archetype payload', async () => {
    const el = await fixture<AgentAvatar>(html`
      <agent-avatar .archetype=${{ family: 'Sage', subArchetype: 'Detective' }}></agent-avatar>
    `);
    await el.updateComplete;
    const svg = el.shadowRoot?.querySelector('svg');
    expect(svg).toBeTruthy();
  });

  it('renders SVG from compact code', async () => {
    const el = await fixture<AgentAvatar>(html`
      <agent-avatar code="mythic:P0"></agent-avatar>
    `);
    await el.updateComplete;
    const svg = el.shadowRoot?.querySelector('svg');
    expect(svg).toBeTruthy();
  });

  it('renders fallback when no input provided', async () => {
    const el = await fixture<AgentAvatar>(html`<agent-avatar></agent-avatar>`);
    await el.updateComplete;
    const svg = el.shadowRoot?.querySelector('svg');
    expect(svg).toBeTruthy();
  });

  it('respects consumer aria-label', async () => {
    const el = await fixture<AgentAvatar>(html`
      <agent-avatar aria-label="Custom label" .archetype=${{ family: 'Hero', subArchetype: 'Warrior' }}></agent-avatar>
    `);
    expect(el.getAttribute('aria-label')).toBe('Custom label');
  });

  it('defaults to md size', async () => {
    const el = await fixture<AgentAvatar>(html`<agent-avatar></agent-avatar>`);
    expect(el.size).toBe('md');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement agent-avatar.ts**

LitElement component per spec: `size`, `archetype`, `collection`, `code` properties. `connectedCallback` sets `role="img"`. `willUpdate` manages reactive `aria-label`. `render()` calls `renderAvatar()` or builds from payload. Uses `unsafeHTML` for SVG injection.

- [ ] **Step 4: Update index.ts exports**

Export `AgentAvatar`, `AgentAvatarProps` (for registry), `renderAvatar`, `encodePreset`, `encodeCustom`, `decodeCode`.

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Commit**

```
feat(agent-avatar-2d): <agent-avatar> component with payload and code modes Refs #167
```

### Task 7: DiceBear removal + registry + cleanup

**Files:**
- Modify: `packages/agent-avatar-2d/package.json` (remove DiceBear deps — already done in Task 1)
- Delete: `packages/agent-avatar-2d/dist/` (old compiled output)
- Modify: `packages/blocks-ui-schema/src/registry.ts` (add agent-avatar entry)

**Interfaces:**
- Consumes: `AgentAvatarProps` from agent-avatar-2d index.ts
- Produces: BlocksComponentRegistry entry for `agent-avatar`

- [ ] **Step 1: Delete old dist/ directory**

Remove `packages/agent-avatar-2d/dist/` — the old DiceBear compiled output. New dist will be generated from `yarn build`.

- [ ] **Step 2: Add BlocksComponentRegistry entry**

Use `ide_search_text` to find the registry file, then add the entry.

- [ ] **Step 3: Run `yarn build` for agent-avatar-2d**

Verify TypeScript compilation succeeds.

- [ ] **Step 4: Run `yarn workspace @casehubio/blocks-ui-schema run generate`**

Regenerate Zod schemas to include the new component.

- [ ] **Step 5: Run full test suite**

Run: `yarn test` at the workspace root. Verify no regressions.

- [ ] **Step 6: Commit**

```
feat(agent-avatar-2d): remove DiceBear, register in BlocksComponentRegistry Refs #167
```

---

## Batch 5: Consumer Wiring + FullAgentDescriptor

After this batch: `FullAgentDescriptor` carries archetype data. Consumer components can be updated to use the new API (on the issue-166 branch).

### Task 8: FullAgentDescriptor type additions

**Files:**
- Modify: `packages/blocks-ui-core/src/types/agent.ts`
- Test: `packages/blocks-ui-core/src/types/agent.test.ts`

**Interfaces:**
- Produces: `archetypeFamily`, `subArchetype`, `archetypeAdjectives`, `avatar` fields on `FullAgentDescriptor`

- [ ] **Step 1: Write failing test**

```typescript
// Add to existing agent.test.ts
it('FullAgentDescriptor supports archetype fields', () => {
  const descriptor: FullAgentDescriptor = {
    agentId: 'test-1',
    name: 'Inspector',
    tenancyId: 'tenant-1',
    archetypeFamily: 'Sage',
    subArchetype: 'Detective',
    archetypeAdjectives: ['meticulous', 'persistent'],
    avatar: 'mythic:P1B',
  };
  expect(descriptor.archetypeFamily).toBe('Sage');
  expect(descriptor.avatar).toBe('mythic:P1B');
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Add fields to FullAgentDescriptor**

Use `ide_edit_member` to add the 4 optional fields to the interface.

- [ ] **Step 4: Update blocks-ui-core types/index.ts re-exports if needed**

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Commit**

```
feat(blocks-ui-core): add archetype and avatar fields to FullAgentDescriptor Refs #167
```

---

## References

- [2026-09-21-archetype-avatar-system-design.md] — design spec this plan implements
- [part-catalogue.md] — full 48-archetype part assignments with visual hooks
- [avatar-preview.html] — live visual reference (12 family roots + part catalogues)
- [decisions.md] — 15 design decisions (D1-D15) with rationale
- [packages/agent-avatar-2d/dist/] — current DiceBear implementation (to be replaced)
- [packages/blocks-ui-core/src/types/agent.ts] — FullAgentDescriptor (source on issue-166 branch)
- [docs/protocols/blocks-ui/component-registry-props.md] — PP-20260907-fd8ee7
- [eidos specs/avatar-generator-contract.md] — archetype model, adjective catalog
- [GitHub #167] — focal issue
- [GitHub casehubio/eidos#181] — cross-repo: avatar field in agent YAML
