# Agent Setup Wizard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create issues during slot setup
**Issue group:** platform#285, eidos#175, blocks-ui#157

**Goal:** Four independent web components that make LLM and agent setup
trivial — manifest editor, agent template catalog, agent profile, and
relationship editor — plus a 2D avatar generation package.

**Architecture:** Independent LitElement web components following blocks-ui
patterns. Direct-fetch data model (no DataSourceMixin — these are
object-tree components, not tabular). DiceBear Avataaars for 2D avatar
generation. All components headless/payload-driven with dual data mode
(inline `data` property or `endpoint` URL).

**Tech Stack:** LitElement, TypeScript, vitest, DiceBear Avataaars, ELK
layout (existing), graph-stencil-org (existing)

## Global Constraints

- All components use `--pages-*` CSS custom properties from `pages-ui-tokens`
- ARIA attributes mandatory on every `@customElement` — role + aria-label minimum
- Tests include ARIA assertions
- Event topics use colon-delimited convention via `emitPagesEvent`
- TypeScript project references — each package has tsconfig.json referencing dependencies
- Package names: `@casehubio/blocks-ui-<name>`
- Workspace registration: root tsconfig.json needs `references` entry for each new package

---

## Batch 1: Foundation — blocks-ui-core type additions

### Task 1: Manifest types

**Files:**
- Create: `packages/blocks-ui-core/src/types/manifest.ts`
- Modify: `packages/blocks-ui-core/src/types/index.ts`
- Test: `packages/blocks-ui-core/src/types/manifest.test.ts`

**Interfaces:**
- Produces: `Manifest`, `ProviderDeclaration`, `ModelDescriptor`, `AliasDeclaration`, `SourceDeclaration`, `LocalModelDeclaration`, `ManifestDefaults`, `CredentialRef`, `ModelTier`, `ModelLocality`, `CostTier`

- [ ] **Step 1: Write the type file**

```typescript
// packages/blocks-ui-core/src/types/manifest.ts

export type ModelTier = 'FLAGSHIP' | 'STANDARD' | 'FAST' | 'EMBEDDING';
export type ModelLocality = 'CLOUD' | 'LOCAL' | 'HYBRID';
export type CostTier = 'FREE' | 'LOW' | 'MEDIUM' | 'HIGH' | 'PREMIUM';

export interface ProviderDeclaration {
  vendor: string;
  credential?: string | Record<string, string>;
  host?: string;
}

export interface ModelDescriptor {
  id: string;
  apiModelId?: string;
  backendKey?: string;
  backendInstanceId?: string;
  vendor?: string;
  family?: string;
  displayName?: string;
  tier?: ModelTier;
  capabilities?: string[];
  contextWindow?: number;
  maxOutput?: number;
  locality?: ModelLocality;
  costTier?: CostTier;
  authMethod?: string;
  properties?: Record<string, string>;
}

export interface AliasDeclaration {
  tier?: ModelTier;
  capabilities?: string[];
  locality?: ModelLocality;
  maxCost?: CostTier;
  minContext?: number;
  minOutput?: number;
  preferVendor?: string;
}

export interface SourceDeclaration {
  uri: string;
  priority?: number;
}

export interface LocalModelDeclaration {
  id: string;
  backendKey?: string;
  host?: string;
}

export interface ManifestDefaults {
  backend?: string;
}

export interface Manifest {
  providers?: ProviderDeclaration[];
  models?: ModelDescriptor[];
  aliases?: Record<string, AliasDeclaration>;
  defaults?: ManifestDefaults;
  sources?: SourceDeclaration[];
  localModels?: LocalModelDeclaration[];
}

export type CredentialRef =
  | { type: 'env'; name: string }
  | { type: 'file'; path: string }
  | { type: 'ref'; name: string };

export function parseCredentialRef(raw: string): CredentialRef {
  if (raw.startsWith('env:')) return { type: 'env', name: raw.slice(4) };
  if (raw.startsWith('file:')) return { type: 'file', path: raw.slice(5) };
  if (raw.startsWith('ref:')) return { type: 'ref', name: raw.slice(4) };
  return { type: 'env', name: raw };
}

export function formatCredentialRef(ref: CredentialRef): string {
  switch (ref.type) {
    case 'env': return `env:${ref.name}`;
    case 'file': return `file:${ref.path}`;
    case 'ref': return `ref:${ref.name}`;
  }
}
```

- [ ] **Step 2: Write the test**

```typescript
// packages/blocks-ui-core/src/types/manifest.test.ts
import { describe, it, expect } from 'vitest';
import { parseCredentialRef, formatCredentialRef } from './manifest.js';
import type { Manifest, CredentialRef } from './manifest.js';

describe('parseCredentialRef', () => {
  it('parses env: prefix', () => {
    const ref = parseCredentialRef('env:ANTHROPIC_API_KEY');
    expect(ref).toEqual({ type: 'env', name: 'ANTHROPIC_API_KEY' });
  });

  it('parses file: prefix', () => {
    const ref = parseCredentialRef('file:/etc/secrets/key');
    expect(ref).toEqual({ type: 'file', path: '/etc/secrets/key' });
  });

  it('parses ref: prefix', () => {
    const ref = parseCredentialRef('ref:vault-anthropic');
    expect(ref).toEqual({ type: 'ref', name: 'vault-anthropic' });
  });

  it('defaults to env for unprefixed string', () => {
    const ref = parseCredentialRef('MY_KEY');
    expect(ref).toEqual({ type: 'env', name: 'MY_KEY' });
  });
});

describe('formatCredentialRef', () => {
  it('round-trips env ref', () => {
    const ref: CredentialRef = { type: 'env', name: 'KEY' };
    expect(formatCredentialRef(ref)).toBe('env:KEY');
    expect(parseCredentialRef(formatCredentialRef(ref))).toEqual(ref);
  });

  it('round-trips file ref', () => {
    const ref: CredentialRef = { type: 'file', path: '/tmp/key' };
    expect(formatCredentialRef(ref)).toBe('file:/tmp/key');
    expect(parseCredentialRef(formatCredentialRef(ref))).toEqual(ref);
  });

  it('round-trips ref ref', () => {
    const ref: CredentialRef = { type: 'ref', name: 'vault-key' };
    expect(formatCredentialRef(ref)).toBe('ref:vault-key');
    expect(parseCredentialRef(formatCredentialRef(ref))).toEqual(ref);
  });
});

describe('Manifest type', () => {
  it('accepts a valid manifest with arrays', () => {
    const manifest: Manifest = {
      providers: [{ vendor: 'anthropic', credential: 'env:ANTHROPIC_API_KEY' }],
      models: [{ id: 'claude-sonnet-5', tier: 'FLAGSHIP', contextWindow: 200000 }],
      aliases: { 'reasoning-heavy': { tier: 'FLAGSHIP', capabilities: ['reasoning'] } },
    };
    expect(manifest.providers).toHaveLength(1);
    expect(manifest.models![0]!.tier).toBe('FLAGSHIP');
  });

  it('accepts map credential on provider', () => {
    const manifest: Manifest = {
      providers: [{ vendor: 'vertex', credential: { projectId: 'my-proj', region: 'us-central1' } }],
    };
    expect(typeof manifest.providers![0]!.credential).toBe('object');
  });
});
```

