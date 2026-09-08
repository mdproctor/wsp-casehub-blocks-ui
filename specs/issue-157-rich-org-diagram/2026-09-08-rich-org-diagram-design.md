# Rich Org Diagram — Design Spec

**Issue:** casehubio/blocks-ui#157
**Date:** 2026-09-08
**Status:** Draft
**Builds on:** casehubio/blocks-ui#155 (base org diagram — adapter, stencils, archetype detection, layout, editing)

---

## Overview

Enrich the existing org-diagram component with full information density: rich agent cards showing descriptor properties (slot, disposition, capabilities, supervision targets, escalation paths, backup, attestation), kind-differentiated unit containers with capability pills, labeled relationship lines with scope badges, standalone panels (escalation chain, attestation grants, legend), and interactive features (collapsible units, hover tooltips, relationship highlighting).

The reference target is `casehubio/eidos/docs/diagrams/gastown-org-structure.svg` — this spec matches or exceeds its information density in an interactive, editable form.

---

## Data Model

### Agent Descriptor Integration (D1, D2)

The component receives two data sources via a single API surface:

| Property | Type | Source | Editable |
|----------|------|--------|----------|
| `yaml` / `src` | `string` | Org YAML (units, members, relationships) | Yes — via DiagramBaseMixin pipeline |
| `agents` | `Record<string, AgentDescriptor>` | Agent descriptors keyed by agentId | No — read-only enrichment |

```typescript
interface AgentDescriptor {
  slot?: string;
  capabilities?: AgentCapability[];
  disposition?: Partial<DispositionAxes>;
  goals?: AgentGoal[];
  constraints?: AgentConstraint[];
  briefing?: string;
}

interface DispositionAxes {
  autonomy?: string;
  ruleFollowing?: string;
  socialOrient?: string;
  riskAppetite?: string;
  conflictMode?: string;
}
```

The adapter `toOrgGraph(yaml, agents?)` merges descriptor data into `GraphNode.properties` for each agent node. When `agents` is undefined or a descriptor is missing for an agent, the card degrades gracefully to the basic rendering (agentId + role only).

### Derived Relationship Data (D3)

After building nodes and edges, the adapter computes per-agent derived data by walking the relationship graph:

| Derived property | Computation | Stored in |
|-----------------|-------------|-----------|
| `supervisionTargets` | Outgoing SUPERVISES edges from this agent | `string[]` — target agentIds |
| `escalationChain` | Walk ESCALATES_TO from this agent to terminal | `string[]` — ordered chain of agentIds |
| `backupAgents` | BACKS_UP edges involving this agent | `{agentId: string, scope?: string, direction: 'backs' \| 'backed-by'}[]` |
| `attestationGrants` | From SUPERVISES edges with attestation property | `{targetAgentId: string, scope?: string, dimensions: string[], signalTypes?: string[]}[]` |

These are stored in `GraphNode.properties` alongside the descriptor data. The stencil renderer reads them like any other property.

### Adapter Signature Change

```typescript
// Before
export function toOrgGraph(yaml: string): OrgAdapterResult;

// After
export function toOrgGraph(
  yaml: string,
  agents?: Readonly<Record<string, AgentDescriptor>>,
  options?: {
    collapsedUnits?: ReadonlySet<string>;
    kindColors?: Readonly<Record<string, { start: string; end: string }>>;
  },
): OrgAdapterResult;

// OrgAdapterResult extended
export interface OrgAdapterResult {
  readonly model: GraphModel;
  readonly yamlPaths: ReadonlyMap<string, readonly (string | number)[]>;
  readonly nodeSizes: ReadonlyMap<string, { width: number; height: number }>;
  readonly escalationChains: readonly { path: string[]; terminal: string }[];
  readonly attestationSummary: readonly {
    source: string; target: string;
    scope?: string; dimensions: string[];
    signalTypes?: string[];
  }[];
}
```

`escalationChains` and `attestationSummary` are org-wide derived data consumed by the standalone panels (D4). They're computed during the same graph walk that produces per-agent derived properties.

When `collapsedUnits` contains a unit ID, the adapter omits that unit's agent nodes and their edges from the graph model, and produces the unit node at a compact height (~40px, header only).

