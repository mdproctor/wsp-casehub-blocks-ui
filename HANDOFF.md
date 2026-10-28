# HANDOFF — casehub-mosaic

## Last Session (2026-10-28)

Model alignment sweep — traced mosaic TS types back to engine/agency/eidos Java records. Four issues handled, two with code changes, two closed as invalid or superseded.

1. **#263** — Removed 3 UI-only fields from `CoordinationStep` (`drillDown`, `icon`, `iconColor`). Retained 3 that have engine sources (`outputFormat`, `channel`, `channelCount`). Removed fields now derived at render time: drillability from `action === 'agent-delegate'`, match-case icons/colors from `PATTERN_STYLES` map keyed by pattern name.
2. **#262** — Investigation revealed all 17 PatternSpec fields exist in agency's `PatternSpec.java` sealed interface. Closed as invalid. Filed #294 for the real gap: missing `PatternView` API projection.
3. **#257** — Already superseded by #283/#284 (both closed). Closed.
4. **#256** — Aligned `AgentDescriptor` TS to eidos Java records (16→30 fields). Upgraded `HealthStatus` from 3-string union to 7-state `CapabilityStatus` discriminated union matching `CapabilityHealth.CapabilityStatus` sealed hierarchy. Health dots now render with status-derived tooltips.

## Immediate Next Step

Continue through the agent-viewer/playbook issue queue. Recommended order: #291 (consume eidos personality-vocab.json — S/Med), then #235 (org-diagram degraded mode fix — S/Med), then #258 (gallery demo data from real YAML — M/Med).

## Known Issues

- **3 stale findings** in `.audit/findings.jsonl` from branch `issue-233-org-relationship-editor-diagram` — pre-existing, not blocking
- **Pre-existing test failures** — 5 in deliberation views (voting/debate threshold rendering)
- **Clinical repo** — extensive stale `blocks-` references

## Cross-Module

- pages#474 — hold-to-drag pan fix (open)
- pages#530 — Milkdown editor + `pages-document-diff` extraction
- agency#368 — CognitionDefinition REST endpoint
- #294 — PatternView API projection (filed this session, blocks real YAML data in gallery)

## References

- Blog: `blog/2026-10-27-mdp02-coordination-step-field-sourcing.md`
- Blog: `blog/2026-10-28-mdp01-eidos-type-alignment.md`
