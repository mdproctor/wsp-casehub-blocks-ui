# Agent Setup Wizard — Design Decisions

## D1: Component architecture

**Choice:** Independent standalone components, composition shell deferred
**Alternatives:**
- Single multi-step wizard — tighter guided flow but less reusable individually
- Independent components with shell — premature until pieces exist and natural flow emerges
**Rationale:** Build each component independently first. The composition question answers itself once the pieces exist. Matches blocks-ui pattern where components are standalone web components. Note: these components are being designed directly in blocks-ui, bypassing the normal promotion pipeline (domain app → blocks-ui on second consumer). This is intentional — they are platform-wide agent setup tools with no single domain home.
**Trade-offs:** No guided end-to-end flow out of the box initially. Skipping incubation means less iterative domain-specific refinement.
**Sources:** blocks-ui CLAUDE.md (design philosophy), existing component patterns (split-workbench, detail-pane), ARC42STORIES.MD §9 (promotion pipeline — deliberately bypassed here)
**Exploration:** quick
**Status:** captured

## D2: Manifest editor — preset model

**Choice:** Presets with drill-down — complete manifest templates with expandable sections for granular editing
**Alternatives:**
- Complete manifest templates only — fastest but least flexible
- Granular step-by-step wizard — most control but slowest to set up
**Rationale:** Presets get users 80% there, drill-down for the rest. Main providers (Anthropic, OpenAI, Google, local/Ollama), main models. Three credential backends matching `CredentialRef` sealed interface: `env:VAR_NAME` (environment variable), `file:/path` (file path), `ref:name` (external credential store via pluggable CredentialResolver SPI). Must consume the actual wizard API contract from platform#291: `GET /llm/providers`, `POST /llm/configure`, `GET /llm/configured`.
**Trade-offs:** More UI surface than pure presets
**Sources:** platform ManifestLoader (layered loading with merge-by-priority), platform#291 wizard API (GET /llm/providers, POST /llm/configure, GET /llm/configured), CredentialRef sealed interface (EnvRef, FileRef, ExternalRef)
**Exploration:** quick
**Status:** captured — revised after review (R1-03, R1-08)

## D3: Manifest editor — guidance and detection model

**Choice:** Progressive disclosure with live detection
**Alternatives:**
- Wizard-style step flow ("What do you want to do?" → template → configure) — more guided but imposes use-case framing that doesn't map to manifest data model
**Rationale:** The manifest is fundamentally a provider+model+credential structure. Auto-detection with provider cards showing detected state, inline help, and "test connection" validation. Detection API surface: either `GET /llm/providers` from platform#291 returns availability/detected state, or a dedicated `GET /llm/detect` endpoint is needed. The frontend cannot detect Ollama without a backend proxy. ManifestCredentialResolver only resolves known references — it doesn't discover providers.
**Trade-offs:** Detection logic needs backend API support (cannot be frontend-only). Per-provider detection implementation.
**Sources:** ManifestCredentialResolver (env:VAR_NAME pattern), platform agent-config-core, platform#291 wizard API
**Exploration:** quick
**Status:** captured — revised after review (R1-09)

## D4: Agent template catalog — scope and browsing

**Choice:** Comprehensive agent template catalog with pill-based filtering, popular section, from-scratch wizard
**Alternatives:**
- Narrow (~5 presets) — validates UI but not useful
- Core domains (~10-15) — moderate breadth
**Rationale:** This is an agent template catalog, not just a personality catalog. Each template produces a valid `AgentDescriptor` covering: identity (agentId, name, slot), capabilities (with quality/latency/cost hints, epistemic domains), disposition (5 axes: SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY, CONFLICT_MODE with vocabulary-controlled DispositionValue terms), goals, constraints, and briefing. Dense browsing with pill filters (domain, task type, disposition traits). "Popular/Common" section at top. Each field refineable. From-scratch guided wizard for custom agents.
**Trade-offs:** Large upfront content investment for catalog. Must align with AgentDisposition vocabulary system (dispositionVocabulary, axisVocabularies).
**Sources:** eidos AgentDescriptor (23+ fields), AgentDisposition (5-axis model with DispositionValue lists), customer-support-agent.yaml profile. Note: eval profiles reference Belbin/DISC as theoretical frameworks for test data derivation, but the runtime model is the 5-axis AgentDisposition — the catalog produces AgentDisposition instances, not Belbin/DISC mappings.
**Exploration:** quick
**Status:** captured — revised after review (R1-04, R1-06). Renamed from "Personality catalog" to "Agent template catalog".

## D5: Avatar generator

**Choice:** 2D profile avatars complementing the existing 3D TalkingHead system. Deterministic default from disposition axes, browse valid candidates, customise features.
**Alternatives:**
- Random generation with user pick — fun but unpredictable
- Fully manual — no personality connection
- Replace 3D with 2D — would abandon lip-sync, WebSocket audio, viseme morphing
**Rationale:** blocks-ui already ships a 3D avatar system in `packages/avatar/` (TalkingHead, GLB models, lip-sync, WebSocket audio streaming, conversation turn protocol). The 2D avatar generator is complementary — profile icons, thumbnails, catalog cards, org diagram nodes. The 3D system handles live interaction. Boundary: 2D for identity representation (static), 3D for conversation (interactive).

