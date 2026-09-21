# Archetype Avatar System — Design Spec

Replace DiceBear Avataaars with a purpose-built configurable avatar platform driven by the Hartwell & Chen archetype model. 12 families × 4 sub-archetypes = 48 preset configurations, rendered from composable SVG parts via a mechanical builder. Extensible via themed collections.

**Note on count:** The issue title and eidos contract header say "60 sub-archetypes" (12 × 5), but the contract data and ArchetypeTerm.java list exactly 4 per family = 48. This spec uses 48. If eidos adds a 5th sub-archetype per family, the system accommodates it — a new row in the config table and new part renderers for any new props.

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

**A collection IS two SVG files.** That's the entire deliverable:

| File | Purpose | Contents |
|------|---------|----------|
| `mythic.parts.svg` | Construction — all parts for mechanical composition | ~140 `<symbol>` elements (heads, hairs, costumes, props, etc.) |
| `mythic.preview.svg` | Browsing — pre-rendered family roots for theme selection | 12 `<symbol>` elements, one per family, fully composed |

Drop two files in = new collection. No TypeScript, no config, no registration code.

### Parts File — `{collection}.parts.svg`

A single SVG file containing every composable part as a named `<symbol>`. The builder locates parts by ID using the convention `{category}:{part-id}`.

```xml
<!-- mythic.parts.svg -->
<svg xmlns="http://www.w3.org/2000/svg">
  <!-- ═══ Heads ═══ -->
  <symbol id="head:oval" viewBox="0 0 200 240">
    <ellipse cx="100" cy="95" rx="38" ry="42" fill="var(--skin)"/>
  </symbol>
  <symbol id="head:round" viewBox="0 0 200 240">
    <circle cx="100" cy="90" r="42" fill="var(--skin)"/>
  </symbol>
  <symbol id="head:square-jaw" viewBox="0 0 200 240">
    <path d="M65,55 Q62,95 75,115 ..." fill="var(--skin)"/>
  </symbol>

  <!-- ═══ Hairs ═══ -->
  <symbol id="hair:bald-sides" viewBox="0 0 200 240">
    <path d="M62,88 Q62,55 80,50 ..." fill="var(--hair-color)"/>
  </symbol>
  <symbol id="hair:wild-einstein" viewBox="0 0 200 240">
    <path d="M55,80 Q45,50 60,35 ..." fill="var(--hair-color)"/>
  </symbol>

  <!-- ═══ Costumes ═══ -->
  <symbol id="costume:blazer-tie" viewBox="0 0 200 240">
    <path d="M60,240 L60,160 ..." fill="var(--primary)"/>
    <path d="M75,145 L100,160 ..." fill="var(--accent)"/>
  </symbol>

  <!-- ═══ Props ═══ -->
  <symbol id="prop:magnifying-glass" viewBox="0 0 200 240">
    <circle cx="158" cy="175" r="18" fill="none" stroke="var(--accent)" stroke-width="4"/>
    <line x1="145" y1="188" x2="132" y2="210" stroke="var(--accent)" stroke-width="5"/>
  </symbol>

  <!-- ═══ Glasses ═══ -->
  <symbol id="glasses:round-wire" viewBox="0 0 200 240">
    <circle cx="85" cy="92" r="12" fill="none" stroke="#333" stroke-width="2.5"/>
    <circle cx="115" cy="92" r="12" fill="none" stroke="#333" stroke-width="2.5"/>
    <line x1="97" y1="92" x2="103" y2="92" stroke="#333" stroke-width="2"/>
  </symbol>

  <!-- ═══ Eyebrows ═══ -->
  <symbol id="brow:thin-arched" viewBox="0 0 200 240">
    <path d="M74,81 Q80,77 96,80" fill="none" stroke="var(--hair-color)" stroke-width="2.5"/>
    <path d="M126,81 Q120,77 104,80" fill="none" stroke="var(--hair-color)" stroke-width="2.5"/>
  </symbol>

  <!-- ═══ Facial Hair ═══ -->
  <symbol id="beard:goatee" viewBox="0 0 200 240">
    <path d="M93,118 Q96,125 100,138 Q104,125 107,118" fill="var(--hair-color)"/>
  </symbol>

  <!-- ═══ Accessories ═══ -->
  <symbol id="acc:ear-piercings" viewBox="0 0 200 240">
    <circle cx="63" cy="85" r="2.5" fill="#888"/>
    <circle cx="63" cy="85" r="1.2" fill="#c0c0c0"/>
  </symbol>

  <!-- ... ~130 more symbols -->
</svg>
```

Parts use CSS custom properties for palette colouring:
- `var(--skin)` — skin tone
- `var(--primary)` — family primary colour
- `var(--secondary)` — family secondary colour
- `var(--accent)` — family accent colour
- `var(--hair-color)` — hair/beard colour

The builder sets these as inline styles on the wrapping SVG when composing.