### Color Resolution in the Adapter

The adapter resolves kind-to-color mappings and stores them in node properties so stencils remain pure renderers:

- **Unit nodes** get `kindColorStart` and `kindColorEnd` properties (resolved from `kindColors` option → default palette → auto-assignment fallback).
- **Agent nodes** get `unitKind` (parent unit's kind) and `unitColorStart`/`unitColorEnd` (parent unit's resolved colors) so the agent stencil can tint its header, circle indicator, and border without accessing the parent node.

This ensures stencils never need the palette or the parent graph — everything they render is in their own `node.properties`.

---

## Rich Agent Card Stencil (org-agent)

Replace the current minimal card with a multi-row rendered card matching the reference SVG density.

### Card Structure

```
┌──────────────────────────────────────────┐
│ ● agent-id                    ROLE BADGE │  ← header (tinted bg)
├──────────────────────────────────────────┤
│ SLOT     worker                          │
│ CAPS     full-stack-code-work            │
│ DISPOSITION  [autonomy: semi-auto]       │
│              [rules: measured]           │
│ SUPERVISES   polecat-1, polecat-2        │
│ ESCALATES    → witness-α → deacon → boot │
│ BACKUP       polecat-2 (scope: code-a…)  │
│ ATTESTATION  [LATENCY] [ATTEST_RATE]     │
│              signals: COMPLIANT, VIOLATED│
└──────────────────────────────────────────┘
```

### Row Rendering Rules

Each row is conditionally rendered based on data availability:

| Row | Condition | Label style | Value style |
|-----|-----------|-------------|-------------|
| SLOT | `descriptor.slot` present | Gray 9px bold | Dark 9px |
| CAPS | `descriptor.capabilities` non-empty | Gray 9px bold | Dark 9px, comma-separated |
| DISPOSITION | Any axis populated | Gray 9px bold | Color-coded pills (see below) |
| SUPERVISES | `supervisionTargets` non-empty | Gray 9px bold | Dark 9px, comma-separated agentIds |
| ESCALATES | `escalationChain` non-empty | Gray 9px bold | Red 9px, `→` separated chain |
| BACKUP | `backupAgents` non-empty | Gray 9px bold | Dark 9px, with scope in parens |
| ATTESTATION | `attestationGrants` non-empty | Gray 9px bold | Purple dimension pills + signal text |

### Header

- Colored circle indicator: matches unit kind color (purple for oversight, blue for rig — from D6 palette)
- Agent ID: bold 12px, colored to match unit kind
- Role badge: uppercase 8px, right-aligned, unit kind color

### Disposition Axis Pills

| Axis | Background | Text color |
|------|------------|------------|
| `autonomy` | `#fed7d7` | `#c53030` |
| `ruleFollowing` | `#fefcbf` | `#975a16` |
| `socialOrient` | `#c6f6d5` | `#276749` |
| `riskAppetite` | `#c6f6d5` | `#276749` |
| `conflictMode` | `#ebf8ff` | `#2b6cb0` |

Format: `axis-short-name: value` in a rounded pill (3px radius, 14px height).

Short names: `autonomy`, `rules`, `social`, `risk`, `conflict`.

### Node Sizing (D5)

The adapter pre-computes each agent node's height based on populated rows:

```typescript
const ROW_HEIGHT = 16;
const HEADER_HEIGHT = 28;
const DISPOSITION_ROW_HEIGHT = 18;
const ATTESTATION_HEIGHT = 32;
const PADDING = 12;
const AGENT_WIDTH = 280;

function computeAgentHeight(props: Record<string, unknown>): number {
  let h = HEADER_HEIGHT + PADDING;
  if (props['slot']) h += ROW_HEIGHT;
  if ((props['capabilities'] as unknown[])?.length) h += ROW_HEIGHT;
  const disposition = props['disposition'] as Record<string, string> | undefined;
  if (disposition && Object.keys(disposition).length > 0) {
    h += DISPOSITION_ROW_HEIGHT;  // pills wrap on one line
  }
  if ((props['supervisionTargets'] as string[])?.length) h += ROW_HEIGHT;
  if ((props['escalationChain'] as string[])?.length) h += ROW_HEIGHT;
  if ((props['backupAgents'] as unknown[])?.length) h += ROW_HEIGHT;
  if ((props['attestationGrants'] as unknown[])?.length) h += ATTESTATION_HEIGHT;
  return h;
}
```

