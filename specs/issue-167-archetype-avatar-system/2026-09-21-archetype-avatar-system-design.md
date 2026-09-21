# Archetype Avatar System — Design Spec

Replace DiceBear Avataaars with a purpose-built configurable avatar platform driven by the Hartwell & Chen archetype model. 12 families × 4 sub-archetypes = 48 preset configurations, rendered from composable SVG parts via a mechanical builder. Extensible via themed collections.

## Architecture

### Data Flow

```
eidos (server)                          blocks-ui (client)
┌──────────────────┐                    ┌──────────────────────────────┐
│ ArchetypeResolver│                    │ <agent-avatar>               │
│ disposition →    │──→ API ──→         │                              │
│ archetype identity                    │  1. Read config              │
│                  │                    │  2. Resolve: preset+overrides│
│ Agent YAML:      │                    │  3. Palette from family      │
│   avatar:mythic:P1B                   │  4. Part renderers → SVG    │
└──────────────────┘                    │  5. Assemble layers          │
                                        │  6. Render in shadow DOM     │
                                        └──────────────────────────────┘
```

### Three Visual Layers (priority order)

1. **Primary:** Family + sub-archetype → base template (head shape, body, costume, props, palette)
2. **Secondary:** Adjectives → expression/styling refinements (intensity, temperament, energy, precision, organic)
3. **Tertiary:** Canonical axes → subtle expression geometry (brow angle, mouth curve, eye openness)

### Size-Responsive Rendering

| Size | Pixels | Detail Level |
|------|--------|-------------|
| xs | 24px | Silhouette + palette only. Head shape is the differentiator. |
| sm | 40px | Head shape + palette + hair outline. Glasses hint if present. |
| md | 64px | Full template. Props rendered. Facial features present. |
| lg | 128px | Full detail — props, facial hair, adjective effects, axis modifiers. |

## Component API

### Input

The `<agent-avatar>` component accepts a single archetype payload. Clean break from the old `disposition` property — DiceBear is removed entirely.

```typescript
interface AvatarPayload {
  family: ArchetypeFamily;
  subArchetype: string;
  adjectives?: string[];
  canonicalAxes?: Partial<Record<DispositionAxis, { term: string; weight: number }>>;
  overrides?: Partial<PartAssignment>;
}

type ArchetypeFamily =
  | 'Caregiver' | 'Everyman' | 'Creator' | 'Innocent'
  | 'Explorer'  | 'Hero'     | 'Jester'  | 'Lover'
  | 'Magician'  | 'Rebel'    | 'Sage'    | 'Sovereign';

interface PartAssignment {
  head: string;
  hair: string;
  facialHair: string;
  costume: string;
  props: string[];
  glasses: string | null;
  eyebrows: string;
  accessories: string[];
}
```

### Component

```typescript
@customElement('agent-avatar')
export class AgentAvatar extends LitElement {
  @property({ type: String }) size: 'xs' | 'sm' | 'md' | 'lg' = 'md';
  @property({ type: Object }) archetype?: AvatarPayload;
  @property({ type: String }) collection: string = 'mythic';
  @property({ type: String }) code?: string; // alternative: compact code e.g. "mythic:P1B"
}
```

Two input modes:
- **Payload mode:** set `archetype` property with the full object
- **Code mode:** set `code` property with the compact identity string (e.g., `mythic:P1B`)

If both set, `archetype` takes precedence. If neither, renders the Everyman/Citizen fallback in desaturated greyscale.

### ARIA

```typescript
connectedCallback() {
  this.setAttribute('role', 'img');
  if (!this.hasAttribute('aria-label')) {
    this.setAttribute('aria-label',
      `${this.archetype?.family ?? 'Unknown'} ${this.archetype?.subArchetype ?? ''} agent avatar`);
  }
}
```

## Mechanical Composition

### Part Registry

Each part category has a finite set of options. Parts are abstract semantic identifiers — the rendering is provided by the active collection.

