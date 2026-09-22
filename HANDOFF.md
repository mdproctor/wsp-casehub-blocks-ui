# HANDOFF — casehub-blocks-ui

## Current Session

**Branch:** `issue-167-archetype-avatar-system` (active, not closed)
**Issue:** casehubio/blocks-ui#167 — Archetype-driven avatar system
**Queue:** 0/1 — design phase complete, implementation not started

### What Was Done

**Design session — archetype avatar system from spec through collection prototyping:**

1. **Brainstorming + 15 design decisions** — scope (SVG templates only), composable parts, archetype payload API, mechanical composition, deterministic identity codes, collection theming, compact codes, YAML integration
2. **Design spec written + reviewed** — standard review (21 issues, 15 verified, 3 accepted, 3 deferred). Spec revised with: expanded type interfaces (AvatarModifiers, PartModifiers, AxisExpression), reactive ARIA, collection loading/sanitization/fallback
3. **Implementation plan written** — 5 batches, 8 tasks, injected into .plan queue
4. **Part catalogue** — all 48 sub-archetype assignments with combined props, visual hooks, colour palettes, head shapes

**Collection prototyping (5 collections designed/generated):**

| Collection | Style Tag | Symbols | Source | Status |
|---|---|---|---|---|
| mythic | `flat-illustration` | 12 roots + catalogue | Hand-built (this session) | 12 roots done, 48 sub-archetypes in catalogue only |
| bauhaus | `geometric-polygon` | 177 | Gemini | Complete |
| donut-creek | `overbite` | 209 + 27 extras | Gemini | Complete, cast preview with 12 Springfield residents |
| donut-creek-homer | `overbite` | 209 | Gemini (same base) | Complete |
| neon | `wireframe-glow` | 354 | Gemini (two sessions) | Complete, includes expanded vocab |
| blueprint | `wireframe-clean` | 189 | Gemini | Complete — palette-neutral parts usable as baseline |

**Expanded part categories added to spec:**
- `expression:` — face mood overlays (mouth + eye variants), 35 parts across collections
- `hat:` — separated from accessories, own z-order layer (sm+), 40 parts
- Updated builder z-order: costume → head → hair → hat → beard → expression → brow → glasses → props → accessories

**Collection metadata system:**
- Style tags (`overbite`, `geometric-polygon`, `wireframe-glow`, `wireframe-clean`, `flat-illustration`) declare composability
- Parts with same style compose naturally; cross-style mixing produces warnings
- `CollectionManifest` interface with `styleProperties` (strokeWeight, fillMode, outline, glow)

**Cross-repo:**
- casehubio/eidos#181 — avatar field in agent YAML (created)
- casehubio/blocks-ui#168 — v2 composable hand gestures (created, with exploration + spatial analysis)

### Key Files (workspace)

| Path | What |
|------|------|
| `specs/issue-167-archetype-avatar-system/2026-09-21-archetype-avatar-system-design.md` | Design spec (reviewed) |
| `specs/issue-167-archetype-avatar-system/decisions.md` | 15 design decisions |
| `specs/issue-167-archetype-avatar-system/part-catalogue.md` | Full 48-archetype part assignments |
| `specs/issue-167-archetype-avatar-system/avatar-preview.html` | Mythic roots visual preview |
| `specs/issue-167-archetype-avatar-system/bauhaus.parts.svg` | Bauhaus collection (177 symbols) |
| `specs/issue-167-archetype-avatar-system/donut-creek.parts.svg` | Donut Creek base (209 symbols) |
| `specs/issue-167-archetype-avatar-system/donut-creek-extras.parts.svg` | Donut Creek character modifiers (27) |
| `specs/issue-167-archetype-avatar-system/donut-creek-cast.preview.svg` | Cast preview (12 Springfield residents) |
| `specs/issue-167-archetype-avatar-system/neon.parts.svg` | Neon collection (354 symbols) |
| `specs/issue-167-archetype-avatar-system/shared-expanded-parts.svg` | Blueprint style expanded vocab (123) |
| `specs/issue-167-archetype-avatar-system/shared-expressions-hats.svg` | Blueprint style expressions + hats (66) |
| `specs/issue-167-archetype-avatar-system/hands-exploration.html` | Hands prototype (reference for #168) |
| `specs/issue-167-archetype-avatar-system/hands-reference-issue-168.svg` | Hand gesture SVGs (20) |
| `specs/issue-167-archetype-avatar-system/grok-geometric.parts.svg` | Grok's bauhaus alternative (reference) |
| `specs/issue-167-archetype-avatar-system/expanded-vocab-preview.html` | Visual catalogue of all expanded parts |
| `plans/2026-09-21-archetype-avatar-system.md` | Implementation plan (5 batches, 8 tasks) |

### What's Left To Do

**Before implementation (design completion):**

1. **Remap expanded vocab IDs** — `shared-expanded-parts.svg` and `shared-expressions-hats.svg` use non-standard IDs (`facial-hair:` → `beard:`, `accessory:` → `acc:`, creative names → canonical). Build a mapping table and remap.

2. **Update part catalogue** — add `expression` and `hat` columns to all 48 archetype assignment rows. Decide which expression and hat each sub-archetype gets.

3. **Update implementation plan** — plan was written before expressions/hats were added. Needs new tasks for the expanded categories and updated config table structure.

4. **Update compact code encoding** — plan references 48-bit encoding, spec now says 60-bit (10 base64 chars). Plan's Task 3 (code.ts) needs updating.

**Collection gaps:**

5. **bauhaus** — needs expression and hat symbols in polygon style (currently has neither). 177 → ~250+.

6. **donut-creek** — needs expression and hat symbols in overbite style. Has face extras already but they need formalising as expressions.

7. **mythic** — needs full parts.svg built from the avatar-preview.html roots + catalogue. Currently only has 12 roots as HTML, not a proper SVG parts file.

8. **neon** — already has expressions, faces, and hats from the expanded merge. May need ID remapping for consistency.

**Future collections:**

9. **scummbar** — LucasArts adventure game pixel art style. Inspired by: Monkey Island, Day of the Tentacle, Maniac Mansion, Discworld, Sam & Max, Full Throttle, Grim Fandango, Beneath a Steel Sky, Broken Sword. Higher res than original (32×48 grid upscaled to 200×240). Hard pixel edges, limited palette per family (8-12 colours), dithering for gradients. The finale collection.

**Implementation (when design is complete):**

10. **Execute implementation plan** — Batch 1: types + config + encoding. Batch 2: builder + collection registry. Batch 3: mythic SVG parts + extract script. Batch 4: component + DiceBear removal. Batch 5: FullAgentDescriptor wiring.

### Compact Code System (for reference)

Avatar identity: `{collection}:{P|C}{encoded}` — e.g., `mythic:P1B` (preset Detective) or `neon:Co/grLNBF` (customised). Deterministic: same code → same SVG. Stored in eidos agent YAML `avatar` field.

### Collection Architecture (for reference)

A collection = 2 SVG files (`{name}.parts.svg` + `{name}.preview.svg`). Parts file contains `<symbol>` elements with `{category}:{part-id}` IDs. Parts use `var(--skin)`, `var(--primary)`, `var(--secondary)`, `var(--accent)`, `var(--hair-color)` for palette-driven colours. Builder applies palette via string substitution, not CSS cascade. Collections with same `style` tag compose freely; cross-style mixing is warned.
