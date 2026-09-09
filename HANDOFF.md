# HANDOFF — casehub-blocks-ui

## Current Session

**Branch:** `issue-157-rich-org-diagram` (active, not closed)
**Issue:** casehubio/blocks-ui#157 — Rich org diagram
**Queue:** 1/1 — all plan tasks complete, NOT merged to main yet

### What Was Done

**Session 6 — design + implementation of #157:**

- Brainstormed and captured 10 design decisions (D1-D10) with light decision review and light post-spec review
- Wrote full design spec: `specs/issue-157-rich-org-diagram/2026-09-08-rich-org-diagram-design.md`
- Wrote implementation plan: `plans/2026-09-08-rich-org-diagram.md`
- Implemented all 8 tasks across 4 batches:

**Batch 1 — Adapter enrichment (graph-stencil-org):**
- `types.ts`: Added `AgentDescriptor`, `DispositionAxes`, `OrgAgentNodeData`, `OrgUnitNodeData`
- `adapter/enrichment.ts`: `enrichWithDescriptors(model, agents)` — merges descriptor data into agent node properties
- `adapter/derived-data.ts`: `computeDerivedData(model)` — walks relationship graph for supervision targets, escalation chains (with cycle detection), backup agents, attestation grants

**Batch 2 — Visual helpers (graph-stencil-org):**
- `adapter/kind-colors.ts`: `resolveKindColors(model, kindColors?)` — default palette + override, auto-assigns unknown kinds
- `adapter/node-sizing.ts`: `computeNodeSizes(model)` — pre-computes agent card heights based on populated rows
- `adapter/collapse.ts`: `applyCollapsedUnits(model, yamlPaths, collapsed)` — filters agent nodes + edges for collapsed units
- `adapter/edge-labels.ts`: `applyOrgEdgeLabels(edges)` — sets ReactFlow native label/labelBgStyle per edge type
- `adapter/selection-highlight.ts`: `applySelectionHighlight(edges, selectedNodeId?)` — CSS class on connected edges, opacity on others

**Batch 3 — Rich stencils (graph-stencil-org):**
- `stencils/org-agent.ts`: Multi-row card with header (colored circle + role badge), conditional rows (SLOT, CAPS, DISPOSITION pills, SUPERVISES, ESCALATES, BACKUP, ATTESTATION)
- `stencils/org-unit.ts`: Gradient header by kind color, kind badge, member count, capability pills

**Batch 4 — Component integration (org-diagram):**
- `blocks-org-diagram.ts`: `agents` + `kindColors` properties, full adapter pipeline in `_adaptYaml`, lightweight `_updateEdgeStyles` for selection changes, collapse toggle handler, hover/panel agent click handlers
- `panels/escalation-chain-panel.ts`: Red-tinted panel with chain paths, terminal markers, agent click events
- `panels/attestation-panel.ts`: Purple-tinted panel with dimension pills, signal pills, scope text
- `panels/org-legend.ts`: Full legend — relationships, unit kinds, disposition axes, scope & attestation, eidos model layers
- `panels/tooltip.ts`: Floating hover tooltip with agent properties, 150ms dismiss delay
- `index.ts` + `OrgDiagramProps` in BlocksComponentRegistry + Zod schema regeneration
- `examples/src/pages/org-diagram-page.ts`: Gastown showcase with full agent descriptors

**Test results:** 98 graph-stencil-org tests pass (12 new test files), 11 schema tests pass (registry completeness + staleness). 3 pre-existing test file failures from `@xyflow/react` dist issue.

### CRITICAL: Blocking Issue — Examples Shell Freeze

**The examples app (`examples/src/main.ts`) freezes in the browser.** This blocks manual visual verification of the org diagram.