```typescript
interface PartRegistry {
  heads: Map<string, HeadRenderer>;
  hairs: Map<string, HairRenderer>;
  facialHairs: Map<string, FacialHairRenderer>;
  costumes: Map<string, CostumeRenderer>;
  props: Map<string, PropRenderer>;
  glasses: Map<string, GlassesRenderer>;
  eyebrows: Map<string, EyebrowRenderer>;
  accessories: Map<string, AccessoryRenderer>;
}

type PartRenderer = (palette: FamilyPalette, modifiers: PartModifiers) => string;
```

Every renderer is a pure function: `(palette, modifiers) → SVG path string`. No side effects, no DOM access, fully testable.

### Archetype Config Table

The 48 archetype presets are a lookup table. Each row maps an archetype to a specific combination of parts. This table is the TypeScript translation of `part-catalogue.md`.

```typescript
const ARCHETYPE_CONFIGS: Record<string, PartAssignment> = {
  'Sage/Detective': {
    head: 'oval',
    hair: 'bald-sides',
    facialHair: 'none',
    costume: 'blazer-tie',
    props: ['magnifying-glass', 'notebook'],
    glasses: 'round-wire',
    eyebrows: 'thin-arched',
    accessories: [],
  },
  'Sage/Mentor': {
    head: 'oval',
    hair: 'wild-einstein',
    facialHair: 'bushy-white',
    costume: 'tweed-patches',
    props: ['book-open', 'chalk'],
    glasses: 'half-rim',
    eyebrows: 'bushy-wild',
    accessories: [],
  },
  // ... 46 more entries from part-catalogue.md
};
```

### Builder

The builder assembles SVG by stacking layers in z-order:

```typescript
function buildAvatar(
  config: PartAssignment,
  palette: FamilyPalette,
  size: AvatarSize,
  modifiers: AvatarModifiers,
  registry: PartRegistry,
): string {
  const layers: string[] = [];
  const detail = DETAIL_TIERS[size];

  layers.push(registry.costumes.get(config.costume)!(palette, modifiers));
  layers.push(registry.heads.get(config.head)!(palette, modifiers));
  layers.push(registry.hairs.get(config.hair)!(palette, modifiers));

  if (detail >= DetailLevel.MD) {
    if (config.facialHair !== 'none') {
      layers.push(registry.facialHairs.get(config.facialHair)!(palette, modifiers));
    }
    layers.push(renderFace(config.eyebrows, modifiers, registry));
    if (config.glasses) {
      layers.push(registry.glasses.get(config.glasses)!(palette, modifiers));
    }
    for (const prop of config.props) {
      layers.push(registry.props.get(prop)!(palette, modifiers));
    }
  }

  if (detail >= DetailLevel.LG) {
    for (const acc of config.accessories) {
      layers.push(registry.accessories.get(acc)!(palette, modifiers));
    }
  }

  return assembleSvg(layers, size);
}
```

### Adjective Modifiers

Adjectives map to one of 5 effect categories. Each category modifies specific layers:

| Category | Effect | Layers Affected |
|----------|--------|----------------|
| Intensity (fierce, gentle, bold, subtle) | Expression tension — brow angle, mouth set | Face (eyebrows, mouth) |
| Temperament (warm, cool, measured, passionate) | Colour temperature, saturation shifts | Palette accents |
| Energy (restless, calm, driven, contemplative) | Pose dynamism — implied motion vs static | Body, hair |
| Precision (meticulous, precise, methodical, sharp) | Line quality — clean edges, geometric | All part outlines |
| Organic (intuitive, flowing, natural, earthy) | Shape language — softer curves, textures | All part shapes |

Modifiers are passed to part renderers via the `PartModifiers` object. Each renderer decides how to apply them (or ignore them at smaller sizes).

### Canonical Axis Expression Geometry

The 5 canonical axes map to discrete expression variants (not continuous interpolation). Applied at md+ sizes only.

| Axis | Visual Effect |
|------|-------------|
| socialOrientation | independent → closed expression · collaborative → open/welcoming |
| ruleFollowing | strict → angular features · flexible → relaxed features |
| riskAppetite | cautious → narrower eyes, tension · bold → wider eyes, confident |
| autonomy | autonomous → self-contained posture · directed → attentive posture |
| conflictMode | competing → assertive jaw/brow · avoiding → withdrawn · accommodating → yielding |

## Collections

### Concept