- [ ] **Step 3: Run tests to verify they pass**

Run: `yarn workspace @casehubio/blocks-ui-core test`

- [ ] **Step 4: Add re-export to types barrel**

Add `export * from './manifest.js';` to `packages/blocks-ui-core/src/types/index.ts`.

- [ ] **Step 5: Typecheck**

Run: `yarn workspace @casehubio/blocks-ui-core typecheck`

- [ ] **Step 6: Commit**

```bash
git add packages/blocks-ui-core/src/types/manifest.ts packages/blocks-ui-core/src/types/manifest.test.ts packages/blocks-ui-core/src/types/index.ts
git commit -m "feat(blocks-ui-core): add Manifest type mirrors for agent-config-core"
```

### Task 2: Expanded AgentDescriptor and AgentDisposition types

**Files:**
- Create: `packages/blocks-ui-core/src/types/agent.ts`
- Modify: `packages/blocks-ui-core/src/types/index.ts`
- Test: `packages/blocks-ui-core/src/types/agent.test.ts`

**Interfaces:**
- Consumes: `AgentCapability`, `AgentGoal`, `AgentConstraint` from existing types
- Produces: `DispositionValue`, `AgentDisposition`, `FullAgentDescriptor`, `primaryTerm()`

- [ ] **Step 1: Write the type file**

```typescript
// packages/blocks-ui-core/src/types/agent.ts
import type { AgentCapability, AgentGoal, AgentConstraint } from '../index.js';

export interface DispositionValue {
  term: string;
  weight: number;
}

export interface AgentDisposition {
  socialOrient?: DispositionValue[];
  ruleFollowing?: DispositionValue[];
  riskAppetite?: DispositionValue[];
  autonomy?: DispositionValue[];
  conflictMode?: DispositionValue[];
  delegation?: boolean;
  dispositionProfile?: DispositionValue[];
  styleProfile?: DispositionValue[];
}

export type DispositionAxis = keyof Pick<AgentDisposition,
  'socialOrient' | 'ruleFollowing' | 'riskAppetite' | 'autonomy' | 'conflictMode'>;

export const DISPOSITION_AXES: readonly DispositionAxis[] = [
  'socialOrient', 'ruleFollowing', 'riskAppetite', 'autonomy', 'conflictMode',
] as const;

export function primaryTerm(
  disposition: AgentDisposition | undefined,
  axis: DispositionAxis,
): DispositionValue | undefined {
  const values = disposition?.[axis];
  if (!values?.length) return undefined;
  return values.reduce((best, v) => v.weight > best.weight ? v : best, values[0]!);
}

export interface FullAgentDescriptor {
  agentId: string;
  name: string;
  version?: string;
  slot?: string;
  tenancyId: string;
  provider?: string;
  modelFamily?: string;
  modelVersion?: string;
  domainVocabulary?: string;
  slotVocabulary?: string;
  dispositionVocabulary?: string;
  axisVocabularies?: Record<string, string>;
  capabilities?: AgentCapability[];
  disposition?: AgentDisposition;
  goals?: AgentGoal[];
  constraints?: AgentConstraint[];
  briefing?: string;
  templates?: Record<string, string>;
  extensionData?: Record<string, unknown>;
}
```

- [ ] **Step 2: Write the test**

```typescript
// packages/blocks-ui-core/src/types/agent.test.ts
import { describe, it, expect } from 'vitest';
import { primaryTerm, DISPOSITION_AXES } from './agent.js';
import type { AgentDisposition, FullAgentDescriptor } from './agent.js';

describe('primaryTerm', () => {
  it('returns highest-weight term', () => {
    const disposition: AgentDisposition = {
      socialOrient: [
        { term: 'collaborative', weight: 0.7 },
        { term: 'independent', weight: 0.3 },
      ],
    };
    expect(primaryTerm(disposition, 'socialOrient')).toEqual({ term: 'collaborative', weight: 0.7 });
  });

  it('returns undefined for missing axis', () => {
    const disposition: AgentDisposition = {};
    expect(primaryTerm(disposition, 'autonomy')).toBeUndefined();
  });

  it('returns undefined for undefined disposition', () => {
    expect(primaryTerm(undefined, 'autonomy')).toBeUndefined();
  });

  it('returns single term when only one', () => {
    const disposition: AgentDisposition = {
      riskAppetite: [{ term: 'cautious', weight: 1.0 }],
    };
    expect(primaryTerm(disposition, 'riskAppetite')?.term).toBe('cautious');
  });
});

describe('DISPOSITION_AXES', () => {
  it('has exactly 5 axes', () => {
    expect(DISPOSITION_AXES).toHaveLength(5);
  });
});

describe('FullAgentDescriptor type', () => {
  it('accepts a complete descriptor', () => {
    const desc: FullAgentDescriptor = {
      agentId: 'support-01',
      name: 'Customer Support Agent',
      tenancyId: 'tenant-1',
      version: '1.0',
      provider: 'anthropic',
      modelFamily: 'claude',
      modelVersion: 'claude-sonnet-5',
      slot: 'supporter',
      capabilities: [{ name: 'issue-resolution' }],
      disposition: {
        socialOrient: [{ term: 'collaborative', weight: 1.0 }],
        delegation: true,
      },
      goals: [{ name: 'resolve-quickly', priority: 'PRIMARY' }],
      constraints: [{ name: 'hipaa', severity: 'HARD' }],
      briefing: 'You are a customer support agent.',
    };
    expect(desc.agentId).toBe('support-01');
    expect(desc.disposition?.delegation).toBe(true);
  });
});
```

- [ ] **Step 3: Run tests**

Run: `yarn workspace @casehubio/blocks-ui-core test`

- [ ] **Step 4: Add re-export to types barrel**

Add `export * from './agent.js';` to `packages/blocks-ui-core/src/types/index.ts`.

- [ ] **Step 5: Typecheck**

Run: `yarn workspace @casehubio/blocks-ui-core typecheck`

- [ ] **Step 6: Commit**

```bash
git add packages/blocks-ui-core/src/types/agent.ts packages/blocks-ui-core/src/types/agent.test.ts packages/blocks-ui-core/src/types/index.ts
git commit -m "feat(blocks-ui-core): add FullAgentDescriptor and AgentDisposition types"
```

### Task 3: Event topics and relationship registry extensions

