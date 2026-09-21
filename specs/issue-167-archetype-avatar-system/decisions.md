# Decisions — #167 Archetype Avatar System

## D1: Scope — SVG templates only

**Choice:** Build the 60-template SVG system + updated `<agent-avatar>` component. Faceted personality selector is a separate issue.
**Alternatives:**
- Both subsystems — delivers complete pipeline but doubles scope
- Templates + basic wiring — middle ground, but the faceted selector UI deserves its own design cycle
**Rationale:** The faceted selector is a complex interactive UI with its own interaction model, conflict detection, and state management. Separating it allows focused delivery of the visual system.
**Trade-offs:** Callers must provide pre-resolved archetype data until the selector is built.
**Sources:** eidos avatar-generator-contract.md (Sections 1-5 vs Section 3)
**Exploration:** quick
**Status:** captured

## D2: Visual fidelity — composable SVG parts

**Choice:** Build a template system from composable SVG parts (head shapes, hair, props, costumes, palettes) that combine programmatically.
**Alternatives:**
- 60 hand-drawn SVGs — highest quality but enormous art effort, hard to maintain
- Parameterised single template — simpler but insufficient visual distinction
- 12 family templates + parametric variants — middle ground, but limits sub-archetype differentiation
**Rationale:** Composable parts give maximum flexibility with manageable complexity. New sub-archetypes added by registering part combinations. Each part independently testable.
**Trade-offs:** Requires designing a coherent part system where pieces compose well together.
**Sources:** Issue #167 design references, previous session prototyping insights
**Exploration:** quick
**Status:** captured

## D3: Component API — archetype payload only

**Choice:** Accept `{ family, subArchetype, adjectives?, canonicalAxes? }`. Clean break from DiceBear. Old disposition-only mode removed.
**Alternatives:**
- Dual-mode with fallback — backward compatible but adds complexity
- Extend AgentDisposition — muddies personality axes vs archetype identity
**Rationale:** Archetype identity is a fundamentally different data model from disposition axes. Clean API avoids confusion. The faceted selector (future) handles convergence from disposition to archetype.
**Trade-offs:** Breaking change for existing callers (agent-catalog, agent-profile, agent-wizard). All must provide archetype data.
**Sources:** eidos avatar-generator-contract.md Section 5 (input schema)
**Exploration:** quick
**Status:** captured

## D4: SVG composition — layered builder with part registries

**Choice:** Layered composition with part registries. Avatar is an SVG canvas with stacked layers (body → costume → head → hair → face → facial hair → props → accessories). Each part is a pure function: `(palette, modifiers) → SVG path string`.
**Alternatives:**
- Monolithic templates with parametric fills — faster to build but harder to maintain, testing individual parts is harder, adjective modifiers need DOM manipulation
**Rationale:** Full adjective system and 60 sub-archetypes make composability essential. Monolithic templates become unmanageable at this scale. Part registry aligns with how the spec organises visual elements.
**Trade-offs:** More upfront work to design the part system and ensure visual coherence across combinations.
**Sources:** eidos avatar-generator-contract.md Section 5 (visual template guidelines)
**Exploration:** quick
**Status:** captured

## D5: Adjective system — full implementation

**Choice:** Implement per-sub-archetype valid/invalid adjective lists and all 5 effect categories (intensity, temperament, energy, precision, organic).
**Alternatives:**
- Include basic modifiers only — forward-compatible but limited
- Defer adjectives entirely — simpler scope but incomplete payload support
**Rationale:** The adjective catalog is fully defined in the eidos contract. Implementing it now means the avatar system is complete when the faceted selector lands.
**Trade-offs:** Significant scope — need to define visual effects for 5 adjective categories across all 60 sub-archetypes.
**Sources:** eidos avatar-generator-contract.md Section 4 (adjective catalog + visual effects)
**Exploration:** quick
**Status:** captured

## D6: Canonical axes — tertiary expression modifiers

**Choice:** Map the 5 canonical axes to subtle SVG expression adjustments (brow angle, mouth curve, eye openness). Tertiary layer — archetype template dominates.
**Alternatives:**
- Defer — archetype + adjectives enough visual variation
**Rationale:** The axes are the same 5 the current DiceBear system uses, so the mapping concept is proven. They add personality nuance without conflicting with the archetype template.
**Trade-offs:** Additional complexity in the face layer rendering.
**Sources:** eidos avatar-generator-contract.md Section 5 (canonical axes → expression geometry)
**Exploration:** quick
**Status:** captured

## D7: DiceBear removal

**Choice:** Remove @dicebear/core and @dicebear/avataaars entirely from agent-avatar-2d.
**Alternatives:**
- Keep as fallback for missing archetype data
**Rationale:** The inline SVG builder replaces DiceBear completely. Keeping it adds bundle size and maintenance burden for no benefit since the API is a clean break anyway (D3).
**Trade-offs:** No fallback if archetype data is missing — component must handle missing data gracefully itself.
**Depends on:** D3 (archetype payload only API)
**Sources:** packages/agent-avatar-2d/dist/generator.js (current DiceBear usage)
**Exploration:** quick
**Status:** captured

## D8: Visual design constraints (from prototyping)

**Choice:** Apply these constraints from previous session prototyping:
1. Half-body framing (head to waist — not full legs, not just bust)
2. Distinct head shapes per family (round for Innocent/Caregiver, square jaw for Hero, diamond for Magician, heart for Lover, etc.)
3. Archetype-specific props (magnifying glass, orb/wand, paintbrush/palette, shield, crown/scepter, compass/hat, juggling balls, gavel, etc.)
4. Character variety — diverse facial hair (Einstein wild, manicured moustache, bushy beard), diverse hair styles
5. Family colour palettes from spec (cool blues for Sage, deep purples for Magician, bold primaries for Hero, etc.)
**Alternatives:** None — these are validated design constraints from iterative prototyping
**Rationale:** Previous session validated these through v1-v4 prototype iterations. They produce visually distinctive, recognisable archetypes.
**Trade-offs:** Head shape variety increases the number of base body/head SVG parts needed.
**Sources:** eidos avatar-generator-contract.md Section 5 (visual template guidelines, colour tendencies), previous session prototyping
**Exploration:** quick
**Status:** captured
