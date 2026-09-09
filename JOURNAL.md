# Design Journal — issue-157-rich-org-diagram

## 2026-09-08 — Design and implementation

### Decisions

10 design decisions captured (D1-D10). Key architectural choices:
- **Adapter decomposition** (D1, D3): `toOrgGraph` stays pure YAML→GraphModel. Separate `enrichWithDescriptors` and `computeDerivedData` functions chain after it. Decision review validated this separation — the original plan to merge into `toOrgGraph` would have violated the protocol's "definition data parsed from YAML" semantics.
- **Panels as internal modules** (D4): Escalation chain, attestation, and legend panels start as internal Lit elements in `org-diagram/src/panels/`, not standalone packages. Follows the promotion pipeline — promote when a second consumer appears.
- **Edge labels via ReactFlow native properties** (D7): Post-process edges with `applyOrgEdgeLabels` to set `label`, `labelBgStyle`, etc. No custom edge components or cross-repo changes needed.

### Implementation

All 8 plan tasks completed across 4 batches:
1. **Adapter enrichment**: Types, enrichWithDescriptors, computeDerivedData — 9 new test files, 98 tests passing
2. **Visual helpers**: resolveKindColors, computeNodeSizes, applyCollapsedUnits, applyOrgEdgeLabels, applySelectionHighlight
3. **Rich stencils**: Multi-row agent cards with disposition pills, escalation chains, backup, attestation. Gradient unit headers with capability pills.
4. **Integration**: Component pipeline wiring, internal panels, hover tooltip, OrgDiagramProps registry entry

### Blocking issue discovered

The examples shell freezes due to a **pre-existing build defect** in `.casehub-packages/packages/graph-renderer/dist/mapping.js` — the dist `.js` file contains unstripped `import type {}` (TypeScript syntax). esbuild/vite's dependency scanner can't parse this, causing the dependency optimization to fail silently. The browser never finishes loading.

This was latent before — vite's scanner avoided the broken file. My changes widened the import surface area (new panel files, new adapter function imports), causing vite to scan more of the dependency graph and hit the broken file.

Local workaround: manually strip `import type` lines from the dist file. Proper fix: the casehub-pages build for graph-renderer must strip type imports during compilation (likely a tsconfig `isolatedModules` or build tool issue).
