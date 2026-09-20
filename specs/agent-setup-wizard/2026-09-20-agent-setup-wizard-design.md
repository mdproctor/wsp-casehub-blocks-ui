# Agent Setup Wizard — Design Spec

Making LLM and agent setup trivial. Four independent web components that cover
the full agent lifecycle: LLM provider configuration, agent template
browsing/creation, agent profile display/editing, and relationship management.
Each component is standalone, headless (payload-driven), and works with or
without a backend.

## Scope

| Component | Tag | Purpose |
|-----------|-----|---------|
| Manifest editor | `<agent-manifest-editor>` | LLM provider/model/credential config with presets, auto-detection, guidance |
| Agent catalog | `<agent-catalog>` | Comprehensive agent template browsing, filtering, selection, refinement, from-scratch creation |
| Agent profile | `<agent-profile>` | Rich agent summary page — the "character sheet" — with inline editing |
| Relationship editor | `<agent-relationship-editor>` | Table-based relationship management with experimental visualization views |

Supporting packages:

| Package | Purpose |
|---------|---------|
| `packages/agent-avatar-2d/` | 2D avatar generation from disposition axes, DiceBear Avataaars |
| `packages/blocks-ui-core/` additions | TS type mirrors for Manifest, CredentialRef, ProviderDeclaration |

No composition shell — each component works independently. Host apps wire
them together. The composition question is deferred until the pieces exist and
the natural flow emerges.

These components are designed directly in blocks-ui (bypassing the normal
promotion pipeline) because they are platform-wide agent setup tools with no
single domain home.

## Constraints

**C1: Headless/payload data mode.** All components support inline data mode —
accept payloads for full UI simulation. Standard blocks-ui dual data mode
(endpoint or inline property). "Just works" when backend available, fully
simulatable without it.

**C2: Platform-tier type dependencies.** These components depend on eidos API
types (AgentDescriptor, AgentDisposition, DispositionValue) and platform
agent-config-core types (Manifest, CredentialRef, ProviderDeclaration). TS
type mirrors follow the pattern established by graph-stencil-org (types.ts).
New mirrors added to blocks-ui-core.

**C3: ARIA.** Every component ships with ARIA attributes per blocks-ui
requirements. Tests include ARIA assertions.

---

## Component 1: `agent-manifest-editor`

Makes LLM setup non-daunting through presets, auto-detection, and guided help.

### UI Structure

**Provider cards grid** — one card per known provider (Anthropic, OpenAI,
Google, Ollama/local). Each card shows:
- Provider logo + name
- Detection badge: green if credentials detected, amber if partial, grey if unconfigured
- Model count badge
- Click to expand/select

**Expanded provider section** (progressive disclosure on card click):
- Credential entry — type selector matching CredentialRef sealed interface:
  `env:VAR_NAME` (environment variable), `file:/path` (file path), `ref:name`
  (external credential store via CredentialResolver SPI). Inline help per type.
  Detection pre-fills `env:` fields when matching env vars found.
- Model list — checkboxes grouped by tier (FLAGSHIP, STANDARD, FAST,
  EMBEDDING). Each model: context window, cost hint.
- Alias editor — map capability requirements to model selections (e.g.
  `reasoning-heavy` → Claude Opus). Preset aliases pre-filled, editable.
- Test connection button — validates credentials + model availability end-to-end.

**Guidance layer:**
- Contextual help text at each field
- "Getting started" expandable section per provider
- Warning badges for incomplete/inconsistent configuration

**Preset templates** — card row at top: "Anthropic Production", "OpenAI
Standard", "Local Development (Ollama)", "Multi-provider". Selecting a preset
populates everything; user drills down to customise.

### Detection API

Detection is backend-provided, not client-side. The component consumes
`GET /llm/providers` from the platform#291 wizard API, which returns provider
availability state. ManifestCredentialResolver only resolves known references —
it does not discover providers. If the endpoint includes detection state
(env vars found, Ollama reachable), the UI reflects it. If not, a dedicated
`GET /llm/detect` endpoint is needed — this is a platform API follow-up.

### Data Contract

- **Input:** `Manifest` object (inline property or from `GET /llm/configured`)
- **Output:** emits `pages-event` with the configured `Manifest` payload
- **Detection:** consumes `GET /llm/providers` for availability state
- **Validation:** `POST /llm/configure` for step-by-step validation (when available)

---

## Component 2: `agent-catalog`

Comprehensive agent template browsing. Each template produces a valid
`AgentDescriptor` covering identity, capabilities, disposition, goals,
constraints, and briefing.

### UI Structure

**Popular/Common section** — horizontal scrolling card strip at top. ~8-10
most commonly used templates (Customer Support, Software Developer, Data
Analyst, Medical Triage, Legal Reviewer, Financial Advisor, Operations
Coordinator, Security Analyst). Each card: 2D avatar thumbnail, name, domain
tag, one-line description.

