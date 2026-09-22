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
  hat: string | null;
  facialHair: string;
  expression: string | null;
  costume: string;
  props: string[];
  glasses: string | null;
  eyebrows: string;
  accessories: string[];
}

interface AvatarModifiers {
  adjectives?: string[];
  canonicalAxes?: Partial<Record<DispositionAxis, { term: string; weight: number }>>;
}

interface PartModifiers {
  intensity: number;    // -1 gentle/subtle … 0 neutral … +1 fierce/bold
  temperament: number;  // -1 cool/measured … 0 neutral … +1 warm/passionate
  energy: number;       // -1 calm/contemplative … 0 neutral … +1 restless/driven
  precision: number;    // -1 loose/rough … 0 neutral … +1 meticulous/geometric
  organic: number;      // -1 structured/angular … 0 neutral … +1 flowing/natural
  expression: AxisExpression;
}

interface AxisExpression {
  socialOrientation?: 'independent' | 'collaborative';
  ruleFollowing?: 'strict' | 'flexible';
  riskAppetite?: 'cautious' | 'bold';
  autonomy?: 'autonomous' | 'directed';
  conflictMode?: 'competing' | 'avoiding' | 'accommodating';
}
```

`AvatarModifiers` is the top-level input — raw adjective strings and canonical axis terms from the `AvatarPayload`. `PartModifiers` is the resolved numeric state consumed by renderers. `resolveModifiers(input: AvatarModifiers): PartModifiers` converts one to the other:
1. Each adjective is looked up in the category mapping table (`modifiers.ts`)
2. Adjectives in the same category combine: multiple "precision" adjectives reinforce, not stack
3. Each canonical axis is reduced to its discrete endpoint term (weight > 0.5 selects the strong variant)

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

**Overrides vs Tier 2 codes:** `AvatarPayload.overrides` is a payload-mode concept — it records user deltas from the archetype preset (e.g., "Sage/Detective but with aviator glasses"). Tier 2 custom codes encode the resolved absolute part selection (preset merged with overrides). Decoding a Tier 2 code produces a flat `PartAssignment`, not a base-plus-delta. The override information is consumed during encoding and does not survive roundtrip — once encoded, the code IS the identity.

### ARIA

```typescript
connectedCallback() {
  this.setAttribute('role', 'img');
}

protected willUpdate(changed: PropertyValues) {
  if (changed.has('archetype') || changed.has('code')) {
    if (!this.hasAttribute('aria-label') || this._ariaManaged) {
      const family = this.archetype?.family ?? 'Unknown';
      const sub = this.archetype?.subArchetype ?? '';
      this.setAttribute('aria-label', `${family} ${sub} agent avatar`.trim());
      this._ariaManaged = true;
    }
  }
}
```

The ARIA label updates reactively when `archetype` or `code` properties change. If the consumer provides an explicit `aria-label` attribute, the component does not override it.

## Mechanical Composition

### Part Registry

Each part category has a finite set of options. Parts are abstract semantic identifiers — the rendering is provided by the active collection's SVG symbols.

```typescript
interface PartRegistry {
  parts: Map<string, PartRenderer>;
}

