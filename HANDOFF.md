# HANDOFF — casehub-mosaic

## Last Session

Completed three XS issues in one session:

- **#334** — Migrated diagram toolbar badge colours from hardcoded hex to `--pages-*` CSS tokens. Replaced `DIAGRAM_TYPE_COLORS` hex map with `DIAGRAM_TYPE_VARIANT` semantic category map (info, accent, warning, success, danger, neutral). Badge colours now adapt to light/dark themes via the pages token system. 3 new tests for variant classes, neutral fallback, and absence of inline styles.

- **#335** — Replaced hardcoded `diagramType: 'choreography'` in `diagram-workbench.ts` with `detectDiagramTypeFromSrc(this.src) ?? detectDiagramType(this.yaml)`. Note: `rootLevel.diagramType` is not currently consumed by any rendering path (the root renders through `<casehub-diagram>` which does its own detection). Fix is for data correctness, not visible behaviour today.

- **#339** — Confirmation card ARIA label now reflects current status: "Confirmation approved/declined/expired" instead of always "Confirmation required". Visual status text uses terse form ("Confirmed"), ARIA uses natural speech ("Confirmation approved").

Filed two new issues from code review:
- **#340** — Remaining hardcoded badge colours in agent-card.ts, shared-styles.ts, decorator-badge.ts, org-legend.ts
- **#341** — Workbench collapsed strip should show diagram type badge

Blog: `2026-11-24-mdp01-three-lies-your-ui-tells.md` — theme of UI honesty gaps at abstraction boundaries.

## Active Item

None — #339 is done and awaiting landing. Queue has items waiting.

## Queue State

Done (awaiting landing): #334, #335, #339
Queued: #340 remaining badge colour migrations, #341 workbench collapsed strip type badge

## Immediate Next Step

Run work end to land the three done items, then work start on #340 (remaining badge colour migrations — M/Low, same domain as #334).

## Cross-Module

- platform#898 — REST endpoints for ServiceConnection and WebAuthn (filed in prior session, not yet implemented)