Part IDs are abstract. A **collection** provides concrete SVG renderers for every part ID. Different collections render the same archetype in different visual styles.

```typescript
interface AvatarCollection {
  id: string;                    // e.g., 'mythic'
  name: string;                  // e.g., 'Mythic'
  version: number;               // renderer version for cache invalidation
  registry: PartRegistry;        // concrete renderers for all part IDs
  palettes: Record<ArchetypeFamily, FamilyPalette>;
  fallbackRenderer: PartRenderer; // used when a part ID has no specific renderer
}
```

### mythic — First Collection

The inaugural collection. Flat illustration style with half-body framing (head to waist).

Visual language:
- Distinct head shapes per family (round, square jaw, diamond, heart, oval, angular, etc.)
- Family colour palettes (cool blues for Sage, deep purples for Magician, bold primaries for Hero)
- Archetype-specific props (magnifying glass, orb, paintbrush, shield, crown, compass, etc.)
- Character variety (8+ hair styles, 8+ facial hair, 8 glasses, 8 eyebrow types)
- Size-responsive detail tiers (xs silhouette → lg full detail)

Full visual reference: `avatar-preview.html`
Part assignments: `part-catalogue.md`

### Registering a Collection

```typescript
import { registerCollection } from '@casehubio/agent-avatar-2d';
import { mythicCollection } from './collections/mythic.js';

registerCollection(mythicCollection);
```

The default collection is `mythic`. If a compact code references a collection that isn't registered, the component falls back to `mythic`.

## Compact Avatar Identity

### Encoding

Every avatar has a reproducible compact code: `collection:config`.

**Tier 1 — Preset (no overrides):**
```
mythic:P1B
```
- `mythic` — collection slug
- `P` — preset marker
- `1B` — archetype index in base36 (0-47 → "0"-"1B")

**Tier 2 — Customised (user overrides):**
```
mythic:Co/grLNBF
```
- `C` — custom marker
- 8 base64 chars encoding 44 bits of part selections:
  - head (4b) + hair (4b) + facialHair (4b) + costume (5b)
  - prop1 (6b) + prop2 (6b) + glasses (4b)
  - eyebrows (3b) + accessory (4b) + palette (4b)

### Properties

- **Deterministic:** same code → same SVG, always
- **Decodable:** code → full part list without database lookup
- **Compact:** 6-15 characters total
- **Versionable:** collection renderers can evolve; codes stay stable (append-only part indices)

### eidos YAML Integration

The avatar code is stored as a field on the agent definition:

```yaml
agent:
  name: Inspector Morse
  archetype: Sage/Detective
  avatar: mythic:P1B
  disposition:
    socialOrient: [{ term: independent, weight: 1.0 }]
```

When `avatar` is absent, the UI derives it from `archetype` using the default collection and preset config.

## Package Structure

All new code lives in `packages/agent-avatar-2d/src/`, replacing the current dist-only package. The package name (`@casehubio/agent-avatar-2d`) and element tag (`agent-avatar`) are unchanged.

```
packages/agent-avatar-2d/
├── src/
│   ├── index.ts                    # public API exports
│   ├── agent-avatar.ts             # <agent-avatar> LitElement component
│   ├── types.ts                    # AvatarPayload, PartAssignment, FamilyPalette, etc.
│   ├── builder.ts                  # buildAvatar() — layer assembly
│   ├── config-table.ts             # ARCHETYPE_CONFIGS — 48 preset mappings
│   ├── palettes.ts                 # FAMILY_PALETTES — 12 colour palettes
│   ├── code.ts                     # encode/decode compact avatar codes
│   ├── modifiers.ts                # adjective→effect mapping, axis→expression mapping
│   ├── collections/
│   │   ├── registry.ts             # registerCollection(), getCollection()
│   │   └── mythic/
│   │       ├── index.ts            # mythicCollection export
│   │       ├── heads.ts            # head shape renderers (12)
│   │       ├── hairs.ts            # hair style renderers (8+)
│   │       ├── facial-hairs.ts     # facial hair renderers (8+)
│   │       ├── costumes.ts         # costume renderers (20)
│   │       ├── props.ts            # prop renderers (48)
│   │       ├── glasses.ts          # glasses renderers (8)
│   │       ├── eyebrows.ts         # eyebrow renderers (8)
│   │       └── accessories.ts      # accessory renderers (12)
│   └── __tests__/
│       ├── agent-avatar.test.ts    # component tests + ARIA
│       ├── builder.test.ts         # layer assembly tests
│       ├── config-table.test.ts    # preset completeness, uniqueness
│       ├── code.test.ts            # encode/decode roundtrip
│       └── modifiers.test.ts       # adjective/axis mapping tests
├── package.json
├── tsconfig.build.json
├── tsconfig.json
└── vitest.config.ts
```