**Files:**
- Modify: `packages/blocks-ui-core/src/types/events.ts`
- Modify: `packages/blocks-ui-core/src/types/relationship.ts`
- Test: `packages/blocks-ui-core/src/types/events.test.ts` (create if not exists)

**Interfaces:**
- Consumes: `Manifest`, `FullAgentDescriptor` from Tasks 1-2
- Produces: `AgentSetupEventTopics`, agent relationship type registrations

- [ ] **Step 1: Add event topics**

Add to `packages/blocks-ui-core/src/types/events.ts`:

```typescript
import type { Manifest } from './manifest.js';
import type { FullAgentDescriptor } from './agent.js';
import type { AgentRelationship } from '../../graph-stencil-org'; // or inline

export const AgentSetupEventTopics = {
  MANIFEST_CONFIGURED: 'manifest:configured',
  AGENT_SELECTED: 'agent:selected',
  AGENT_CREATED: 'agent:created',
  AGENT_UPDATED: 'agent:updated',
  RELATIONSHIP_CHANGED: 'relationship:changed',
} as const;
```

Note: check the actual import path for `AgentRelationship` — it may need to be defined locally if cross-package import is not viable. If so, define a lightweight `AgentRelationshipChangeset` interface inline.

- [ ] **Step 2: Register agent relationship kinds**

Add to `packages/blocks-ui-core/src/types/relationship.ts` after existing registrations:

```typescript
registerRelationshipType('supervises', {
  label: 'Supervises', color: '#6366f1', style: 'solid', directed: true,
});
registerRelationshipType('delegates_to', {
  label: 'Delegates To', color: '#8b5cf6', style: 'solid', directed: true,
});
registerRelationshipType('escalates_to', {
  label: 'Escalates To', color: '#ef4444', style: 'dashed', directed: true,
});
registerRelationshipType('reports_to', {
  label: 'Reports To', color: '#3b82f6', style: 'solid', directed: true,
});
registerRelationshipType('backs_up', {
  label: 'Backs Up', color: '#10b981', style: 'dotted', directed: true,
});
registerRelationshipType('extended', {
  label: 'Extended', color: '#6b7280', style: 'dashed', directed: true,
});
```

- [ ] **Step 3: Write tests**

```typescript
// Verify event topics are strings
import { AgentSetupEventTopics } from './events.js';
import { lookupRelationshipType } from './relationship.js';

describe('AgentSetupEventTopics', () => {
  it('defines all required topics', () => {
    expect(AgentSetupEventTopics.MANIFEST_CONFIGURED).toBe('manifest:configured');
    expect(AgentSetupEventTopics.AGENT_SELECTED).toBe('agent:selected');
    expect(AgentSetupEventTopics.AGENT_CREATED).toBe('agent:created');
    expect(AgentSetupEventTopics.AGENT_UPDATED).toBe('agent:updated');
    expect(AgentSetupEventTopics.RELATIONSHIP_CHANGED).toBe('relationship:changed');
  });
});

describe('agent relationship types', () => {
  it('registers SUPERVISES', () => {
    const desc = lookupRelationshipType('supervises');
    expect(desc).toBeDefined();
    expect(desc!.label).toBe('Supervises');
    expect(desc!.directed).toBe(true);
  });

  it('registers all 6 agent relationship kinds', () => {
    for (const kind of ['supervises', 'delegates_to', 'escalates_to', 'reports_to', 'backs_up', 'extended']) {
      expect(lookupRelationshipType(kind)).toBeDefined();
    }
  });
});
```

- [ ] **Step 4: Run tests and typecheck**

Run: `yarn workspace @casehubio/blocks-ui-core test && yarn workspace @casehubio/blocks-ui-core typecheck`

- [ ] **Step 5: Commit**

```bash
git add packages/blocks-ui-core/src/types/events.ts packages/blocks-ui-core/src/types/relationship.ts packages/blocks-ui-core/src/types/events.test.ts
git commit -m "feat(blocks-ui-core): add agent setup event topics and relationship kind registry"
```

---

## Batch 2: agent-avatar-2d package

### Task 4: Scaffold agent-avatar-2d package with canonical term registry

**Files:**
- Create: `packages/agent-avatar-2d/package.json`
- Create: `packages/agent-avatar-2d/tsconfig.json`
- Create: `packages/agent-avatar-2d/tsconfig.build.json`
- Create: `packages/agent-avatar-2d/src/index.ts`
- Create: `packages/agent-avatar-2d/src/canonical-registry.ts`
- Create: `packages/agent-avatar-2d/src/types.ts`
- Modify: `tsconfig.json` (root — add reference)
- Test: `packages/agent-avatar-2d/src/canonical-registry.test.ts`

**Interfaces:**
- Consumes: `AgentDisposition`, `DispositionAxis`, `primaryTerm` from blocks-ui-core
- Produces: `CanonicalTermRegistry`, `resolveFeatures()`, `AvatarFeatures`, `AvatarConfig`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/agent-avatar-2d",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "@dicebear/core": "^9.0.0",
    "@dicebear/avataaars": "^9.0.0",
    "lit": "^3.2.1"
  },
  "devDependencies": {
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  }
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src"],
  "references": [
    { "path": "../../packages/blocks-ui-core" }
  ]
}
```

Create `tsconfig.build.json`: `{ "extends": "./tsconfig.json", "exclude": ["src/**/*.test.ts"] }`

- [ ] **Step 3: Create types**

```typescript
// packages/agent-avatar-2d/src/types.ts
export interface AvatarFeatures {
  mouth?: string;
  eyes?: string;
  eyebrows?: string;
  clothing?: string;
  clothingGraphic?: string;
  accessories?: string;
  facialHair?: string;
  hairColor?: string;
  top?: string;
  skinColor?: string;
}

export interface AvatarConfig {
  features: AvatarFeatures;
  overrides?: Partial<AvatarFeatures>;
}
```

- [ ] **Step 4: Create canonical-registry with tests (TDD)**

Write test first:

```typescript
// packages/agent-avatar-2d/src/canonical-registry.test.ts
import { describe, it, expect } from 'vitest';
import { resolveFeatures, NEUTRAL_FEATURES } from './canonical-registry.js';
import type { AgentDisposition } from '@casehubio/blocks-ui-core';

