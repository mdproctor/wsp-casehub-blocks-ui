# HANDOFF — casehub-mosaic

## Last Session

Completed #332 (confirmation auto-expiry timer). Added a 1-second interval check in `casehub-transcript` that transitions pending confirmations to `'expired'` when their `expiresAt` deadline passes. Dispatches `avatar:confirmation-expired` event. Timer cleans up on disconnect. 7 tests covering expiry, non-expiry, event dispatch, idempotency, no-deadline, cleanup, and multi-expiry. Code review found one pre-existing issue — ARIA label doesn't reflect confirmation status changes — filed as #339.

Revised blog `2026-11-20-mdp01-the-avatar-gets-discipline.md` to replace the "What's left unbuilt" section with the auto-expiry design story — timer lives in the transcript not the session layer.

## Active Item

#334 — Badge colours should use pages CSS tokens. Not started.

## Queue State

Active: #334 (not started)
Queued: #335 workbench dynamic type, #339 confirmation ARIA label

## Immediate Next Step

Start #334. Check `components/casehub-diagram/` for hardcoded badge colour values and migrate to `--pages-*` CSS custom properties.

## Cross-Module

- platform#898 — REST endpoints for ServiceConnection and WebAuthn (filed in prior session, not yet implemented)

## References

- Blog: `blog/2026-11-20-mdp01-the-avatar-gets-discipline.md` (revised this session)
- Filed issues: #339 (confirmation ARIA label)
