# HANDOFF — casehub-blocks-ui

## Context

Phase 4 of the evolution conductor epic (casehubio/engine#1149). Engine-side work is complete — domain-agnostic SPIs, persistence, wiring, and generic extraction all landed in slot 197 (archived). This slot builds the UI layer in blocks-ui.

## Immediate Next Step

`work start`. Begin with #175 (deny-pattern-editor) — the three editor components (#175-#177) are independent and can be built in any order. The workbench (#178) depends on all three.

## Key Design Decisions

From slot 197 spec (`specs/issue-1148-generalise-evolution-conductor/decisions.md`):

- D1: CapabilityArea IS the HealthSensor — no separate health SPI
- D2: ImprovementCategoryProvider SPI — domains contribute categories with metadata
- D3: ImprovementProposalSource SPI — domains register proposal generators
- D8: Domain-contributed improvement stages — stage IDs are strings, not enums
- D9: blocks-ui gets composable primitives, devtown composes them into the developer workbench
- D10: Devtown is the first consumer — observes its own development pipeline

## Existing Components to Reuse

| Conductor concept | Existing component | Reuse level |
|---|---|---|
| Gate decisions | `approval-gate` | Direct |
| Pending inbox | `notification-inbox`, `work-item-inbox` | High |
| Health scores | `kpi-metric-row` | Direct |
| Compliance levels | `compliance-summary` | Direct |
| Tick history | `event-trail`, `blocks-timeline` | High |
| Improvement streams | `work-item-detail`, `work-item-row` | High |
| Audit trail | `audit-trail-viewer` | Direct |
| Confidence scores | `trust-score-panel` | Partial |

## New Components

1. **deny-pattern-editor** (#175) — static + dynamic deny pattern management
2. **watch-pattern-editor** (#176) — escalation watch pattern CRUD
3. **gate-policy-editor** (#177) — configure GATED/AUTO/NOTIFY per stage

## Workbench

4. **evolution-workbench** (#178) — composes all of the above into a domain-extensible dashboard
5. **Sample page** (#179) — reference view any app can adopt

## Repos in Slot

- **blocks-ui** (primary) — new components and workbench
- **blocks** — domain specialisation
- **engine** — API surface (read-only reference, no changes expected)

## References

- Engine spec: slot 197 workspace `specs/issue-1148-generalise-evolution-conductor/`
- Engine decisions: slot 197 workspace `specs/issue-1148-generalise-evolution-conductor/decisions.md`
- Epic: casehubio/blocks-ui#174
- Parent epic: casehubio/engine#1149