describe('resolveFeatures', () => {
  it('returns neutral features for empty disposition', () => {
    const features = resolveFeatures({});
    expect(features).toEqual(NEUTRAL_FEATURES);
  });

  it('maps collaborative socialOrient to smile', () => {
    const disposition: AgentDisposition = {
      socialOrient: [{ term: 'collaborative', weight: 1.0 }],
    };
    const features = resolveFeatures(disposition);
    expect(features.mouth).toBe('smile');
    expect(features.eyes).toBe('happy');
  });

  it('maps cautious riskAppetite to glasses', () => {
    const disposition: AgentDisposition = {
      riskAppetite: [{ term: 'cautious', weight: 1.0 }],
    };
    const features = resolveFeatures(disposition);
    expect(features.accessories).toBe('prescription02');
    expect(features.clothing).toBe('blazerAndShirt');
  });

  it('uses primary term for multi-term axis', () => {
    const disposition: AgentDisposition = {
      socialOrient: [
        { term: 'collaborative', weight: 0.7 },
        { term: 'competitive', weight: 0.3 },
      ],
    };
    const features = resolveFeatures(disposition);
    expect(features.mouth).toBe('smile');
  });

  it('falls back to neutral for unknown vocabulary term', () => {
    const disposition: AgentDisposition = {
      socialOrient: [{ term: 'dialectical-engagement', weight: 1.0 }],
    };
    const features = resolveFeatures(disposition);
    expect(features.mouth).toBe(NEUTRAL_FEATURES.mouth);
  });

  it('combines features from multiple axes', () => {
    const disposition: AgentDisposition = {
      socialOrient: [{ term: 'collaborative', weight: 1.0 }],
      riskAppetite: [{ term: 'cautious', weight: 1.0 }],
      ruleFollowing: [{ term: 'strict', weight: 1.0 }],
    };
    const features = resolveFeatures(disposition);
    expect(features.mouth).toBe('smile');
    expect(features.accessories).toBe('prescription02');
    expect(features.eyebrows).toBe('defaultNatural');
  });
});
```

Then implement:

```typescript
// packages/agent-avatar-2d/src/canonical-registry.ts
import type { AgentDisposition, DispositionAxis } from '@casehubio/blocks-ui-core';
import { primaryTerm, DISPOSITION_AXES } from '@casehubio/blocks-ui-core';
import type { AvatarFeatures } from './types.js';

export const NEUTRAL_FEATURES: Readonly<AvatarFeatures> = {
  mouth: 'default',
  eyes: 'default',
  eyebrows: 'default',
  clothing: 'collarAndSweater',
  accessories: 'blank',
  top: 'shortCurly',
  skinColor: 'light',
};

type TermMapping = Record<string, Partial<AvatarFeatures>>;
type AxisMapping = Record<DispositionAxis, TermMapping>;

const AXIS_MAPPINGS: AxisMapping = {
  socialOrient: {
    collaborative: { mouth: 'smile', eyes: 'happy' },
    competitive: { mouth: 'serious', eyes: 'squint' },
    independent: { mouth: 'default', eyes: 'default' },
  },
  ruleFollowing: {
    strict: { eyebrows: 'defaultNatural', clothing: 'blazerAndShirt' },
    adaptive: { eyebrows: 'default', clothing: 'collarAndSweater' },
    creative: { eyebrows: 'raisedExcited', clothing: 'graphicShirt' },
  },
  riskAppetite: {
    adventurous: { accessories: 'sunglasses', clothing: 'hoodie' },
    moderate: { accessories: 'blank', clothing: 'collarAndSweater' },
    cautious: { accessories: 'prescription02', clothing: 'blazerAndShirt' },
  },
  autonomy: {
    'fully-autonomous': { top: 'shortFlat', facialHair: 'blank' },
    'semi-autonomous': { top: 'shortCurly', facialHair: 'blank' },
    directed: { top: 'shortRound', facialHair: 'blank' },
  },
  conflictMode: {
    assertive: { eyebrows: 'angryNatural', mouth: 'serious' },
    diplomatic: { eyebrows: 'default', mouth: 'smile' },
    avoidant: { eyebrows: 'sadConcerned', mouth: 'concerned' },
  },
};

export function resolveFeatures(disposition: AgentDisposition): AvatarFeatures {
  const result: AvatarFeatures = { ...NEUTRAL_FEATURES };

  for (const axis of DISPOSITION_AXES) {
    const term = primaryTerm(disposition, axis);
    if (!term) continue;

    const mapping = AXIS_MAPPINGS[axis];
    const features = mapping[term.term];
    if (features) {
      Object.assign(result, features);
    }
    // Unknown terms: fall back to neutral (no change) — deterministic
  }

  return result;
}
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/agent-avatar-2d test`

- [ ] **Step 6: Add root tsconfig reference**

Add `{ "path": "packages/agent-avatar-2d/tsconfig.build.json" }` to root `tsconfig.json` references.

- [ ] **Step 7: Commit**

```bash
git add packages/agent-avatar-2d/ tsconfig.json
git commit -m "feat(agent-avatar-2d): scaffold package with canonical term registry"
```

### Task 5: DiceBear integration and avatar generation

**Files:**
- Create: `packages/agent-avatar-2d/src/generator.ts`
- Create: `packages/agent-avatar-2d/src/agent-avatar.ts`
- Modify: `packages/agent-avatar-2d/src/index.ts`
- Test: `packages/agent-avatar-2d/src/generator.test.ts`
- Test: `packages/agent-avatar-2d/src/agent-avatar.test.ts`

**Interfaces:**
- Consumes: `resolveFeatures`, `AvatarFeatures`, `AvatarConfig` from Task 4
- Produces: `generateAvatar()`, `generateCandidates()`, `customiseAvatar()`, `<agent-avatar>` element

- [ ] **Step 1: Write generator tests**

```typescript
// packages/agent-avatar-2d/src/generator.test.ts
import { describe, it, expect } from 'vitest';
import { generateAvatar, generateCandidates, customiseAvatar } from './generator.js';
import type { AgentDisposition } from '@casehubio/blocks-ui-core';

describe('generateAvatar', () => {
  it('returns an SVG string', () => {
    const svg = generateAvatar({});
    expect(svg).toContain('<svg');
    expect(svg).toContain('</svg>');
  });

  it('is deterministic — same disposition produces same SVG', () => {
    const disposition: AgentDisposition = {
      socialOrient: [{ term: 'collaborative', weight: 1.0 }],
    };
    const svg1 = generateAvatar(disposition);
    const svg2 = generateAvatar(disposition);
    expect(svg1).toBe(svg2);
  });

  it('produces different SVGs for different dispositions', () => {
    const svg1 = generateAvatar({ socialOrient: [{ term: 'collaborative', weight: 1.0 }] });
    const svg2 = generateAvatar({ socialOrient: [{ term: 'competitive', weight: 1.0 }] });
    expect(svg1).not.toBe(svg2);
  });
});

describe('generateCandidates', () => {
  it('returns an array of SVGs', () => {
    const candidates = generateCandidates({});
    expect(Array.isArray(candidates)).toBe(true);
    expect(candidates.length).toBeGreaterThan(0);
    expect(candidates.length).toBeLessThanOrEqual(20);
    for (const svg of candidates) {
      expect(svg).toContain('<svg');
    }
  });
});

