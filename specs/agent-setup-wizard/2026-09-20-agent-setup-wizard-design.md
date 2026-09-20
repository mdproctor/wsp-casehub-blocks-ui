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
New mirrors added to blocks-ui-core. Precedent: `packages/avatar/` and
`packages/graph-stencil-org/` already live directly in blocks-ui without
domain-app promotion.

**C3: ARIA.** Every component ships with ARIA attributes per blocks-ui
requirements. Tests include ARIA assertions.

**C4: Disposition model — deliberate simplification.** The Java
`AgentDisposition` supports multi-term weighted values per axis
(`List<DispositionValue>` where each has `term` + `weight`). The wizard
deliberately simplifies to single-term selection for ergonomics — the common
case. An "advanced" toggle allows multi-term weighted profiles for power
users. Avatar generation uses `primaryTerm()` (first/highest-weight term per
axis). Round-tripping preserves multi-term if the original had it — the
simplified view shows the primary term but does not discard secondary terms.

**C5: Data integration pattern.** These components consume object-tree data
(Manifest, AgentDescriptor), not tabular data. They do NOT use
DataSourceMixin/TypedDataSet — those are for column-oriented table data.
Instead, they follow a direct-fetch pattern: `endpoint` property for backend
URL (returns JSON object), `data` property for inline payload, component
manages its own fetch lifecycle. This matches `gdpr-erasure-action` (which
extends LitElement directly, no DataSourceMixin) and `worker-task-pane`
(direct fetch, no DataSourceMixin).

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

- **Input:** `Manifest` object (inline `data` property or from `endpoint` → `GET /llm/configured`)
- **Output:** emits `pages-event` topic `manifest:configured` with the configured `Manifest` payload
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
   hints, epistemic domains). Includes `delegation` toggle ("can spawn
   sub-agents") — this is a platform capability, not a disposition trait.
3. Disposition — primary term picker per axis (SOCIAL_ORIENTATION,
   RULE_FOLLOWING, RISK_APPETITE, AUTONOMY, CONFLICT_MODE) with
   vocabulary-controlled terms and plain-language descriptions. "Advanced"
   toggle reveals multi-term weighted entry per axis (add secondary terms
   with weights). See constraint C4.
4. Goals and constraints — add/remove with priority/severity
5. Briefing — voice/identity/mannerisms (max 2000 chars), template composition
6. Avatar — generates from disposition, browse candidates, customise
7. Preview at each step

### Data Contract

- **Input:** `AgentDescriptor[]` templates (inline `data` property or from `endpoint`)
- **Output:** emits `pages-event` topic `agent:selected` on pick, `agent:created` on from-scratch completion, with `AgentDescriptor` payload
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
- 5-axis radar chart using primary term weight per axis (single scalar per
  axis — for multi-term dispositions, shows the dominant term's weight)
- Per-axis rows: primary term + weight, secondary terms if present, plain-language explanation
- Disposition pills matching org diagram stencil for visual consistency
- `delegation` badge if enabled

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
`pages-event` topic `agent:updated` with updated `AgentDescriptor`.

### Data Contract

- **Input:** single `AgentDescriptor` (inline `data` property or from `endpoint`)
- **Output:** emits `pages-event` topic `agent:updated` with modified `AgentDescriptor` on edits

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
- **Ego diagram** — experimental reuse of graph-stencil-org + ELK. Read-only.
  **Prerequisite:** a neighbourhood filter function must be built in
  graph-stencil-org: `extractEgoSubgraph(model: GraphModel, egoAgentId: string): GraphModel`
  — returns the 1-hop neighbourhood with unit context preserved (the ego
  agent's unit becomes the root container). Without this, the view renders the
  entire graph and defeats the ego-centric purpose. If ELK produces poor layout
  for the resulting star topology, this view is replaced with a custom renderer.

**Relationship form** (for add / complex edit):
- Target agent picker (searchable dropdown from org roster)
- Kind selector with plain-language descriptions
- Optional scope: capability name, domain, or custom
- Optional attestation grant: dimensions, signal types, capability scope
- Direction toggle: "I [kind] them" vs "They [kind] me"

### Data Contract

- **Input:** `AgentRelationship[]` for the selected agent + agent roster
  (inline `data` property or from `endpoint`)
- **Output:** emits `pages-event` topic `relationship:changed` with relationship
  changeset (additions/removals)

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

Unknown vocabulary terms fall back to the axis's neutral default (not
"nearest known term" — no similarity metric). Deterministic and predictable.

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
and eidos types. Verified against decompiled Java bytecode from
`casehub-platform-agent-config-core-0.2-SNAPSHOT.jar` and `casehub-eidos-api`.

### Manifest types (from platform agent-config-core)

```typescript
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
```

### Expanded AgentDescriptor (extending graph-stencil-org mirror)

The existing `AgentDescriptor` in graph-stencil-org has 6 optional fields.
The agent-facing components need a broader mirror. Rather than duplicating,
extend `blocks-ui-core` with the full descriptor:

```typescript
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

The existing graph-stencil-org `AgentDescriptor` (6 fields) remains for
enrichment — it is a subset. `FullAgentDescriptor` is used by the agent
catalog, profile, and from-scratch wizard. The avatar generator accepts
`AgentDisposition` and uses `primaryTerm()` logic: first element of each
axis's `DispositionValue[]`.

### Event topics

Register in `blocks-ui-core/src/types/events.ts`:

```typescript
'manifest:configured'    // Manifest payload
'agent:selected'         // FullAgentDescriptor payload (catalog pick)
'agent:created'          // FullAgentDescriptor payload (from-scratch)
'agent:updated'          // FullAgentDescriptor payload (profile edit)
'relationship:changed'   // AgentRelationship[] changeset
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

## Test Strategy

All components use vitest + jsdom, following the existing blocks-ui pattern.

**Per-component:**
- **Manifest editor:** Mock `GET /llm/providers` responses for detection
  states (all detected, partial, none). Test credential type switching
  (env/file/ref). Test preset selection populates fields. Test connection
  validation emits correct event.
- **Agent catalog:** Unit tests on filter pipeline (pill selection → filtered
  results). Test from-scratch wizard step navigation. Test catalog card
  rendering from fixture data.
- **Agent profile:** Test inline edit toggle per section. Test disposition
  radar chart renders from multi-term weighted data. Test event emission on
  edit.
- **Relationship editor:** Test grouped table rendering from fixture
  relationships. Test add/remove changeset emission. Test before/after
  highlighting (pending additions/deletions).
- **Avatar-2d:** Test canonical term registry mapping (known terms → expected
  features). Test unknown term → neutral default fallback. Test determinism
  (same disposition → same SVG).

**ARIA:** Every component test file includes assertions verifying `role` and
`aria-label` attributes per blocks-ui ARIA requirements.

**Inline data mode:** Every component tested with inline `data` property
(no endpoint) to verify payload-driven rendering.

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
