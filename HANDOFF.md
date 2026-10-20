# HANDOFF — casehub-blocks-ui

## Last Session

Completed #243 (Agent Diff — Coordination Level). Built `coordination-diff-view` — a Lit component that takes two `CoordinationPipeline` objects, computes a structural diff, and renders the graph topology with added/removed/modified overlays. Three new files: `coordination-diff-types.ts`, `coordination-diff-model.ts`, `coordination-diff-view.ts`. Gallery showcase with KYC Pipeline example (pep-screener added, quorum 2/3→3/4, risk-assessor params updated). 136 tests passing.

During review of the gallery's fabricated demo data, identified that the end-to-end YAML project story has gaps. Filed issues across repos — see "End-to-End YAML Project Context" below.

Branch `issue-243-agent-diff-coordination` is merged to main, pushed, issue closed, branch stamped.

## Immediate Next Step

Work through three blocks-ui issues in order:

1. **#262** — Trim `PatternSpec` (12 of 17 fields fabricated, no YAML source). Decide which fields to keep (derivable from case structure) vs remove.
2. **#263** — Clean up `CoordinationStep` UI-only fields (`outputFormat`, `drillDown`, `icon`, `channel`). Source from YAML, keep as optional, or remove.
3. **#258** — Build the projection layer: YAML → UI types (`AgentDescriptor`, `CoordinationPipeline`, `PatternSpec`). Data-source-agnostic — pure transform functions from parsed YAML objects to UI types. The gallery switches from inline JSON to real YAML fixtures via the projection.

## End-to-End YAML Project Context

**Goal:** A local project view that reads YAML files from a git clone on disk and renders them as agent coordination visualizations. No REST endpoints — the UI reads files directly. The projection logic is data-source-agnostic (takes parsed YAML in, produces UI types out; where the YAML comes from is a separate concern).

**The pipeline:** Author YAML → Parser reads files → Projection functions → UI components render

**What exists today:**
- Engine case definition YAML (`dsl/spec/workers/bindings/capabilities`) — real format, real examples in scaffold
- Eidos agent descriptor YAML — rich format with weighted disposition axes, capabilities, goals, constraints, briefing (27 of 31 `AgentDescriptor` fields are sourced)
- blocks-ui components render the types correctly — the rendering layer is done
- Slot 216 landed `issue-1238-auto-bootstrap-yaml` — classpath scanning infra (server-side, not needed for local project view but validates the YAML formats)

**What's missing (tracked by issues):**
- **eidos#206** — `AgentDescriptor.description` field not in eidos YAML (XS)
- **eidos#207** — `mbtiType`/`enneagramType` not in descriptor (XS, needs design decision)
- **engine#1257** — `quorum` coordination primitive not in engine DSL (S)
- **blocks-ui#262** — `PatternSpec` has 12 fabricated fields (M, trim or design engine pattern DSL)
- **blocks-ui#263** — `CoordinationStep` UI-only fields (XS)
- **blocks-ui#258** — Projection layer + gallery real data (M)

**Key insight from this session:** The previous field-level audit against stale slot repos was badly wrong (reported 13 NO SOURCE fields for AgentDescriptor). The canonical eidos repo already has weighted disposition axes, goals, constraints, capabilities with qualityHint. Always rebase from canonical before auditing. The real gaps are much smaller than initially assessed — mostly `PatternSpec` (engine doesn't model multi-agent patterns explicitly) and `quorum` (no DSL primitive).

## .plan State

The `.plan` in this workspace is stale — it's from the previous epic session (#240/#241) and was corrupted by lifecycle transitions during work-end. The next session should scaffold a fresh `.plan` via `work start` for whichever issue it picks up.