describe('customiseAvatar', () => {
  it('applies overrides to base config', () => {
    const base = { features: { mouth: 'smile', eyes: 'happy' } };
    const svg = customiseAvatar(base, { mouth: 'serious' });
    expect(svg).toContain('<svg');
  });
});
```

- [ ] **Step 2: Implement generator**

```typescript
// packages/agent-avatar-2d/src/generator.ts
import { createAvatar } from '@dicebear/core';
import * as avataaars from '@dicebear/avataaars';
import type { AgentDisposition } from '@casehubio/blocks-ui-core';
import { resolveFeatures, NEUTRAL_FEATURES } from './canonical-registry.js';
import type { AvatarFeatures, AvatarConfig } from './types.js';

function featuresToOptions(features: AvatarFeatures): Record<string, string[]> {
  const opts: Record<string, string[]> = {};
  for (const [key, value] of Object.entries(features)) {
    if (value) opts[key] = [value];
  }
  return opts;
}

export function generateAvatar(disposition: AgentDisposition): string {
  const features = resolveFeatures(disposition);
  const avatar = createAvatar(avataaars, { ...featuresToOptions(features), size: 128 });
  return avatar.toString();
}

export function generateCandidates(disposition: AgentDisposition): string[] {
  const baseFeatures = resolveFeatures(disposition);
  const candidates: string[] = [];

  const variations: Partial<AvatarFeatures>[] = [
    {},
    { top: 'longButNotTooLong' },
    { top: 'dreads' },
    { skinColor: 'tanned' },
    { skinColor: 'brown' },
    { skinColor: 'darkBrown' },
    { top: 'bob' },
    { top: 'bun' },
    { hairColor: 'auburn' },
    { hairColor: 'blonde' },
    { hairColor: 'black' },
    { hairColor: 'brown' },
  ];

  for (const variation of variations) {
    const features = { ...baseFeatures, ...variation };
    const avatar = createAvatar(avataaars, { ...featuresToOptions(features), size: 128 });
    candidates.push(avatar.toString());
  }

  return candidates;
}

export function customiseAvatar(base: AvatarConfig, overrides: Partial<AvatarFeatures>): string {
  const features = { ...base.features, ...base.overrides, ...overrides };
  const avatar = createAvatar(avataaars, { ...featuresToOptions(features), size: 128 });
  return avatar.toString();
}
```

- [ ] **Step 3: Run tests**

Run: `yarn install && yarn workspace @casehubio/agent-avatar-2d test`

Note: `yarn install` needed to resolve new DiceBear dependencies.

- [ ] **Step 4: Create the `<agent-avatar>` web component**

```typescript
// packages/agent-avatar-2d/src/agent-avatar.ts
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import { unsafeHTML } from 'lit/directives/unsafe-html.js';
import type { AgentDisposition } from '@casehubio/blocks-ui-core';
import { generateAvatar } from './generator.js';
import type { AvatarConfig } from './types.js';

export type AvatarSize = 'xs' | 'sm' | 'md' | 'lg';

const SIZE_PX: Record<AvatarSize, number> = { xs: 24, sm: 40, md: 64, lg: 128 };

@customElement('agent-avatar')
export class AgentAvatar extends LitElement {
  static styles = css`
    :host { display: inline-block; }
    .avatar-container { border-radius: 50%; overflow: hidden; }
    .avatar-container svg { display: block; width: 100%; height: 100%; }
  `;

  @property({ type: String }) size: AvatarSize = 'md';
  @property({ type: Object }) disposition?: AgentDisposition;
  @property({ type: Object }) config?: AvatarConfig;

  connectedCallback() {
    super.connectedCallback();
    this.setAttribute('role', 'img');
    if (!this.hasAttribute('aria-label')) {
      this.setAttribute('aria-label', 'Agent avatar');
    }
  }

  render() {
    const px = SIZE_PX[this.size];
    const svg = this.config
      ? generateAvatar(this.disposition ?? {})
      : generateAvatar(this.disposition ?? {});
    return html`
      <div class="avatar-container" style="width:${px}px;height:${px}px;">
        ${unsafeHTML(svg)}
      </div>
    `;
  }
}
```

- [ ] **Step 5: Write component test**

```typescript
// packages/agent-avatar-2d/src/agent-avatar.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './agent-avatar.js';

type AgentAvatarEl = HTMLElement & {
  disposition?: { socialOrient?: { term: string; weight: number }[] };
  size?: string;
  updateComplete: Promise<boolean>;
};

describe('agent-avatar', () => {
  let el: AgentAvatarEl;

  beforeEach(() => {
    el = document.createElement('agent-avatar') as AgentAvatarEl;
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
  });

  it('renders an SVG', async () => {
    await el.updateComplete;
    const svg = el.shadowRoot!.querySelector('svg');
    expect(svg).toBeTruthy();
  });

  it('has ARIA role=img and aria-label', async () => {
    await el.updateComplete;
    expect(el.getAttribute('role')).toBe('img');
    expect(el.getAttribute('aria-label')).toBe('Agent avatar');
  });

  it('accepts disposition property', async () => {
    el.disposition = { socialOrient: [{ term: 'collaborative', weight: 1.0 }] };
    await el.updateComplete;
    const svg = el.shadowRoot!.querySelector('svg');
    expect(svg).toBeTruthy();
  });
});
```

- [ ] **Step 6: Create index.ts barrel**

```typescript
// packages/agent-avatar-2d/src/index.ts
export * from './types.js';
export { resolveFeatures, NEUTRAL_FEATURES } from './canonical-registry.js';
export { generateAvatar, generateCandidates, customiseAvatar } from './generator.js';
export { AgentAvatar } from './agent-avatar.js';
```

- [ ] **Step 7: Run all tests and typecheck**

Run: `yarn workspace @casehubio/agent-avatar-2d test && yarn workspace @casehubio/agent-avatar-2d typecheck`

- [ ] **Step 8: Commit**

```bash
git add packages/agent-avatar-2d/
git commit -m "feat(agent-avatar-2d): DiceBear Avataaars integration with disposition-driven generation"
```

---

## Batch 3: agent-manifest-editor

### Task 6: Scaffold and provider card grid

**Files:**
- Create: `components/agent-manifest-editor/package.json`
- Create: `components/agent-manifest-editor/tsconfig.json`
- Create: `components/agent-manifest-editor/tsconfig.build.json`
- Create: `components/agent-manifest-editor/src/index.ts`
- Create: `components/agent-manifest-editor/src/types.ts`
- Create: `components/agent-manifest-editor/src/presets.ts`
- Create: `components/agent-manifest-editor/src/agent-manifest-editor.ts`
- Modify: `tsconfig.json` (root — add reference)
- Test: `components/agent-manifest-editor/src/agent-manifest-editor.test.ts`

**Interfaces:**
- Consumes: `Manifest`, `ProviderDeclaration`, `ModelDescriptor`, `CredentialRef`, `parseCredentialRef`, `formatCredentialRef`, `AgentSetupEventTopics` from blocks-ui-core
- Produces: `<agent-manifest-editor>` element, `ManifestPreset`, `PROVIDER_PRESETS`

This task creates the scaffold, types, preset definitions, and the
provider card grid with detection badges. The expanded provider section
(credential editing, model list, alias editor) follows in Task 7.

- [ ] **Step 1: Create package.json** (same pattern as avatar-2d, deps: `blocks-ui-core`, `@casehubio/pages-data`, `lit`)

- [ ] **Step 2: Create tsconfig.json and tsconfig.build.json** (same pattern)

- [ ] **Step 3: Create types.ts**

```typescript
// components/agent-manifest-editor/src/types.ts
import type { Manifest, ModelDescriptor, ProviderDeclaration } from '@casehubio/blocks-ui-core';