The `nodeSizes` map is passed to `computeElkLayout()` so ELK allocates correct space.

---

## Rich Unit Container Stencil (org-unit)

### Header Gradient by Kind (D6)

Default palette:

| Kind | Gradient start | Gradient end |
|------|---------------|--------------|
| `supervision-hierarchy` | `#553c9a` | `#6b46c1` |
| `rig` | `#2c5282` | `#3182ce` |
| `department` | `#276749` | `#38a169` |
| `holarchy` | `#3730a3` | `#4f46e5` |
| `team` | `#9c4221` | `#dd6b20` |

Unknown kinds: auto-assigned from unused slots in rotation order. The component's `kindColors` property (`Record<string, {start: string, end: string}>`) overrides any default.

The kind color is also used for:
- Agent card header tint (lighter version)
- Agent card circle indicator fill
- Agent card border stroke
- Agent ID text color

### Capability Pills

Below the header, render unit capabilities as pill badges:

```
[CAP: full-stack-code-work] [CAP: code-review]
```

Style: `#ebf8ff` background, `#90cdf4` border, `#2b6cb0` text, 8px font, 4px radius.

### Collapsible (D8)

Clicking the unit header toggles collapse. The component maintains a `_collapsedUnits: Set<string>` state. On toggle, re-runs the adapter with `{ collapsedUnits }` and re-layouts.

Collapsed state: header bar only (~40px), no agents visible, member count badge shows how many are hidden.

---

## Edge Labels (D7)

A pure function `applyOrgEdgeLabels(edges, selectedAgentId?)` post-processes ReactFlow edges:

### Label Rules

| Edge type | Label | Background | Border | Text color |
|-----------|-------|------------|--------|------------|
| `org-supervises` | `scope: {capabilityName}` when scoped, none otherwise | `#e9d8fd` | `#d6bcfa` | `#553c9a` |
| `org-delegates-to` | `"DELEGATES_TO"` always | `#e6fffa` | `#81e6d9` | `#2c7a7b` |
| `org-escalates-to` | none (line style is distinctive enough) | — | — | — |
| `org-reports-to` | none | — | — | — |
| `org-backs-up` | `"BACKS_UP"` + scope if present | `#ebf8ff` | `#90cdf4` | `#2b6cb0` |
| `org-extended` | `extendedKind` value | `#f3e8ff` | `#c4b5fd` | `#6d28d9` |

ReactFlow properties set per edge: `label`, `labelStyle: { fontSize: 8, fontWeight: 600 }`, `labelBgStyle: { fill, stroke }`, `labelBgPadding: [3, 6]`, `labelBgBorderRadius: 3`.

### Selection Highlighting (D10)

When `selectedAgentId` is provided, edges connected to that agent get `className: 'org-edge-highlighted'`. All other edges get `style: { opacity: 0.15 }`. Clearing selection restores all edges to full opacity.

---

## Standalone Panel Components (D4)

### `<blocks-escalation-chain-panel>`

**Location:** `components/escalation-chain-panel/`

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `chains` | `{path: string[], terminal: string}[]` | Escalation chains from adapter |
| `highlightAgent` | `string \| undefined` | Agent to highlight in the chain |

**Events:**
- `agent-click` — `CustomEvent<{agentId: string}>` when an agent name is clicked

**Rendering:**
Red-tinted panel (`#fff5f5` background, `#feb2b2` border). Title: "ESCALATION CHAIN" bold red. Each chain as a horizontal sequence: agent names in dark text, red `→` arrows between them, `(terminal)` gray text after the final agent. The highlighted agent (if any) gets a bold + underline treatment.

**ARIA:** `role="region"`, `aria-label="Escalation chains"`. Each chain is a list item.

### `<blocks-attestation-panel>`