### Preview File — `{collection}.preview.svg`

Pre-rendered family root avatars — one fully composed avatar per family. Used by the collection/theme browser to show what a collection looks like without running the builder.

```xml
<!-- mythic.preview.svg -->
<svg xmlns="http://www.w3.org/2000/svg">
  <symbol id="preview:Caregiver" viewBox="0 0 200 240">
    <!-- Complete Caregiver root avatar — all layers pre-composed -->
  </symbol>
  <symbol id="preview:Sage" viewBox="0 0 200 240">
    <!-- Complete Sage root avatar -->
  </symbol>
  <!-- ... 10 more family previews -->
</svg>
```

A theme browser renders all 12 previews in a row per collection:

```html
<!-- Theme browser — one row per collection -->
<div class="collection-row">
  <span>Mythic</span>
  <svg viewBox="0 0 200 240" width="48"><use href="mythic.preview.svg#preview:Caregiver"/></svg>
  <svg viewBox="0 0 200 240" width="48"><use href="mythic.preview.svg#preview:Sage"/></svg>
  <svg viewBox="0 0 200 240" width="48"><use href="mythic.preview.svg#preview:Hero"/></svg>
  <!-- ... 9 more -->
</div>
<div class="collection-row">
  <span>Pixel</span>
  <svg viewBox="0 0 200 240" width="48"><use href="pixel.preview.svg#preview:Caregiver"/></svg>
  <!-- ... -->
</div>
```

At a glance, you see every collection's interpretation of all 12 families. Pick a row = pick a theme.

### Collection Loading

```typescript
interface AvatarCollection {
  id: string;
  partsUrl: string;       // URL to {collection}.parts.svg
  previewUrl: string;     // URL to {collection}.preview.svg
  parts: Map<string, string>;  // populated after load: symbol ID → SVG content
}

async function loadCollection(id: string, partsUrl: string, previewUrl: string): Promise<AvatarCollection>;
function registerCollection(collection: AvatarCollection): void;
```

For the built-in `mythic` collection, the parts file is inlined at build time (extracted into a TypeScript map) for zero-network rendering. Third-party collections load at runtime via fetch.

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

## Build Pipeline

### Source of Truth → Runtime

The `.parts.svg` file is the source of truth. A build step extracts its `<symbol>` elements into a TypeScript map for the built-in collection (zero-network runtime rendering per D9):

```typescript
// GENERATED — do not edit. Source: mythic.parts.svg
export const MYTHIC_PARTS: Record<string, string> = {
  'head:oval': '<ellipse cx="100" cy="95" rx="38" ry="42" fill="var(--skin)"/>',
  'head:round': '<circle cx="100" cy="90" r="42" fill="var(--skin)"/>',
  'costume:blazer-tie': '<path d="M60,240 L60,160 ..." fill="var(--primary)"/>...',
  'prop:magnifying-glass': '<circle cx="158" cy="175" r="18" .../>...',
  // ... ~140 entries
};
```

Third-party collections skip the build step — they load at runtime via fetch and DOM parsing of the `.parts.svg` file.

### Public API

```typescript
function renderAvatar(code: string): string
```

Give it a compact code (e.g., `mythic:P1B`), get back a complete SVG string. Internally: decode → resolve preset + overrides → look up parts from collection → apply palette as CSS custom properties → assemble layers → return SVG string.

## Package Structure

All new code lives in `packages/agent-avatar-2d/src/`, replacing the current dist-only package. The package name (`@casehubio/agent-avatar-2d`) and element tag (`agent-avatar`) are unchanged.

```
packages/agent-avatar-2d/
├── src/
│   ├── index.ts                    # public API: renderAvatar(), AgentAvatar, encode/decode
│   ├── agent-avatar.ts             # <agent-avatar> LitElement component
│   ├── types.ts                    # AvatarPayload, PartAssignment, FamilyPalette, etc.
│   ├── builder.ts                  # buildAvatar() — layer assembly, palette injection
│   ├── config-table.ts             # ARCHETYPE_CONFIGS — 48 preset mappings
│   ├── palettes.ts                 # FAMILY_PALETTES — 12 colour palettes
│   ├── code.ts                     # encode/decode compact avatar codes
│   ├── modifiers.ts                # adjective→effect mapping, axis→expression mapping
│   ├── collections/
│   │   ├── registry.ts             # registerCollection(), loadCollection()
│   │   └── mythic/
│   │       ├── mythic.parts.svg    # SOURCE OF TRUTH — all ~140 parts as <symbol> elements
│   │       ├── mythic.preview.svg  # 12 pre-rendered family root avatars for theme browsing
│   │       ├── mythic-parts.ts     # GENERATED from mythic.parts.svg — do not edit
│   │       └── index.ts            # mythicCollection export (wraps generated parts)
│   ├── scripts/
│   │   └── extract-parts.ts        # SVG parts file → TypeScript part map
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
