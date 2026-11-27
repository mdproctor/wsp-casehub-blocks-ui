# HANDOFF — casehub-mosaic

## Last Session

Housekeeping session: closed stale issues, fixed queue inconsistencies, implemented two S/Low issues.

**Landed (from prior sessions):**
- **#341** — Workbench collapsed strip type dot. Extracted `DIAGRAM_TYPE_VARIANT` to mosaic-core. Filed casehub-pages#746 for variant badge component.

**Implemented this session (on stacked branches, awaiting landing):**
- **#333** — `ConversationTurn` discriminated union. Split flat interface into `MessageTurn | ToolCallTurn | ConfirmationTurn`. Removed non-null assertions from transcript renderer. On branch `issue-298-worker-chain-tracing`.
- **#340** — Migrated remaining badge colours to pages CSS tokens: agent-card type badges, shared-styles pattern badge fallback, model badge, org-legend disposition/attestation pills. On branch `issue-340-remaining-badge-colours` (stacked on #333's branch).

**Closed stale issues:** #332, #334, #335, #339 (verified landed on main, were left open).

**Filed:**
- **#342** — Remaining non-badge hardcoded colours (text, borders, tabs) across agent-viewer and org-legend. M/Med.
- **casehub-pages#746** — Variant badge component (from prior session, still open).

**Queue fix:** #298, #333, #340 were erroneously marked `[x]` done in the queue with no implementation. Unchecked and re-queued.

**close.py investigation:** Diagnosed a bug where stacked branches fail to land when the parent was squashed before landing (stale base record). Fix already in soredium and synced.

## Active Item

#340 is done on branch `issue-340-remaining-badge-colours`.

## Queue State

Done (awaiting landing): #333, #340
Open: #298 (Worker chain tracing — L, deferred to fresh session)
Filed: #342 (non-badge colour migration — M/Med)

## Immediate Next Step

Run `work end` to land #333 and #340. Then `work start #298` in a fresh session (L-scale, needs brainstorming).

## Cross-Module

- casehub-pages#746 — Variant badge component (blocks shared CSS extraction)
- platform#898 — REST endpoints for ServiceConnection and WebAuthn (prior session, not yet implemented)