type PartRenderer = (palette: FamilyPalette, modifiers: PartModifiers) => string;
```

Every renderer is a pure function: `(palette, modifiers) → SVG fragment string`. No side effects, no DOM access, fully testable.

### How Collections Populate the Registry

The PartRegistry is built FROM a collection's SVG symbol content. When a collection is loaded, each `<symbol>` element is wrapped in a renderer function:

```typescript
function createRenderer(symbolContent: string): PartRenderer {
  return (palette, modifiers) => {
    let svg = symbolContent;
    svg = applyPalette(svg, palette);       // inject CSS custom property values
    svg = applyModifiers(svg, modifiers);   // SVG transforms, attribute adjustments
    return svg;
  };
}
```

This reconciles the two layers:
- **Collections** provide base SVG geometry (static `<symbol>` elements with `var(--skin)` etc.)
- **PartRenderers** wrap that geometry, applying palette via string substitution of `var()` references and modifier effects via SVG transforms

Modifier effects that go beyond palette operate on the SVG fragment via:
- **SVG transforms** — rotation, scale, translate for expression/pose adjustments (energy, intensity)
- **Stroke/path attribute adjustments** — stroke-width, dash patterns for line quality (precision)
- **SVG filter injection** — blur, saturation for temperament effects

### Variant Selection

Collections MAY provide multiple symbol variants per part (e.g., `hair:buzz`, `hair:buzz:energy-high`). Variant selection happens in the **builder**, not the renderer — the builder knows both the modifier state and the full registry:

```typescript
function resolvePartKey(baseKey: string, modifiers: PartModifiers, registry: PartRegistry): string {
  const variantKey = deriveVariantKey(baseKey, modifiers); // e.g., "hair:buzz:energy-high"
  if (variantKey && registry.parts.has(variantKey)) return variantKey;
  return baseKey;
}
```

The builder calls `resolvePartKey` before looking up the renderer. `createRenderer` stays simple — one symbol in, one renderer out. This means adjective effects on shape (D5's "Expression tension — brow angle, mouth set" for Intensity, "Pose dynamism" for Energy) are achieved through either SVG transforms on the base symbol (always available) or variant symbols in the collection (richer, but requires collection author to provide them). Both mechanisms compose: a variant symbol still receives modifier transforms.

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
  const partMods = resolveModifiers(modifiers);
  const render = (baseKey: string) => {
    const key = resolvePartKey(baseKey, partMods, registry);
    return registry.parts.get(key)!(palette, partMods);
  };

  // Layer 1-3: always rendered (all sizes)
  layers.push(render(`costume:${config.costume}`));
  layers.push(render(`head:${config.head}`));
  layers.push(render(`hair:${config.hair}`));

  // Layer 4: hat replaces top of hair silhouette (sm+)
  if (detail >= DetailLevel.SM && config.hat) {
    layers.push(render(`hat:${config.hat}`));
  }

  if (detail >= DetailLevel.MD) {
    // Layer 5: facial hair
    if (config.facialHair !== 'none') {
      layers.push(render(`beard:${config.facialHair}`));
    }
    // Layer 6: expression overlay (mouth + eye mood)
    if (config.expression) {
      layers.push(render(`expression:${config.expression}`));
    }
    // Layer 7: eyebrows
    layers.push(render(`brow:${config.eyebrows}`));
    // Layer 8: glasses
    if (config.glasses) {
      layers.push(render(`glasses:${config.glasses}`));
    }
    // Layer 9: props
    for (const prop of config.props) {
      layers.push(render(`prop:${prop}`));
    }
  }

  if (detail >= DetailLevel.LG) {
    // Layer 10: accessories
    for (const acc of config.accessories) {
      layers.push(render(`acc:${acc}`));
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

### Adjective-to-Category Mapping Principle

Every adjective in `ArchetypeTerm.java` maps to exactly one of the 5 effect categories. The mapping follows from the adjective's semantic domain:

| Category | Mapping Rule | Examples from ArchetypeTerm.java |
|----------|-------------|--------------------------------|
| Intensity | Adjectives describing force, restraint, or emotional magnitude | fierce, gentle, bold, subtle, intense, soft, firm, passionate |
| Temperament | Adjectives describing warmth, coolness, or emotional tone | warm, serene, measured, radiant, compassionate, cool |
| Energy | Adjectives describing motion, pace, or dynamism | restless, driven, energetic, contemplative, spontaneous, calm, relentless |
| Precision | Adjectives describing exactness, method, or rigour | meticulous, precise, methodical, systematic, rigorous, analytical, disciplined |
| Organic | Adjectives describing naturalness, intuition, or fluidity | intuitive, flowing, natural, holistic, empathic, perceptive, mystical |

Adjectives that don't clearly fit one category (e.g., "resourceful", "strategic") map to the category most relevant to their visual expression — typically Precision for cognitive/structural adjectives and Energy for action/behavior adjectives.

`ArchetypeTerm.java` is the single source of truth for adjective lists. The full adjective → category mapping table is implementation data generated from ArchetypeTerm.java and codified in `modifiers.ts`. The avatar-generator-contract's adjective tables in Section 4 diverge from ArchetypeTerm.java in several places (e.g., Angel has 8 adjectives in the contract vs 7 in Java, Clown uses "absurd/silly/self-deprecating" in the contract vs "absurdist/surprising/expressive" in Java) — where they conflict, ArchetypeTerm.java prevails.

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
| `mythic.parts.svg` | Construction — all parts for mechanical composition | ~250+ `<symbol>` elements (heads, hairs, hats, expressions, costumes, props, etc.) |
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

Parts use CSS custom property references as palette placeholders:
- `var(--skin)` — skin tone
- `var(--primary)` — family primary colour
- `var(--secondary)` — family secondary colour
- `var(--accent)` — family accent colour
- `var(--hair-color)` — hair/beard colour

`applyPalette()` performs string substitution: each `var(--skin)` reference in the SVG fragment is replaced with the concrete colour value from the `FamilyPalette` (e.g., `var(--skin)` → `#f5d6c3`). This is required because `renderAvatar()` returns a standalone SVG string that must render without a DOM context — CSS custom property cascade is not available. The `<agent-avatar>` component uses the same string substitution path for consistency.

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