**Location:** `components/attestation-panel/`

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `grants` | `{source: string, target: string, scope?: string, dimensions: string[], signalTypes?: string[]}[]` | Attestation grants from adapter |
| `highlightAgent` | `string \| undefined` | Agent to highlight |

**Events:**
- `agent-click` — `CustomEvent<{agentId: string}>`

**Rendering:**
Purple-tinted panel (`#faf5ff` background, `#d6bcfa` border). Title: "ATTESTATION GRANTS (source → target)" bold purple. For each grant: scope line, dimension pills (purple `#e9d8fd`), signal type pills (green `#c6f6d5` for COMPLIANT, red `#fed7d7` for VIOLATED), store target text.

**ARIA:** `role="region"`, `aria-label="Attestation grants"`.

### `<blocks-org-legend>`

**Location:** `components/org-legend/`

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `kindColors` | `Record<string, {start: string, end: string}>` | Kind-to-color mapping (rendered as swatches) |
| `showDisposition` | `boolean` | Whether to show disposition axis section (default true) |
| `showAttestation` | `boolean` | Whether to show attestation section (default true) |

**Sections:**
1. **Relationships** — line samples for SUPERVISES, ESCALATES_TO, BACKS_UP, DELEGATES_TO with matching stroke styles
2. **Unit Kinds** — color swatches with kind names (from `kindColors`)
3. **Agent Properties** — dot indicators (colored per unit kind)
4. **Disposition Axes** — color swatches for each axis with labels
5. **Scope & Attestation** — pill samples: scoped supervision, attestation grant, unit capability
6. **Eidos Model Layers** — brief text description

**ARIA:** `role="region"`, `aria-label="Organization diagram legend"`.

---

## Hover Tooltip (D9)

### Implementation

The org-diagram component renders a floating tooltip on agent node mouseover. The tooltip shows a rendered subset of agent properties — same visual language as the card (pills, badges, text rows) but includes additional detail (goals, constraints) not shown in the card.

**Tooltip content (rendered, read-only):**
- Agent ID + role badge (header)
- Slot
- Capabilities
- Disposition pills
- Goals (name + priority)
- Constraints (name + severity)
- Briefing (truncated to 2 lines)

**Positioning:** Above the hovered node, clamped to viewport. Dismisses on mouseout with a small delay (150ms) to prevent flicker when moving between adjacent cards.

**Relationship to property panel:** The tooltip is a quick preview. The property panel (on click/selection) shows everything the tooltip shows plus editing forms for editable fields (agentId, role via the org YAML).

---

## Component Composition

The org-diagram composes the panels below the canvas:

```
┌─────────────────────────────────────────────────────────┐
│ Toolbar                                                 │
├──────┬──────────────────────────────┬───────────────────┤
│      │                              │                   │
│ Pal  │    Graph Canvas              │  Properties       │
│      │                              │                   │
├──────┴──────────────────────────────┴───────────────────┤
│ ┌─── Escalation Chain ──┐ ┌── Attestation Grants ─────┐ │
│ │ polecat-1 → witness…  │ │ deacon → witnesses        │ │
│ │ polecat-2 → witness…  │ │ scope: rig-monitoring     │ │
│ └───────────────────────┘ └───────────────────────────┘ │
├─────────────────────────────────────────────────────────┤
│ Legend                                                   │
└─────────────────────────────────────────────────────────┘
```

The panels receive data from the adapter result (`escalationChains`, `attestationSummary`). The legend receives `kindColors`. Agent click events from panels trigger node selection in the canvas.

---

## AgentDescriptor Type (new in graph-stencil-org)

```typescript
export interface AgentDescriptor {
  slot?: string;
  capabilities?: AgentCapability[];
  disposition?: Partial<DispositionAxes>;
  goals?: AgentGoal[];
  constraints?: AgentConstraint[];
  briefing?: string;
}

export interface DispositionAxes {
  autonomy?: string;
  ruleFollowing?: string;
  socialOrient?: string;
  riskAppetite?: string;
  conflictMode?: string;
}
```

This type is defined in `graph-stencil-org/src/types.ts` alongside the existing org types. `AgentCapability`, `AgentGoal`, and `AgentConstraint` are already defined there.