### BlocksComponentRegistry

Per protocol `component-registry-props.md`: export `AgentAvatarProps` from `index.ts` and register in `packages/blocks-ui-schema/src/registry.ts`.

## DiceBear Removal

Remove `@dicebear/core` and `@dicebear/avataaars` from package.json. The old `generateAvatar()`, `generateCandidates()`, `customiseAvatar()` functions are replaced by the builder. The old `canonical-registry.ts` AXIS_MAPPINGS are superseded by the canonical axis expression modifiers.

### Fallback Behaviour

When archetype data is missing (no `archetype` property, no `code`), render the Everyman/Citizen preset with a desaturated greyscale palette. Visually distinct from any assigned archetype — signals "identity not resolved" without breaking layout.

## Testing Strategy

| Layer | Test Approach |
|-------|--------------|
| Part renderers | Each renderer returns valid SVG (well-formed XML, non-empty) for all 12 palettes |
| Config table | 48 entries, every archetype mapped, uniqueness audit (≥2 parts different within family) |
| Builder | Layer ordering, size-tier filtering (xs omits props, sm omits facial detail) |
| Code encode/decode | Roundtrip: config → code → config is identity. Preset detection. |
| Component | ARIA attributes (role=img, aria-label), both input modes (payload vs code), fallback |
| Collection | Default collection registered, fallback for unknown collection |
| Adjective modifiers | Valid adjectives per archetype accepted, invalid rejected |

## Migration Path

### Existing Consumers

The `<agent-avatar>` element tag is unchanged. Consumers need to update the data contract:

| Before | After |
|--------|-------|
| `<agent-avatar .disposition=${d}>` | `<agent-avatar .archetype=${a}>` |
| `<agent-avatar .disposition=${d} size="lg">` | `<agent-avatar .archetype=${a} size="lg">` |
| — | `<agent-avatar code="mythic:P1B">` (new) |

Affected components: agent-catalog, agent-profile, agent-wizard (all on the paused issue-166 branch).

### Transition

Server-side `ArchetypeResolver` already converges disposition → archetype identity. Agents store the resolved archetype as part of their entity. Frontend callers receive archetype data alongside other agent metadata via existing API endpoints — no new frontend resolver needed.

## Cross-Repo Concerns

| Concern | Repo | Action |
|---------|------|--------|
| `avatar` field in agent YAML schema | eidos | Add optional `avatar: string` field to agent definition |
| ArchetypeResolver → avatar code | eidos | Generate `mythic:P<index>` default code when archetype resolves |
| Part catalogue rendering | blocks-ui | This issue — the SVG renderers |
| Faceted selector UI | blocks-ui | Separate future issue — consumes avatar system for preview |

## References

- eidos `specs/avatar-generator-contract.md` — archetype model, selection algorithm, adjective catalog, visual generation schema
- eidos `specs/archetype-compatibility-matrix.md` — compatibility data for 6 personality frameworks
- `packages/agent-avatar-2d/dist/` — current DiceBear implementation (to be replaced)
- `packages/blocks-ui-core/dist/types/agent.d.ts` — AgentDisposition, DispositionAxis types
- `docs/protocols/blocks-ui/component-registry-props.md` — Props interface + BlocksComponentRegistry requirement
- `docs/protocols/blocks-ui/component-customisation-pattern.md` — typed config properties pattern
- `part-catalogue.md` — full part assignments for all 48 sub-archetypes
- `avatar-preview.html` — live visual reference (12 family roots + part catalogues)
- `decisions.md` — all 15 design decisions with rationale and trade-offs
