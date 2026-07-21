# HANDOFF — casehub-blocks-ui

**Branch:** issue-33-subscription-editor (closed)
**Date:** 2026-07-21
**Issues:** casehubio/blocks-ui#33, casehubio/blocks-ui#34

## What landed

Schema-driven subscription editor (#33) and notification preferences UI (#34), plus GDPR erasure form migration — all using `pages-schema-form`.

**pages-form improvements (cross-repo, casehub-pages):**
- `oneOf` labeled enums with disabled placeholder in create mode
- `format: 'time'` for HH:mm inputs
- `readOnly` field support (validation skip)
- Recursive validation for nested objects and array items

**New components in notification-inbox:**
- `subscription-editor` — dynamic event-type field rebuild for constraint/template fields
- `channel-preferences` — per-channel delivery mode toggle (immediate/digest), digest schedule, groupBy, quiet hours with action
- `mute-list` — pages-table + inline schema-form with scope-conditional entityType
- `snooze-control` — two-state toggle with date-time picker
- `notification-preferences` — container composing all three

**Migrated:**
- `gdpr-erasure-action` — hand-coded form → schema-form (−48 lines)

**Frontend type alignment:**
- `DigestScheduleWeeklyAt`, `ENTITY_WATCHERS` target, `DigestGroupBy`, `QuietHoursAction`

**Example page:**
- Preferences button + dialog with full mock API routes (channels, preferences PATCH, mutes CRUD)
- Column overlap fix: unread dot merged into title, AGE column widened
- `relativeTime` shows weeks/months/years instead of dates

**Tests:** 177 passing across pages-form (65), notification-inbox (101), gdpr-erasure-action (11)

## What's left

- Replace portal `@casehubio/pages-form` links with published version refs before release · XS · Low
- Update `docs/repos/casehub-blocks-ui.md` in parent repo with new component descriptions · S · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Publish `@casehubio/pages-form` to GitHub Packages | XS | Low | Blocks blocks-ui release |
| — | SSE for preferences/mute/snooze | S | Med | Deferred — fetch-on-mount sufficient |
| — | Drag-and-drop ordering for constraints/targets | S | Med | Deferred — add/remove is functional |