---

## Testing

### graph-stencil-org tests (adapter enrichment)

- **Descriptor merge:** Parse org YAML + agents map → verify agent nodes have descriptor properties (slot, disposition, capabilities)
- **Missing descriptors:** Parse with partial agents map → verify missing descriptors produce basic nodes (agentId + role only)
- **Supervision targets:** Verify each agent's `supervisionTargets` matches outgoing SUPERVISES edges
- **Escalation chains:** Verify full chain walk from leaf to terminal (e.g. polecat-1 → witness-alpha → deacon → boot)
- **Backup agents:** Verify bidirectional backup with scope
- **Attestation grants:** Verify grants derived from SUPERVISES edges with attestation property
- **Node sizing:** Verify `nodeSizes` map has correct heights based on populated rows. Agent with all fields → max height. Agent with only role → min height.
- **Collapsed units:** Verify adapter omits agent nodes for collapsed units, produces compact unit height
- **Edge labels:** Verify `applyOrgEdgeLabels` sets correct label/style on scoped edges, no label on unscoped SUPERVISES

### org-diagram tests

- **ARIA:** Verify role, aria-label on host, canvas, toolbar, tooltip
- **Agents property:** Set `agents` map → verify rich card rendering (disposition pills visible)
- **Hover tooltip:** Mouseover agent node → verify tooltip appears with goals/constraints
- **Collapsible units:** Click unit header → verify member count shown, agents hidden. Click again → verify expanded.
- **Selection highlighting:** Click agent → verify connected edges highlighted, others dimmed
- **Layout override:** Set layoutStrategy → verify ELK options correct
- **Kind colors:** Set custom kindColors → verify unit header uses override color

### Standalone panel tests

- **Escalation chain panel:** Pass chains → verify rendered paths with arrows and terminal markers. Click agent name → verify event emitted.
- **Attestation panel:** Pass grants → verify dimension pills and signal pills rendered. Verify scope text.
- **Legend:** Pass kindColors → verify swatches rendered. Toggle showDisposition → verify section hidden.
- **ARIA:** All three panels: verify role and aria-label.

---

## BlocksComponentRegistry Entries

Per the component-registry-props protocol, each new component needs:

| Component | Tag | Props interface | Registry entry |
|-----------|-----|-----------------|----------------|
| Escalation chain panel | `blocks-escalation-chain-panel` | `EscalationChainPanelProps` | Yes |
| Attestation panel | `blocks-attestation-panel` | `AttestationPanelProps` | Yes |
| Org legend | `blocks-org-legend` | `OrgLegendProps` | Yes |

After adding entries, run `yarn workspace @casehubio/blocks-ui-schema run generate` to regenerate Zod schemas.

---

## References

- `casehubio/eidos/docs/diagrams/gastown-org-structure.svg` — reference SVG target
- `packages/graph-stencil-org/src/adapter/org-adapter.ts` — existing adapter (toOrgGraph)
- `packages/graph-stencil-org/src/stencils/org-agent.ts` — current minimal agent stencil
- `packages/graph-stencil-org/src/stencils/org-unit.ts` — current minimal unit stencil
- `packages/graph-stencil-org/src/types.ts` — existing org domain types
- `components/org-diagram/src/blocks-org-diagram.ts` — existing diagram component
- `docs/protocols/blocks-ui/node-decoration-runtime-boundary.md` — descriptor data is definition, not decoration
- `docs/protocols/blocks-ui/component-registry-props.md` — new components need registry entries
- `docs/protocols/blocks-ui/stencil-package-isolation.md` — panels are separate components, not stencil cross-imports
- `docs/specs/issue-155-org-diagram/2026-09-05-org-diagram-design.md` — base org diagram spec
- `docs/specs/issue-155-org-diagram/decisions.md` — 15 prior decisions (D1-D15)
- ReactFlow Edge API — label, labelBgStyle, labelBgPadding, labelBgBorderRadius
- `casehubio/blocks-ui#157` — issue with full requirements
- `casehubio/blocks-ui#157` comment — OrgDiagramData shape guidance
