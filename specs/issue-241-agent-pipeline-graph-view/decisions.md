# Decisions — #241 Agent Pipeline Graph View

## D1: Toggle location

**Choice:** Inside existing coordination-pipeline-view component
**Alternatives:**
- Wrapper component — adds indirection without benefit, same data shape
- Standalone component — loses toggle, consumers must compose manually
**Rationale:** The graph is an alternate rendering of the same CoordinationPipeline data. A viewMode property and header toggle keeps the API simple.
**Trade-offs:** Component grows in size (two render paths) but avoids cross-component coordination.
**Sources:** coordination-pipeline-view.ts, coordination-types.ts, mockup section 14
**Exploration:** quick
**Status:** captured

## D2: Layout strategy

**Choice:** Data-driven tiering — analyze PhaseItem[] to infer tiers
**Alternatives:**
- Fixed mockup template — hard-codes the 4-tier KYC layout, only works for one pipeline shape
**Rationale:** PhaseItem[] already encodes the tier structure implicitly (steps, parallel blocks, barriers). Walking the array assigns tiers. This adapts to different pipeline shapes.
**Trade-offs:** Slightly more complex tier-assignment logic vs hard-coding, but the sequential nature of PhaseItem[] makes it straightforward.
**Sources:** coordination-types.ts (PhaseItem union type)
**Exploration:** quick
**Status:** captured

## D3: Auto-switch default view

**Choice:** Auto-detect with manual override
**Alternatives:**
- Always default to lanes — conservative, but graph view may go undiscovered
**Rationale:** If pipeline has fan-out/fan-in (parallel block + quorum), graph view is more informative. If purely sequential with named swim lanes, lanes view is better. Toggle allows manual override either way.
**Trade-offs:** Adds shape-detection logic, but it's a simple check for parallel + quorum presence in the phases array.
**Sources:** Issue #241 auto-switch rule, mockup section 14
**Exploration:** quick
**Status:** captured

## D4: Rendering strategy

**Choice:** Pure CSS tiers with CSS connectors — no SVG
**Alternatives:**
- HTML + SVG overlay — SVG arrows between tiers. Fragile: needs getBoundingClientRect, ResizeObserver, manual coordinate math. Misaligns on resize.
- ELK layout + HTML/SVG — robust layout engine but heavy dependency, overkill for sequential data
- Full graph infrastructure (pages-graph-canvas + ELK + stencils) — designed for interactive editing, not read-only visualization
**Rationale:** PhaseItem[] is always sequential — no skip connections, cycles, or arbitrary edges. Fan-out/fan-in is structurally "step → parallel block → quorum → step." CSS flexbox handles this naturally: each tier is a flex row, connectors (chevrons, dotted lines) between rows. Edge labels as inline badges.
**Trade-offs:** Less visual fidelity than SVG arrows for the fan-out/fan-in lines in the mockup, but much simpler, naturally responsive, and animatable via CSS transitions.
**Sources:** coordination-types.ts (PhaseItem sequential structure), mockup section 14, coordination-pipeline-view.ts (existing CSS patterns)
**Exploration:** deep-analysis
**Status:** captured
