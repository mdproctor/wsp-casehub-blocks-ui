# Agent Setup Wizard — Design Decisions

## D1: Component architecture

**Choice:** Independent standalone components, composition shell deferred
**Alternatives:**
- Single multi-step wizard — tighter guided flow but less reusable individually
- Independent components with shell — premature until pieces exist and natural flow emerges
**Rationale:** Build each component independently first. The composition question answers itself once the pieces exist. Matches blocks-ui pattern where components are standalone web components.
**Trade-offs:** No guided end-to-end flow out of the box initially
**Sources:** blocks-ui CLAUDE.md (design philosophy), existing component patterns (split-workbench, detail-pane)
**Exploration:** quick
**Status:** captured

## D2: Manifest editor — preset model

**Choice:** Presets with drill-down — complete manifest templates with expandable sections for granular editing
**Alternatives:**
- Complete manifest templates only — fastest but least flexible
- Granular step-by-step wizard — most control but slowest to set up
**Rationale:** Presets get users 80% there, drill-down for the rest. Main providers (Anthropic, OpenAI, Google, local/Ollama), main models, three credential backends (env var, direct encrypted, vault URI).
**Trade-offs:** More UI surface than pure presets
**Sources:** platform ManifestLoader (layered loading with merge-by-priority), platform#291 (wizard API)
**Exploration:** quick
**Status:** captured

## D3: Manifest editor — guidance and detection model

**Choice:** Progressive disclosure with live detection
**Alternatives:**
- Wizard-style step flow ("What do you want to do?" → template → configure) — more guided but imposes use-case framing that doesn't map to manifest data model
**Rationale:** The manifest is fundamentally a provider+model+credential structure. Auto-detection (sniff env vars, detect Ollama) with provider cards showing detected state, inline help, and "test connection" validation. Makes setup non-daunting without imposing artificial flow.
**Trade-offs:** Detection logic needs per-provider implementation
**Sources:** ManifestCredentialResolver (env:VAR_NAME pattern), platform agent-config-core
**Exploration:** quick
**Status:** captured

## D4: Personality catalog — scope and browsing

**Choice:** Comprehensive catalog with pill-based filtering, popular section, from-scratch wizard
**Alternatives:**
- Narrow (~5 presets) — validates UI but not useful
- Core domains (~10-15) — moderate breadth
**Rationale:** Dense browsing with pill filters (domain, task type, personality traits). "Popular/Common" section at top. Each preset refineable per characteristic. From-scratch guided wizard for custom personalities.
**Trade-offs:** Large upfront content investment for catalog
**Sources:** eidos AgentDescriptor schema, customer-support-agent.yaml profile, Belbin/DISC frameworks
**Exploration:** quick
**Status:** captured

## D5: Avatar generator

**Choice:** Deterministic default from personality traits, browse all valid candidates, customise individual features
**Alternatives:**
- Random generation with user pick — fun but unpredictable
- Fully manual — no personality connection
**Rationale:** Personality axes map to visual features (autonomy→posture, socialOrient→expression, riskAppetite→accessories, ruleFollowing→clothing formality, conflictMode→eyebrow/mouth). Default to first valid match, user can browse all valid candidates for that profile, then customise features. Libraries: DiceBear, Avataaars, Open Peeps.
**Trade-offs:** Needs curated trait→visual mapping per art style
**Sources:** DispositionAxes type (autonomy, ruleFollowing, socialOrient, riskAppetite, conflictMode)
**Exploration:** quick
**Status:** captured

## D6: Memory seeding — scope

**Choice:** Placeholder section with starter functionality, full seeding API as follow-up
**Alternatives:**
- Full seeding UI now — too much without unified API
- Skip entirely — loses the end-to-end feel
**Rationale:** Follow Wacky Manor pattern. Starter: configure beliefs (text + confidence), drives (personality-derived defaults, tweak intensity), inter-agent relationships (PAD or presets). Briefing keeps only voice/identity/mannerisms. Full CognitiveSeeder orchestration API is a neocortex follow-up.
**Trade-offs:** Memory seeding won't actually work end-to-end until neocortex provides unified API
**Sources:** ManorCognitiveSeeder.java, CharacterCognition.java, social-config.yaml (slot 196)
**Exploration:** quick
**Status:** captured

## D7: Headless/payload data mode

**Choice:** All components support inline data mode — accept payloads for full UI simulation
**Alternatives:** None considered — this is a hard requirement and matches existing manifest design
**Rationale:** Manifest already designed for layered/payload-driven loading. All components follow blocks-ui dual data mode pattern (endpoint or inline property). "Just works" when backend available, fully simulatable without it.
**Trade-offs:** None — this is the standard blocks-ui pattern
**Sources:** ManifestLoader (layered loading), blocks-ui dual data mode pattern across all components
**Exploration:** quick
**Status:** captured

## D8: Relationship management approach

**Choice:** Table-based editing + multiple experimental visualization views
**Alternatives:**
- Layered ego star + table — more spatial but needs ~400px, still has crossing risk
- Neo4j-style card panel + mini force graph — spaghetti risk at 15+ nodes
- Dual-column matrix — zero spaghetti but no spatial intuition
- Arc diagram strip + table — low spaghetti but not zero, adds complexity
**Rationale:** Table is the primary editing interface (grouped by relationship kind, add/remove via forms). Multiple "views" for visualization: table view, arc diagram view, ego-centric org diagram view. The ego org diagram view reuses the existing graph-stencil-org renderer + ELK layout as an experiment — filter the org YAML to the selected agent's 1-hop neighbourhood and render in read-only mode. If ELK handles the star topology cleanly, we get ego-centric visualization for free. If not, a custom renderer follows later.
**Trade-offs:** Multiple views means more code surface, but avoids premature commitment to a single visualization approach. Ego org diagram reuse is an experiment — may need custom renderer.
**Sources:** graph-stencil-org (existing renderer), ELK layout engine, 2024 ScienceDirect ego network study (layered node-link preferred), arc diagram pattern (data-to-viz.com)
**Exploration:** deep-analysis
**Status:** captured