export interface ProviderDetection {
  vendor: string;
  detected: boolean;
  partial: boolean;
  envVarsFound?: string[];
  reachable?: boolean;
}

export interface ManifestPreset {
  id: string;
  name: string;
  description: string;
  manifest: Manifest;
}

export interface ManifestEditorProps {
  data?: Manifest;
  endpoint?: string;
  detectionEndpoint?: string;
  presets?: ManifestPreset[];
}
```

- [ ] **Step 4: Create presets.ts with built-in preset templates**

```typescript
// components/agent-manifest-editor/src/presets.ts
import type { ManifestPreset } from './types.js';

export const BUILT_IN_PRESETS: ManifestPreset[] = [
  {
    id: 'anthropic-production',
    name: 'Anthropic Production',
    description: 'Claude models for production use',
    manifest: {
      providers: [{ vendor: 'anthropic', credential: 'env:ANTHROPIC_API_KEY' }],
      models: [
        { id: 'claude-opus-5', displayName: 'Claude Opus 5', tier: 'FLAGSHIP', contextWindow: 1000000, costTier: 'PREMIUM' },
        { id: 'claude-sonnet-5', displayName: 'Claude Sonnet 5', tier: 'STANDARD', contextWindow: 200000, costTier: 'MEDIUM' },
        { id: 'claude-haiku-4-5', displayName: 'Claude Haiku 4.5', tier: 'FAST', contextWindow: 200000, costTier: 'LOW' },
      ],
      aliases: {
        'reasoning-heavy': { tier: 'FLAGSHIP', capabilities: ['reasoning', 'code'] },
        'general': { tier: 'STANDARD' },
        'fast': { tier: 'FAST' },
      },
      defaults: { backend: 'claude' },
    },
  },
  {
    id: 'openai-standard',
    name: 'OpenAI Standard',
    description: 'GPT models for general use',
    manifest: {
      providers: [{ vendor: 'openai', credential: 'env:OPENAI_API_KEY' }],
      models: [
        { id: 'gpt-4o', displayName: 'GPT-4o', tier: 'STANDARD', contextWindow: 128000, costTier: 'MEDIUM' },
        { id: 'gpt-4o-mini', displayName: 'GPT-4o Mini', tier: 'FAST', contextWindow: 128000, costTier: 'LOW' },
      ],
      defaults: { backend: 'openai' },
    },
  },
  {
    id: 'local-ollama',
    name: 'Local Development (Ollama)',
    description: 'Local models via Ollama — no API key needed',
    manifest: {
      providers: [{ vendor: 'ollama', host: 'http://localhost:11434' }],
      localModels: [{ id: 'llama3.1', backendKey: 'ollama', host: 'http://localhost:11434' }],
      defaults: { backend: 'ollama' },
    },
  },
  {
    id: 'multi-provider',
    name: 'Multi-Provider',
    description: 'Anthropic + OpenAI — route by capability',
    manifest: {
      providers: [
        { vendor: 'anthropic', credential: 'env:ANTHROPIC_API_KEY' },
        { vendor: 'openai', credential: 'env:OPENAI_API_KEY' },
      ],
      models: [
        { id: 'claude-sonnet-5', displayName: 'Claude Sonnet 5', vendor: 'anthropic', tier: 'STANDARD', contextWindow: 200000, costTier: 'MEDIUM' },
        { id: 'gpt-4o', displayName: 'GPT-4o', vendor: 'openai', tier: 'STANDARD', contextWindow: 128000, costTier: 'MEDIUM' },
      ],
      aliases: {
        'reasoning-heavy': { tier: 'FLAGSHIP', preferVendor: 'anthropic' },
        'general': { tier: 'STANDARD' },
      },
    },
  },
];
```

- [ ] **Step 5: Create the component with provider cards and write tests (TDD)**

Write tests first covering: renders provider cards, shows detection
badges, selects a preset, emits `manifest:configured` event. Then
implement the LitElement component with provider card grid, preset
selector, detection badge logic. Follow `gdpr-erasure-action` pattern
(extends LitElement directly, `@property endpoint`, `@property data`,
own fetch lifecycle).

- [ ] **Step 6: Add root tsconfig reference, run full test suite**

- [ ] **Step 7: Commit**

```bash
git add components/agent-manifest-editor/ tsconfig.json
git commit -m "feat(agent-manifest-editor): scaffold with provider cards, presets, and detection badges"
```

### Task 7: Expanded provider section — credential editing, model list, alias editor

**Files:**
- Modify: `components/agent-manifest-editor/src/agent-manifest-editor.ts`
- Test: `components/agent-manifest-editor/src/agent-manifest-editor.test.ts`

**Interfaces:**
- Consumes: `parseCredentialRef`, `formatCredentialRef`, `CredentialRef` from blocks-ui-core
- Produces: expanded provider section with credential type switcher, model checkboxes by tier, alias editor, test connection button

- [ ] **Step 1: Write tests for credential type switching**

Tests covering: credential type selector (env/file/ref), pre-fill from
detection, model tier grouping, alias editing, test connection button
emits validation event.

- [ ] **Step 2: Implement expanded section**

Progressive disclosure: clicking a provider card expands to show
credential entry (3-type switcher), model list (grouped by tier with
checkboxes), alias editor (key-value with capability tags), and test
connection button. Contextual help text at each field. "Getting started"
expandable section.

- [ ] **Step 3: Run tests, typecheck**

- [ ] **Step 4: Commit**

```bash
git add components/agent-manifest-editor/
git commit -m "feat(agent-manifest-editor): credential editing, model list, alias editor, test connection"
```

---

## Batch 4: agent-catalog

### Task 8: Agent catalog — filter bar and catalog grid

**Files:**
- Create: `components/agent-catalog/package.json`
- Create: `components/agent-catalog/tsconfig.json`
- Create: `components/agent-catalog/tsconfig.build.json`
- Create: `components/agent-catalog/src/index.ts`
- Create: `components/agent-catalog/src/types.ts`
- Create: `components/agent-catalog/src/catalog-data.ts`
- Create: `components/agent-catalog/src/filter-logic.ts`
- Create: `components/agent-catalog/src/agent-catalog.ts`
- Modify: `tsconfig.json` (root)
- Test: `components/agent-catalog/src/filter-logic.test.ts`
- Test: `components/agent-catalog/src/agent-catalog.test.ts`

**Interfaces:**
- Consumes: `FullAgentDescriptor`, `AgentSetupEventTopics`, `AgentDisposition`, `primaryTerm` from blocks-ui-core; `<agent-avatar>` from agent-avatar-2d
- Produces: `<agent-catalog>` element, `filterTemplates()`, `CatalogFilter`, `CATALOG_TEMPLATES`

- [ ] **Step 1: Create scaffold (package.json, tsconfig, types)**

- [ ] **Step 2: Create catalog-data.ts with built-in template descriptors**

Static JSON array of ~10 `FullAgentDescriptor` templates covering popular
presets: customer-support, software-developer, data-analyst,
medical-triage, legal-reviewer, financial-advisor, operations-coordinator,
security-analyst, content-writer, research-analyst.

- [ ] **Step 3: Write filter-logic.ts with TDD**

Pure function `filterTemplates(templates, filters, search)` — takes
array of descriptors, active filter pills (domain, taskType, disposition),
and search string. Returns filtered array. Write tests first covering:
single pill filter, multi-pill filter, search text, combination, empty
filters return all.

- [ ] **Step 4: Write agent-catalog component with TDD**

Tests: renders popular section, renders filter pills, clicking pill
filters the grid, clicking a card emits `agent:selected`, search filters
by name/description. ARIA: `role="region"`, `aria-label="Agent template catalog"`.

- [ ] **Step 5: Run tests, typecheck, add root reference**

- [ ] **Step 6: Commit**

```bash
git add components/agent-catalog/ tsconfig.json
git commit -m "feat(agent-catalog): template browsing with pill filters, popular section, avatar cards"
```

### Task 9: From-scratch wizard

**Files:**
- Create: `components/agent-catalog/src/agent-wizard.ts`
- Test: `components/agent-catalog/src/agent-wizard.test.ts`

**Interfaces:**
- Consumes: `FullAgentDescriptor`, `AgentDisposition`, `DISPOSITION_AXES`, `AgentSetupEventTopics`
- Produces: `<agent-wizard>` element (used inside agent-catalog)

- [ ] **Step 1: Write tests for wizard step navigation and output**

Tests: renders step indicators, navigates forward/back, step 1 captures
identity, step 3 disposition picks primary term (+ advanced toggle for
multi-term), step 6 shows avatar preview, completing emits
`agent:created` with valid `FullAgentDescriptor`. ARIA: `role="form"`,
`aria-label="Create custom agent"`.

- [ ] **Step 2: Implement 6-step wizard**

Steps: identity → capabilities (+ delegation toggle) → disposition →
goals/constraints → briefing → avatar. Preview panel builds up
progressively.

- [ ] **Step 3: Run tests, typecheck**

- [ ] **Step 4: Commit**

```bash
git add components/agent-catalog/
git commit -m "feat(agent-catalog): from-scratch agent wizard with 6-step flow"
```

---

## Batch 5: agent-profile

### Task 10: Agent profile — character sheet display

**Files:**
- Create: `components/agent-profile/package.json`
- Create: `components/agent-profile/tsconfig.json`
- Create: `components/agent-profile/tsconfig.build.json`
- Create: `components/agent-profile/src/index.ts`
- Create: `components/agent-profile/src/agent-profile.ts`
- Modify: `tsconfig.json` (root)
- Test: `components/agent-profile/src/agent-profile.test.ts`

**Interfaces:**
- Consumes: `FullAgentDescriptor`, `AgentDisposition`, `primaryTerm`, `DISPOSITION_AXES`, `AgentSetupEventTopics` from blocks-ui-core; `<agent-avatar>` from agent-avatar-2d
- Produces: `<agent-profile>` element

- [ ] **Step 1: Create scaffold**

- [ ] **Step 2: Write tests (TDD)**

Tests: renders header with name and avatar, renders capabilities cards,
renders disposition radar chart (verify SVG path generation), renders
goals and constraints, renders memory seed placeholder (greyed section),
inline edit toggle switches to form mode, edit emits `agent:updated`.
ARIA: `role="region"`, `aria-label` includes agent name.

- [ ] **Step 3: Implement component sections**

Header band, identity/briefing section, capabilities card grid,
disposition radar chart + per-axis rows with pills (matching org stencil
rendering), goals/constraints two-column, memory seed placeholder
(dashed border, greyed), relationships summary. Pencil icon per section
toggles inline edit.

- [ ] **Step 4: Run tests, typecheck, add root reference**

- [ ] **Step 5: Commit**

```bash
git add components/agent-profile/ tsconfig.json
git commit -m "feat(agent-profile): character sheet with disposition radar, capabilities, inline edit"
```

---

## Batch 6: agent-relationship-editor

### Task 11: Relationship table editing

**Files:**
- Create: `components/agent-relationship-editor/package.json`
- Create: `components/agent-relationship-editor/tsconfig.json`
- Create: `components/agent-relationship-editor/tsconfig.build.json`
- Create: `components/agent-relationship-editor/src/index.ts`
- Create: `components/agent-relationship-editor/src/types.ts`
- Create: `components/agent-relationship-editor/src/agent-relationship-editor.ts`
- Modify: `tsconfig.json` (root)
- Test: `components/agent-relationship-editor/src/agent-relationship-editor.test.ts`

**Interfaces:**
- Consumes: `AgentRelationship`, `RelationshipKind` from graph-stencil-org; `lookupRelationshipType` from blocks-ui-core; `AgentSetupEventTopics`
- Produces: `<agent-relationship-editor>` element, `RelationshipChangeset`

- [ ] **Step 1: Create scaffold and types**

```typescript
// components/agent-relationship-editor/src/types.ts
import type { AgentRelationship, RelationshipKind } from '@casehubio/graph-stencil-org';