**Note on rendering:** The `<use href>` examples above are illustrative of the ID convention. In practice, preview SVG content is inlined into the DOM rather than referenced via external `<use href>` — cross-origin `<use>` with external SVG files is unreliable across browsers (CORS restrictions, shadow DOM limitations). The theme browser loads the preview file via fetch, extracts symbols, and renders them inline — the same approach as the parts file.

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

### SVG Sanitization

Runtime-loaded collections are sanitized before storage in the parts map. The sanitizer strips:
- `<script>` elements
- Event handler attributes (`onload`, `onclick`, `onerror`, etc.)
- `<foreignObject>` elements
- `<use>` elements with external `href` (cross-origin references)
- `javascript:` URI schemes in any attribute

Built-in collections (inlined at build time) are reviewed code and skip runtime sanitization.

### Collection Fallback Behavior

| Scenario | Behavior |
|----------|----------|
| Collection loading (async fetch in progress) | Render the Everyman/Citizen silhouette in desaturated palette as placeholder. Re-render when collection loads. |
| Collection fetch fails (network error, 404) | Log warning. Fall back to `mythic` collection (always available, inlined at build time). |
| Loaded collection is missing a part | Use the `mythic` collection's renderer for the missing part. Log a warning identifying the gap. |
| Unknown collection ID | Fall back to `mythic` collection. Log warning. |

The `mythic` collection is always the ultimate fallback — it is inlined at build time and never requires network access.

### mythic — First Collection

The inaugural collection. Flat illustration style with half-body framing (head to waist).

Visual language:
- Distinct head shapes per family (round, square jaw, diamond, heart, oval, angular, etc.)
- Family colour palettes (cool blues for Sage, deep purples for Magician, bold primaries for Hero)
- Archetype-specific props (magnifying glass, orb, paintbrush, shield, crown, compass, etc.)
- Character variety: 55 hair styles, 40 hats, 35 expressions, 27 facial hair, 43 glasses, 32 accessories, 8 eyebrow types
- Size-responsive detail tiers (xs silhouette → lg full detail)

Layer z-order (bottom to top):
1. `costume:` — torso, y=130-240
2. `head:` — face shape, centred y=90
3. `hair:` — hair on/around head
4. `hat:` — on top of hair (sm+)
5. `beard:` — facial hair overlay (md+)
6. `expression:` — mouth/eye mood overlay (md+)
7. `brow:` — eyebrows (md+)
8. `glasses:` — eyewear (md+)
9. `prop:` — held/adjacent objects (md+)
10. `acc:` — piercings, scarves, etc. (lg)

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
- 10 base64 chars encoding 60 bits of part selections:
  - version (2b) + head (4b) + hair (6b) + hat (6b) + facialHair (5b)
  - expression (6b) + costume (5b) + glasses (4b) + eyebrows (4b)
  - accessory (5b) + palette (4b) + prop1 (6b) + prop2 (6b) + spare (1b)

60 bits = 10 base64 chars. The expanded part registry (55 hairs, 40 hats, 35 expressions, 43 glasses, 27 beards, 32 accessories) requires wider bit fields than the original 48-bit encoding. Version 0 is the initial encoding.

### Scope of Compact Codes

