# HANDOFF — casehub-blocks-ui

## Context

Evolution conductor UI (casehubio/engine#1149, Phase 4). All initial components landed: three configuration editors (#175-#177), workbench shell (#178), sample page (#179). Follow-up epic #188 covers operational completeness.

## What Was Built

| Issue | Component | Status |
|-------|-----------|--------|
| #175 | `deny-pattern-editor` — static + dynamic deny pattern CRUD | Done |
| #176 | `watch-pattern-editor` — escalation watch pattern CRUD | Done |
| #177 | `gate-policy-editor` — per-stage GATED/AUTO/NOTIFY config | Done |
| #178 | `evolution-workbench` — summary bar + tabbed dashboard | Done |
| #179 | Sample page with trading risk panel domain extension demo | Done |
| — | `detail-pane` standalone mode enhancement | Done |

Two new packages: `evolution-config/` (shared types/API + 3 editors) and `evolution-workbench/`. 85 tests passing.

## Engine Issues Filed

| Issue | What | Why |
|-------|------|-----|
| engine#1186 | `getWatchPatterns` | watch-pattern-editor endpoint mode |
| engine#1187 | `getStages`/`getCategories` | gate-policy-editor stage metadata |
| engine#1188 | `getGatePolicy` | gate-policy-editor pre-population |

## What's Next

**Follow-up epic: casehubio/blocks-ui#188 — Evolution conductor UI operational completeness**

| Priority | Issue | Scale | Complexity | Blocked by | Notes |
|----------|-------|-------|------------|------------|-------|
| 1 | engine#1186 | XS | Low | — | Unblocks watch-pattern endpoint mode |
| 1 | engine#1187 | XS | Low | — | Unblocks gate-policy endpoint mode |
| 1 | engine#1188 | XS | Low | — | Unblocks gate-policy pre-population |
| 2 | #186 | M | Med | — | Streams tab — pages-table with block/unblock |
| 2 | #187 | M | Med | — | Inbox tab — pages-table with approve/reject |
| 3 | #181 | S | Med | engine#1186 | Deny-pattern preview against active streams |
| 3 | #182 | S | Med | #186 | Gate-policy impact preview |
| 4 | #183 | S | Low | — | Audit trail tab (reuses audit-trail-viewer) |
| 4 | #184 | S | Low | — | Health detail tab (reuses trust-score-panel) |
| 5 | #185 | S | Low | — | Responsive layout |

Engine issues are quick wins (thin adapter methods). Start there, then Streams + Inbox tabs to make the workbench operational.

## Repos in Slot

- **blocks-ui** (primary) — new components and workbench
- **blocks** — domain specialisation
- **engine** — API surface (3 issues filed for endpoint gaps)