export interface RelationshipChangeset {
  additions: AgentRelationship[];
  removals: AgentRelationship[];
}

export interface AgentRosterEntry {
  agentId: string;
  name: string;
  unitId?: string;
}
```

- [ ] **Step 2: Write tests (TDD)**

Tests: renders grouped sections for each relationship kind, shows
direction arrows, add form creates new relationship in pending state
(green highlight), delete marks as pending removal (red strikethrough),
emits `relationship:changed` with changeset on confirm, agent picker
dropdown populated from roster. ARIA: `role="region"`,
`aria-label="Relationship editor for [agentId]"`.

- [ ] **Step 3: Implement table component**

Grouped by `RelationshipKind`. Each section: header with kind label +
color from relationship registry, rows with direction/target/role/scope/delete,
inline add row with agent picker + kind selector + scope fields.
Before/after: pending additions green, pending deletions struck through.

- [ ] **Step 4: Run tests, typecheck, add root reference**

- [ ] **Step 5: Commit**

```bash
git add components/agent-relationship-editor/ tsconfig.json
git commit -m "feat(agent-relationship-editor): grouped table with add/remove and changeset emission"
```

### Task 12: Visualization views (arc diagram + ego diagram experiment)

**Files:**
- Create: `components/agent-relationship-editor/src/arc-view.ts`
- Create: `packages/graph-stencil-org/src/adapter/ego-subgraph.ts`
- Modify: `components/agent-relationship-editor/src/agent-relationship-editor.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`
- Test: `components/agent-relationship-editor/src/arc-view.test.ts`
- Test: `packages/graph-stencil-org/src/adapter/ego-subgraph.test.ts`

**Interfaces:**
- Consumes: `AgentRelationship[]`, `GraphModel` from graph-core
- Produces: `extractEgoSubgraph()`, `<arc-view>` element

- [ ] **Step 1: Write ego subgraph extraction with TDD**

Test first: given a full org graph model + ego agent ID, returns 1-hop
neighbourhood with the ego agent's unit as root container. Preserves
unit context. Agents not connected to the ego are excluded.

```typescript
// packages/graph-stencil-org/src/adapter/ego-subgraph.test.ts
import { describe, it, expect } from 'vitest';
import { extractEgoSubgraph } from './ego-subgraph.js';
import { toOrgGraph } from './org-adapter.js';