**Filter bar** — pill-based multi-select filtering:
- Domain pills: customer-support, software, medical, legal, finance,
  operations, HR, security, content, data, research
- Task type pills: conversational, analytical, decision-making, monitoring,
  creative, coordination
- Disposition pills: high-autonomy, collaborative, risk-averse, rule-strict,
  adaptive
- Search box for free text filtering

**Catalog grid** — dense card layout. Each card:
- 2D avatar (from agent-avatar-2d)
- Agent name + domain badge
- Key capabilities (top 3, as pills)
- Disposition summary (dominant axis value)
- Click to open in agent-profile

**From-scratch wizard** — "Create Custom Agent" button. Guided steps:
1. Name, domain, slot
2. Capabilities — add from library or define custom (name, quality/latency/cost
   hints, epistemic domains)
3. Disposition — term picker per axis (SOCIAL_ORIENTATION, RULE_FOLLOWING,
   RISK_APPETITE, AUTONOMY, CONFLICT_MODE) with vocabulary-controlled terms
   and plain-language descriptions
4. Goals and constraints — add/remove with priority/severity
5. Briefing — voice/identity/mannerisms (max 2000 chars), template composition
6. Avatar — generates from disposition, browse candidates, customise
7. Preview at each step

### Data Contract

- **Input:** `AgentDescriptor[]` templates (inline property or from endpoint)
- **Output:** emits `pages-event` with selected/customised `AgentDescriptor`
- **Catalog content:** shipped as static JSON, extensible via endpoint

---

## Component 3: `agent-profile`

Rich agent summary — the "character sheet." Every section is editable inline.

### UI Structure

**Header band:**
- Large 2D avatar (left)
- Agent name, slot badge, domain badge
- Version indicator
- Provider + model badge

**Identity section:**
- Briefing text rendered as styled prose
- Template composition indicator (which templates contribute)

**Capabilities section:**
- Card grid, one per capability
- Each card: name, quality/latency/cost as gauges or badges, input/output
  types as pills, epistemic domains as mini bar chart

**Disposition section:**
- 5-axis radar chart for at-a-glance profile
- Per-axis rows: term, weight, plain-language explanation
- Disposition pills matching org diagram stencil for visual consistency

**Goals & Constraints section:**
- Two-column: goals (left, priority badges), constraints (right, severity badges)
- Each: name, description, visibility indicator

**Memory seed preview** (placeholder — D6):
- Beliefs list (text + confidence bar)
- Drives summary (names + intensity)
- Relationship stubs (connected agents + PAD/preset label)
- Greyed/dashed border indicating "will be seeded when backend available"

**Relationships summary:**
- Compact list grouped by kind
- "Edit relationships" link opens agent-relationship-editor

**Editing:** pencil icon per section toggles inline edit mode. Changes emit
`pages-event` with updated `AgentDescriptor`.

### Data Contract

- **Input:** single `AgentDescriptor` (inline or from endpoint)
- **Output:** emits `pages-event` with modified `AgentDescriptor` on edits

---

## Component 4: `agent-relationship-editor`

Table-based relationship editing with experimental visualization views.

### UI Structure

**Agent selector** — dropdown/pill showing the currently selected agent. All
relationships shown are from/to this agent (ego-centric).

**Editing table** (primary, grouped by relationship kind):
- Section headers: SUPERVISES, DELEGATES_TO, ESCALATES_TO, REPORTS_TO,
  BACKS_UP, EXTENDED
- Each row: direction arrow (→/←), target agent (avatar thumbnail + name),
  role, scope, delete button
- Inline "add" row at bottom of each section — agent picker + scope/attestation
- Empty sections collapsed with "+ Add [kind]" button
- Before/after: pending additions highlighted green, pending deletions struck
  through red

**Visualization views** (tab strip above table):
- **Table** (default) — the editing table
- **Arc** — horizontal arc diagram. Ego on left, connected agents arranged
  right sorted by kind. Arcs above for outgoing, below for incoming, coloured
  by kind. Hover highlights table row. Read-only.
- **Ego diagram** — experimental reuse of graph-stencil-org + ELK. Filters org
  YAML to 1-hop neighbourhood. Read-only. If ELK produces poor layout for star
  topology, this view is replaced with a custom renderer later. Note: toOrgGraph
  adapter does not currently support subgraph extraction — a neighbourhood
  filter must be built.

**Relationship form** (for add / complex edit):
- Target agent picker (searchable dropdown from org roster)
- Kind selector with plain-language descriptions
- Optional scope: capability name, domain, or custom
- Optional attestation grant: dimensions, signal types, capability scope
- Direction toggle: "I [kind] them" vs "They [kind] me"

### Data Contract

- **Input:** `AgentRelationship[]` for the selected agent + agent roster
  (inline or from endpoint)
- **Output:** emits `pages-event` with relationship changeset
  (additions/removals)

---

## Package: `agent-avatar-2d`