Disposition axes map to visual features. Because DispositionValue terms are vocabulary-controlled strings, the mapping needs a canonical term registry: a fixed set of known terms per axis that map to visual features, with a fallback for unknown vocabulary terms (e.g., closest known term or neutral default). Libraries: DiceBear Avataaars (richest combinatorial space, MIT, SVG, client-side, deterministic).
**Trade-offs:** Needs curated canonical-term→visual mapping. Custom vocabularies fall back to defaults until extended.
**Sources:** packages/avatar/ (existing 3D system), DispositionAxes type, AgentDisposition vocabulary system
**Exploration:** quick
**Status:** captured — revised after review (R1-02, R1-11)

## D6: Memory seeding — scope

**Choice:** Placeholder section with starter functionality, full seeding API as follow-up
**Alternatives:**
- Full seeding UI now — too much without unified API
- Skip entirely — loses the end-to-end feel
**Rationale:** Follow the pattern established in the Wacky Manor example (slot 196, outside the main dependency chain): briefing keeps voice/identity, discoverable memory gets beliefs, drives, needs, relationships. The ManorCognitiveSeeder is a reference implementation showing the target architecture — seeding beliefs as MindMap nodes with confidence scores, drives with intensity, relationships with PAD model.

Starter UI: configure beliefs (text + confidence), drives (personality-derived defaults from CognitiveDerivationEngine, tweak intensity), inter-agent relationships (PAD or presets). When no backend seeding API is available, the component renders the configuration and emits the structured payload — the host can persist it or display it. The UI is useful as a design/preview tool even before the seeding API exists.
**Trade-offs:** Memory seeding won't actually seed until neocortex provides unified API. Component is useful for configuration preview and payload generation in the interim.
**Sources:** Wacky Manor ManorCognitiveSeeder (reference implementation, slot 196), CognitiveDerivationEngine (personality→cognitive defaults bridge), CharacterCognition (runtime memory-first belief retrieval)
**Exploration:** quick
**Status:** captured — revised after review (R1-05)

## C1: Headless/payload data mode (constraint)

All components support inline data mode — accept payloads for full UI simulation. This is the standard blocks-ui dual data mode pattern (endpoint or inline property), not a design choice. Manifests are already designed for layered/payload-driven loading. "Just works" when backend available, fully simulatable without it.

**Sources:** ManifestLoader (layered loading), blocks-ui dual data mode pattern

## C2: Platform-tier type dependencies (constraint)

These components depend on eidos API types (AgentDescriptor, AgentDisposition, DispositionValue) and platform agent-config-core types (Manifest, CredentialRef, ProviderDeclaration). This is a new dependency category for blocks-ui — existing components depend only on casehub-pages. TypeScript type mirrors must be maintained, following the pattern established by graph-stencil-org (which already mirrors DispositionAxes, AgentDescriptor as TS interfaces in types.ts). Extend blocks-ui-core with the additional type mirrors needed.

**Sources:** graph-stencil-org/src/types.ts (existing TS mirror pattern), ARC42STORIES.MD §2 (dependency rules)

## D8: Relationship management approach

**Choice:** Table-based editing + multiple experimental visualization views
**Alternatives:**
- Layered ego star + table — more spatial but needs ~400px, still has crossing risk
- Neo4j-style card panel + mini force graph — spaghetti risk at 15+ nodes
- Dual-column matrix — zero spaghetti but no spatial intuition
- Arc diagram strip + table — low spaghetti but not zero, adds complexity
**Rationale:** Table is the primary editing interface (grouped by relationship kind, add/remove via forms). Multiple "views" for visualization: table view, arc diagram view, ego-centric org diagram view. The ego org diagram view reuses the existing graph-stencil-org renderer + ELK layout as an experiment — filter the org YAML to the selected agent's 1-hop neighbourhood and render in read-only mode. If ELK handles the star topology cleanly, we get ego-centric visualization for free. If not, a custom renderer follows later. Note: the toOrgGraph adapter doesn't currently support subgraph extraction — a neighbourhood filter must be built, and the resulting subgraph may lose unit-grouping context.
**Trade-offs:** Multiple views means more code surface, but avoids premature commitment to a single visualization approach. Ego org diagram reuse is an experiment — may need custom renderer. Subgraph extraction is non-trivial.
**Sources:** graph-stencil-org (existing renderer), ELK layout engine, toOrgGraph adapter (org-adapter.ts), 2024 ScienceDirect ego network study (layered node-link preferred), arc diagram pattern (data-to-viz.com)
**Exploration:** deep-analysis
**Status:** captured — addendum after review (R1-13)