const ORG_YAML = `
organization:
  units:
    - unitId: team1
      name: Team One
      kind: rig
      tenancyId: t1
      members:
        - agentId: alice
          role: lead
        - agentId: bob
          role: worker
      capabilities: []
      goals: []
      constraints: []
    - unitId: team2
      name: Team Two
      kind: rig
      tenancyId: t1
      members:
        - agentId: carol
          role: lead
        - agentId: dave
          role: worker
      capabilities: []
      goals: []
      constraints: []
  relationships:
    - sourceAgentId: alice
      targetAgentId: bob
      kind: SUPERVISES
      tenancyId: t1
    - sourceAgentId: alice
      targetAgentId: carol
      kind: DELEGATES_TO
      tenancyId: t1
    - sourceAgentId: carol
      targetAgentId: dave
      kind: SUPERVISES
      tenancyId: t1
`;

describe('extractEgoSubgraph', () => {
  it('returns only 1-hop neighbours of ego agent', () => {
    const { model } = toOrgGraph(ORG_YAML);
    const sub = extractEgoSubgraph(model, 'alice');
    const agentIds = sub.nodes
      .filter(n => n.type === 'org-agent')
      .map(n => n.properties['agentId']);
    expect(agentIds).toContain('alice');
    expect(agentIds).toContain('bob');
    expect(agentIds).toContain('carol');
    expect(agentIds).not.toContain('dave');
  });

  it('preserves unit context for ego agent', () => {
    const { model } = toOrgGraph(ORG_YAML);
    const sub = extractEgoSubgraph(model, 'alice');
    const units = sub.nodes.filter(n => n.type === 'org-unit');
    expect(units.length).toBeGreaterThan(0);
  });

  it('includes edges to/from ego only', () => {
    const { model } = toOrgGraph(ORG_YAML);
    const sub = extractEgoSubgraph(model, 'alice');
    expect(sub.edges).toHaveLength(2);
  });
});
```

Then implement `extractEgoSubgraph(model, egoAgentId)`.

- [ ] **Step 2: Write arc-view component with TDD**

SVG arc diagram: ego on left, connected agents right, semi-circle arcs
above (outgoing) and below (incoming), colored by relationship kind.
Tests: renders SVG, correct arc count, hover highlight.

- [ ] **Step 3: Add view tab strip to relationship editor**

Tab strip above table: Table (default) | Arc | Ego Diagram. Table tab
shows the editing table. Arc tab shows arc-view (read-only). Ego Diagram
tab renders the filtered org subgraph via graph-stencil-org (experimental).

- [ ] **Step 4: Export ego subgraph from graph-stencil-org index**

Add `export { extractEgoSubgraph } from './adapter/ego-subgraph.js';` to
`packages/graph-stencil-org/src/index.ts`.

- [ ] **Step 5: Run all tests, typecheck**

- [ ] **Step 6: Commit**

```bash
git add components/agent-relationship-editor/ packages/graph-stencil-org/
git commit -m "feat(agent-relationship-editor): arc diagram view + ego subgraph extraction"
```

---

## Batch 7: Integration and workspace registration

### Task 13: Workspace registration, full build, schema updates

**Files:**
- Modify: `tsconfig.json` (root — verify all references)
- Modify: `packages/blocks-ui-schema/` (add new components to registry if applicable)
- Modify: `CLAUDE.md` (add new component directory descriptions)

- [ ] **Step 1: Verify all root tsconfig references are present**

Ensure references for: `agent-avatar-2d`, `agent-manifest-editor`,
`agent-catalog`, `agent-profile`, `agent-relationship-editor`.

- [ ] **Step 2: Run full build**

Run: `yarn build`

- [ ] **Step 3: Run full test suite**

Run: `yarn test`

- [ ] **Step 4: Run typecheck**

Run: `yarn typecheck`

- [ ] **Step 5: Run aria-check**

Run: `yarn aria-check`

- [ ] **Step 6: Update CLAUDE.md key directories**

Add entries for each new component and package to the Key Directories
table in CLAUDE.md.

- [ ] **Step 7: Commit**

```bash
git add tsconfig.json CLAUDE.md packages/blocks-ui-schema/
git commit -m "feat: register agent setup components in workspace, update CLAUDE.md"
```

---

## References

- [2026-09-20-agent-setup-wizard-design.md] — design spec this plan implements
- [decisions.md] — 8 design decisions + 2 constraints
- [packages/blocks-ui-core/src/types/] — existing type mirror patterns
- [components/service-card/] — component scaffold reference
- [components/gdpr-erasure-action/] — non-DataSourceMixin component pattern
- [packages/graph-stencil-org/src/types.ts] — existing AgentDescriptor, DispositionAxes mirrors
- [packages/graph-stencil-org/src/adapter/enrichment.ts] — enrichWithDescriptors pattern
- [packages/avatar/] — existing 3D TalkingHead avatar system
- [platform/agent-config-core Manifest.java] — Java source for type mirrors
- [eidos AgentDescriptor, AgentDisposition] — Java source for agent types
- platform#285, platform#315, eidos#175, blocks-ui#157 — related issues