Compact codes encode **part identity** — which parts and which palette. They do not encode adjective or canonical axis modifiers. `renderAvatar(code)` produces the base preset or custom visual without modifier effects. This is intentional: codes are identity tokens for persistence and sharing; modifier effects come from the full `AvatarPayload` at render time.

### Properties

- **Deterministic:** same code → same SVG (base visual without modifiers), always
- **Decodable:** code → full part list without database lookup
- **Compact:** 6-15 characters total
- **Versioned:** 2-bit version field enables encoding evolution without breaking existing codes
- **Append-only:** new parts get new indices; existing indices are stable across versions

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
function renderAvatar(code: string, options?: {
  size?: AvatarSize;           // default: 'md'
  modifiers?: AvatarModifiers;
}): string
```

Give it a compact code (e.g., `mythic:P1B`), get back a complete SVG string. Internally: decode → resolve preset + overrides → look up parts from collection → apply palette via string substitution → resolve modifiers (if provided) → resolve part variants → assemble layers at the requested detail tier → return SVG string.

`size` controls the detail tier (xs silhouette-only through lg full detail) — without it, renders at `md`. Without `modifiers`, the output is the base visual — part selections and palette only. With `modifiers`, adjective and canonical axis effects are applied. This separation is intentional: codes identify WHICH parts; modifiers express HOW those parts render.

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

Export `AgentAvatarProps` from `index.ts` for type safety. Register `'agent-avatar': AgentAvatarProps` in `BlocksComponentRegistry` in `packages/blocks-ui-schema/src/registry.ts` — following the precedent of `commitment-range-bar` and `commitment-transition-badge` which are also registered without the `blocks-*` prefix. The `agent-avatar` element is not renamed to `blocks-agent-avatar`: the protocol PP-20260907-fd8ee7 explicitly scopes to "Any new @customElement('blocks-*') component" — `agent-avatar` predates this protocol and is a domain-specific visualization, not a generic UI primitive.

## DiceBear Removal

Remove `@dicebear/core` and `@dicebear/avataaars` from package.json. The old `generateAvatar()`, `generateCandidates()`, `customiseAvatar()` functions are replaced by the builder. The old `canonical-registry.ts` AXIS_MAPPINGS are superseded by the canonical axis expression modifiers.

### Fallback Behaviour

When archetype data is missing (no `archetype` property, no `code`), render the Everyman/Citizen preset with a desaturated greyscale palette. Visually distinct from any assigned archetype — signals "identity not resolved" without breaking layout.

## Testing Strategy

| Layer | Test Approach |
|-------|--------------|
| Part renderers | Each renderer returns valid SVG (well-formed XML, non-empty) for all 12 palettes |
| Config table | 48 entries, every archetype mapped, uniqueness audit (≥2 parts different within family). Expression and hat fields populated for all entries. |
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

### Wire Format

`FullAgentDescriptor` in `blocks-ui-core` needs new fields to carry archetype data to the frontend:

```typescript
interface FullAgentDescriptor {
  // ... existing fields ...
  archetypeFamily?: string;      // e.g., "Sage"
  subArchetype?: string;         // e.g., "Detective"
  archetypeAdjectives?: string[];// e.g., ["meticulous", "persistent"]
  avatar?: string;               // compact code, e.g., "mythic:P1B"
}
```

The component supports two data paths:
1. **Full payload** — caller constructs `AvatarPayload` from `archetypeFamily`, `subArchetype`, `archetypeAdjectives`, and optionally `disposition` (for canonical axes)
2. **Compact code** — caller passes `avatar` string directly as `code` property

For the common case (no user customisation), only `avatar` is needed. Full payload enables modifier effects.

## Cross-Repo Concerns

| Concern | Repo | Action |
|---------|------|--------|
| `avatar` field in agent YAML schema | eidos | Add optional `avatar: string` field to agent definition |
| ArchetypeResolver → avatar code | eidos | Generate `mythic:P<index>` default code when archetype resolves |
| `FullAgentDescriptor` type changes | blocks-ui-core | Add `archetypeFamily`, `subArchetype`, `archetypeAdjectives`, `avatar` fields |
| API response augmentation | eidos | Include archetype fields in agent descriptor API responses |
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
