## D1: Agent descriptor data flow

**Choice:** Merge into GraphNode.properties during adaptation — extend `toOrgGraph` to accept both YAML and the agents map: `toOrgGraph(yaml, agents?)`. The adapter looks up each member's descriptor and merges slot, disposition, capabilities into the node's properties.
**Alternatives:**
- Pass via NodeDecoration — violates node-decoration-runtime-boundary protocol; descriptors are static identity, not runtime overlays.
- Renderer context/lookup — requires changing the stencil render signature across graph-renderer infrastructure for one consumer's need.
**Rationale:** The adapter already transforms domain data into graph model properties. Descriptor data is static definition data (same as agentId, role). The protocol explicitly says properties carry "definition data." No infrastructure changes needed.
**Trade-offs:** GraphNode.properties grows larger per agent node. Acceptable — the data is needed for rendering and is bounded per agent.
**Sources:** node-decoration-runtime-boundary protocol, org-adapter.ts, issue #157 comment (OrgDiagramData shape)
**Exploration:** quick
**Status:** captured

## D2: Component API — YAML + agents property

**Choice:** Keep YAML for org structure (`yaml`/`src` props via DiagramBaseMixin), add a new `agents` property (`Record<string, AgentDescriptor>`) for read-only descriptor enrichment. The adapter merges them. Descriptors are not editable from this component.
**Alternatives:**
- New `data` property accepting `OrgDiagramData` — bypasses YAML entirely, loses DiagramBaseMixin editing pipeline (undo/redo, CST-preserving edits, persistence). Would require building a parallel editing pipeline and blurs ownership boundary.
**Rationale:** The split mirrors domain ownership: org structure is editable here (OrgRegistry), agent identity is read-only context (AgentRegistry). YAML pipeline stays intact for structural editing. Descriptor enrichment layers on top without affecting the editing contract.
**Trade-offs:** Consumers must split a joined REST response into YAML + agents map. Trivial orchestration in the workbench or app layer.
**Sources:** DiagramBaseMixin (pages-diagram-core), issue #157 comment (OrgDiagramData shape), D1
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D3: Derived relationship data computed in adapter

**Choice:** Compute per-agent derived data (supervision targets, escalation chain, backup agents with scope) in the adapter after building nodes and edges. Store as structured arrays in GraphNode.properties. The stencil renderer reads them like any other property.
**Alternatives:**
- Compute in stencil renderer — mixes graph traversal with rendering. Stencil render function only sees its own node + decoration, would need the full edge list. Breaks the clean render contract.
**Rationale:** The adapter already has the full graph, builds the agent index, and resolves multi-unit relationships. Chain-walking is a natural extension of adaptation. The stencil stays a pure renderer of pre-computed properties.
**Trade-offs:** Adapter grows more complex. Acceptable — it's the right place for graph traversal logic.
**Sources:** org-adapter.ts (existing agentIndex, resolveAgentNode), D1
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D4: Escalation chain, attestation, and legend as standalone Lit components

**Choice:** Standalone Lit components — `<blocks-escalation-chain-panel>`, `<blocks-attestation-panel>`, `<blocks-org-legend>`. Each accepts data via properties and emits events for interaction (e.g. clicking an agent name in the escalation chain). Designed for customisation via parameters and callbacks, not diagram-internal chrome.
**Alternatives:**
- Rendered by org-diagram component outside canvas — tightly coupled to the diagram, no reuse potential. Unnecessary coupling.
- Rendered as synthetic graph nodes — fighting ELK layout for what are annotations, not graph entities.
**Rationale:** No downside to Lit components — reuse may surface later (e.g. attestation panel in a trust workbench context, escalation chain in an incident view). Parameters and callbacks make them composable beyond the org diagram. Follows the repo's established pattern of focused, reusable components.
**Trade-offs:** Three new component packages to maintain. Lightweight — each is a rendering component with no data fetching.
**Sources:** Reference SVG (escalation chain panel, attestation detail panel, legend), component-registry-props protocol
**Exploration:** quick
**Status:** captured

## D5: Pre-compute agent card heights in adapter

**Choice:** Pre-compute node heights in the adapter based on populated content rows. The adapter knows what properties each agent has (descriptor merge D1, derived data D3). Height formula: base header + conditional rows (slot, capabilities, disposition, supervision, escalation, backup, attestation). Return a `nodeSizes` map alongside the GraphModel. Stencil renderer uses the same data, so height and content agree.
**Alternatives:**
- Fixed height with scroll overflow — wastes space on sparse agents, clips dense ones. Loses the information density the reference SVG achieves.
- Two-pass render — render once to measure, re-layout with actual sizes. Doubles rendering work, adds pipeline complexity.
**Rationale:** Deterministic, no double-render. The height formula is simple because content rows are known from adapter output. Adapter and stencil agree on sizing because they use the same data.
**Trade-offs:** Height formula must stay in sync with stencil rendering. Acceptable — both are in graph-stencil-org and tested together.
**Sources:** Reference SVG (variable-height agent cards), ElkLayoutOptions.nodeSizes, D1, D3
**Exploration:** quick
**Depends on:** D1, D3
**Status:** captured