2D avatar generation from disposition axes. Complements the existing 3D
TalkingHead system in `packages/avatar/`. Boundary: 2D for identity
representation (static), 3D for conversation (interactive).

### Canonical Term Registry

Maps known disposition vocabulary terms to DiceBear Avataaars visual features:

| Axis | Example terms → Visual features |
|------|--------------------------------|
| autonomy | fully-autonomous → confident stance, no accessories; semi-autonomous → relaxed stance, badge; directed → attentive stance, headset |
| socialOrient | collaborative → smile, friendly eyes; competitive → determined mouth, focused eyes; independent → neutral mouth, calm eyes |
| riskAppetite | adventurous → goggles, casual clothing; moderate → no accessories, smart-casual; cautious → glasses, formal |
| ruleFollowing | strict → straight eyebrows, uniform; adaptive → relaxed eyebrows, business; creative → raised eyebrows, casual |
| conflictMode | assertive → set jaw, angled eyebrows; diplomatic → relaxed jaw, gentle eyebrows; avoidant → soft jaw, neutral eyebrows |

Unknown vocabulary terms fall back to nearest known term or neutral default.

### API

- `generateAvatar(disposition)` — deterministic default SVG from disposition
- `generateCandidates(disposition)` — all valid feature combinations (capped ~12-20)
- `customiseAvatar(base, overrides)` — manual feature tweaks

### Component

`<agent-avatar>` — renders 2D avatar at configurable sizes (xs/sm/md/lg).
Accepts disposition or explicit avatar config. Edit button for customisation
popover.

### Integration

DiceBear Avataaars style. MIT licensed, SVG output, fully client-side,
deterministic (same seed → same avatar). The canonical term registry is the
bridge between vocabulary-controlled disposition terms and deterministic
visual features.

---

## Type Additions to blocks-ui-core

Extend `packages/blocks-ui-core/src/types.ts` with TS mirrors for platform
types, following the existing pattern (AgentDescriptor, DispositionAxes are
already mirrored):

```typescript
// From platform agent-config-core
export interface ProviderDeclaration {
  vendor: string;
  credential?: string;
  host?: string;
}

export interface ModelDescriptor {
  id: string;
  metadata?: Record<string, unknown>;
}

export interface ManifestAlias {
  tier?: string;
  capabilities?: string[];
  minContext?: number;
}

export interface ManifestDefaults {
  backend?: string;
}

export interface Manifest {
  providers?: Record<string, ProviderDeclaration>;
  models?: Record<string, ModelDescriptor>;
  aliases?: Record<string, ManifestAlias>;
  defaults?: ManifestDefaults;
  sources?: ManifestSource[];
  localModels?: LocalModelDeclaration[];
}

export type CredentialRef =
  | { type: 'env'; name: string }
  | { type: 'file'; path: string }
  | { type: 'ref'; name: string };
```

---

## Existing Issues Alignment

| Issue | Repo | Relevance |
|-------|------|-----------|
| #285 | platform | LLM model registry epic — manifest editor consumes this |
| #315 | platform | AgentProvider simulation adapter — supports headless mode |
| #325 | platform | YAML-driven simulation config — aligns with payload-driven approach |
| #175 | eidos | Standardised demo launchers — catalog + profile serve this |
| #157 | blocks-ui | Rich org diagram — relationship editor complements this |

---

## Out of Scope

- **Composition shell** — deferred until components exist and natural flow emerges
- **Full neocortex seeding API** — memory seeding section is placeholder with payload preview
- **Resource configuration** (calendar, RAG sources, mailbox per agent) — no API exists yet
- **3D avatar changes** — existing TalkingHead system is unchanged; 2D is complementary
- **Custom ego diagram renderer** — experiment with ELK first, build custom only if needed

## References

- `packages/graph-stencil-org/src/types.ts` — existing AgentDescriptor, DispositionAxes TS mirrors
- `packages/graph-stencil-org/src/adapter/enrichment.ts` — enrichWithDescriptors pattern
- `packages/graph-stencil-org/src/stencils/org-agent.ts` — disposition pill rendering
- `packages/avatar/` — existing 3D TalkingHead avatar system
- `platform/agent-config-core` — Manifest, ManifestLoader, CredentialRef, ProviderDeclaration
- `eidos/eval/src/test/resources/profiles/customer-support-agent.yaml` — full agent profile example
- `examples/wacky-manor/src/main/java/.../ManorCognitiveSeeder.java` — memory seeding reference (slot 196)
- `examples/wacky-manor/src/main/java/.../CharacterCognition.java` — runtime memory-first belief retrieval
- `neocortex/cognitive-index/src/main/java/.../CognitiveDerivationEngine.java` — personality→cognitive defaults
- platform#291 — wizard API (GET /llm/providers, POST /llm/configure, GET /llm/configured)
- 2024 ScienceDirect ego network study — layered node-link preferred for ego networks
- DiceBear Avataaars — avatar generation library (MIT, SVG, client-side, deterministic)