**Root cause:** `.casehub-packages/packages/graph-renderer/dist/mapping.js` contains unstripped `import type {}` — TypeScript syntax in a `.js` file. esbuild (used by vite's dependency scanner) can't parse it.

**File:** `.casehub-packages/packages/graph-renderer/dist/mapping.js`
**Lines:** Originally lines 1-4 had `import type { ... } from '...'` statements that should have been stripped during the casehub-pages build.

**Why it surfaced now:** This was a latent defect. The #157 changes added new import paths (panel files importing from `@casehubio/graph-stencil-org`, the org-diagram component importing 7 new functions). This widened vite's dependency scan surface, causing it to discover and attempt to parse the broken dist file. Previously, vite's scan avoided this file path.

**Local workaround applied (not committed — in `.casehub-packages` which is gitignored):** Manually stripped the `import type` lines from `mapping.js`. This is fragile — the file comes from a Maven SNAPSHOT artifact and will be overwritten on next `mvn install`.

**Proper fix:** The casehub-pages build for `@casehubio/graph-renderer` must strip TypeScript type imports during compilation. This is likely a tsconfig issue — `isolatedModules: true` or the build tool not running type erasure on `.js` output. The fix belongs in `casehub-pages`, not `blocks-ui`.

**Verification:** The org diagram component DOES work correctly in isolation — a standalone test page (`examples/org-test.html`) rendered successfully via Playwright and a screenshot was captured showing the rich unit container, agent cards, legend, toolbar, and edge routing. The issue is ONLY the examples shell loading all 50+ page modules simultaneously through vite.

### What's Left To Do

1. **Fix the graph-renderer dist issue** (in casehub-pages or local workaround) so the examples shell loads
2. **Manually verify the rich org diagram** in the examples shell — compare with the reference SVG at `casehubio/eidos/docs/diagrams/gastown-org-structure.svg`
3. **Visual QA and iteration** — the Gastown example should show:
   - Purple gradient header on Oversight Chain, blue gradient on Rig Alpha / Rig Beta
   - Rich agent cards with SLOT, DISPOSITION pills (color-coded), SUPERVISES targets, ESCALATES chain (red), BACKUP info, ATTESTATION pills
   - Scope badges on edges (purple pills: "scope: rig-monitoring")
   - Escalation chain panel (red) showing 3 chains
   - Attestation grants panel (purple) showing deacon → witness attestations
   - Legend with all sections
   - Collapsible units (click header to collapse/expand)
   - Hover tooltip (mouseover agent for quick property view)
   - Selection highlighting (click agent to dim non-connected edges)
4. **Fix any visual issues** found during QA
5. **Run `work end`** to close the branch (code review, squash, push)

### Deferred from Issue #157 (documented in spec)

- Draggable agent repositioning within units (separate concern from information density)
- Double-click inline editing (current editing is via property panel + YAML pane)

### Key Files Changed

| Package | Files | What |
|---------|-------|------|
| `packages/graph-stencil-org/src/types.ts` | Modified | +4 interfaces (AgentDescriptor, DispositionAxes, OrgAgentNodeData, OrgUnitNodeData) |
| `packages/graph-stencil-org/src/adapter/` | 7 new files | enrichment, derived-data, kind-colors, node-sizing, collapse, edge-labels, selection-highlight |
| `packages/graph-stencil-org/src/stencils/` | 2 modified, 1 new test | Rich org-agent + org-unit stencils, render.test.ts |
| `packages/graph-stencil-org/src/index.ts` | Modified | Exports all new functions + types |
| `components/org-diagram/src/blocks-org-diagram.ts` | Modified | Pipeline wiring, agents/kindColors props, panels, tooltip |
| `components/org-diagram/src/panels/` | 4 new files | escalation-chain-panel, attestation-panel, org-legend, tooltip |
| `components/org-diagram/src/index.ts` | New | OrgDiagramProps export |
| `components/org-diagram/package.json` | Modified | main → dist/index.js |
| `packages/blocks-ui-schema/src/registry.ts` | Modified | OrgDiagramProps registry entry |
| `examples/src/pages/org-diagram-page.ts` | Modified | Gastown showcase with agents map |

### Commits on Branch (project repo)

```
1bdbd7c fix(org-diagram): prevent render loop and ELK layout hang
2c02a34 feat(examples): add Gastown rich org diagram showcase
c2295d2 feat(org-diagram): add OrgDiagramProps to BlocksComponentRegistry
41b4947 feat(org-diagram): add escalation chain, attestation, legend panels and hover tooltip
229abdb feat(org-diagram): wire decomposed adapter pipeline with agents, kindColors, collapse
11c467f feat(graph-stencil-org): rich agent cards and unit containers with full information density
d6e0950 feat(graph-stencil-org): add edge label and selection highlight post-processors
20e3c2d feat(graph-stencil-org): add color resolution, node sizing, and collapse filtering
240c2f7 feat(graph-stencil-org): add computeDerivedData
792c178 feat(graph-stencil-org): add AgentDescriptor types and enrichWithDescriptors
```