## D6: Unit kind color mapping — built-in palette with override

**Choice:** Ship a default palette mapping common kinds to gradient pairs (supervision-hierarchy → purple, rig → blue, department → teal, holarchy → indigo). Unknown kinds auto-assigned from unused palette slots. Component exposes `kindColors` property (`Record<string, {start: string, end: string}>`) for consumer overrides.
**Alternatives:**
- Hash-derived colors — no manual mapping but no semantic association. "rig" could end up red.
- Fully configurable, no defaults — tedious for the common case.
**Rationale:** Semantic defaults matter — purple for oversight, blue for work units. Override property for domains with custom vocabularies. Deterministic and predictable.
**Trade-offs:** Default palette may not cover all future kind vocabularies. Override property handles this.
**Sources:** Reference SVG (headerOversight purple, headerRig blue gradients)
**Exploration:** quick
**Status:** captured

## D7: Edge labels via post-processing in org-diagram

**Choice:** Post-process edges after `toReactFlowGraph` in the org-diagram component. Set ReactFlow's native `label`, `labelStyle`, `labelBgStyle`, `labelBgPadding`, `labelBgBorderRadius` properties based on edge data (scope, kind, attestation). Pure function: `applyOrgEdgeLabels(edges): edges`. No cross-repo change needed.
**Alternatives:**
- Extend graph-renderer mapping — cross-repo change for org-specific rendering logic. Wrong layer.
- Custom ReactFlow edge components — heavy infrastructure change for styled text.
**Rationale:** ReactFlow natively supports edge labels including background styling. The canvas already passes edges through to ReactFlow unchanged. Org-specific label logic stays in graph-stencil-org.
**Trade-offs:** Text-only labels (no React element pills). Sufficient — ReactFlow's labelBgStyle produces pill-like appearance with background color, padding, and border radius.
**Sources:** ReactFlow Edge API (label, labelBgStyle), ReactFlowApp.tsx:234 (edges passed directly), reference SVG scope badges
**Exploration:** quick
**Status:** captured

## D8: Collapsible units via adapter filtering with re-layout

**Choice:** Clicking a unit header toggles a `collapsedUnits: Set<string>` on the component. The adapter accepts this set, omits agent nodes and their edges for collapsed units, and produces the unit node at a fixed compact height (header-only, ~40px). ELK re-layouts with the reduced graph. Expanding restores the full node set.
**Alternatives:**
- CSS-only hide — toggle display:none on agent cards. ELK doesn't know about the collapse, unit container keeps full-size layout. Wastes space and looks wrong.
**Rationale:** ELK needs accurate sizes to layout correctly. The adapter already controls what nodes enter the graph; filtering by collapse state is natural.
**Trade-offs:** Re-layout on collapse/expand. Acceptable — ELK is fast for org-sized graphs (5-20 nodes).
**Sources:** org-adapter.ts (node generation loop), ElkLayoutOptions.nodeSizes
**Exploration:** quick
**Status:** captured

## D9: Hover tooltip for quick scanning + property panel for full detail

**Choice:** Both hover and selection. Mouseover shows a lightweight rendered tooltip with key properties (slot, disposition pills, capabilities, supervision targets) — read-only, no forms. Click/selection opens the full property panel with all details and editing. The tooltip enables fast model scanning without clicking each node; the panel provides depth.
**Alternatives:**
- Property panel only — requires clicking each node to see any detail. Slow for scanning.
- Rich hover only — competes with the property panel, creates two editing surfaces.
**Rationale:** Different interaction modes serve different tasks: hover for overview and comparison, click for deep inspection and editing. The tooltip shows a subset of properties in rendered form (pills, badges, text rows) — same visual language as the agent card but with additional detail like goals and constraints.
**Trade-offs:** Two rendering paths for agent detail (tooltip + panel). Tooltip is simpler — rendered-only, no form logic.
**Sources:** Reference SVG (information density per agent card), issue #157 §7 (hover details)
**Exploration:** quick
**Status:** captured

## D10: Relationship highlighting via CSS classes on selection

**Choice:** When an agent is selected, identify all edges connected to that agent and set `className: 'org-edge-highlighted'` on them. Non-connected edges get `style: { opacity: 0.2 }`. The edge post-processing step (D7) handles this when a selection is active. Clearing selection restores all edges.
**Alternatives:**
- Custom edge components with highlight state — requires React-level state management. Over-engineered for opacity toggling.
**Rationale:** CSS class toggling on existing ReactFlow edge properties. Simple, no infrastructure changes. Visually isolates the selected agent's relationship network.
**Trade-offs:** Requires re-processing edges on selection change. Lightweight — just iterating the edge array.
**Sources:** ReactFlow Edge className/style properties, issue #157 §7 (relationship highlighting)
**Exploration:** quick
**Depends on:** D7
**Status:** captured
