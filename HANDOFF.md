# HANDOFF — casehub-blocks-ui

## Last Session

Agent setup wizard — manifest editor UX redesign (#215) and avatar-step filter sync bugfix (#216 partial).

### Completed

- **#215 landed** (ee2d8ff) — manifest editor UX redesign:
  - Pipeline step headers (1 Providers, 2 Aliases) with completion badges and dimming
  - Alias resolution preview — live `→ Claude Opus 4.6` or `→ (no match)` per alias row
  - Add-model on built-in provider cards (parity with Other card)
  - Manifest YAML preview with alias resolution comments
  - Data-loaded model merge fix (models not in presets now visible in cards)

- **#214 closed** — auth pattern abstraction was already landed, issue was stale-open

- **#216 partial fix landed** (ff85b4c) — SDI and Big Five now sync to filter pills on archetype selection. But two problems remain (see below).

### In Progress

- **#216 needs full fix** — two remaining problems:
  1. **Incomplete framework sync**: Only SDI + Big Five sync to `_frameworks`/`_bigFive` in `_selectArchetype()`. MBTI, Enneagram, DISC, Belbin profile values need syncing too so their pills show solid/selected state.
  2. **SDI pill visual ambiguity**: All SDI pills have colored borders by default (`sdi-blue`, `sdi-red`, `sdi-green`, `sdi-hub`). Avatar-match border highlighting is invisible against these. Non-matched pills look the same as matched ones. Needs dimming or a different visual signal.
  
  User wants TDD on the complete matrix — test all six frameworks × three visual states (selected, avatar-match, neutral).

## What's Next

1. **#216** — complete the filter pill sync + SDI visual fix (TDD, design iteration in browser)
2. **#211** — agent-catalog (template browsing, filtering, from-scratch creation). All deps met (#167 avatar, #210 manifest editor done).
3. **#212** — agent-profile (character sheet with inline editing)
4. **#213** — agent-relationship-editor (table + arc diagram + ego subgraph)

## Known Issues

- **Pre-existing typecheck errors** — channel-activity, org-diagram, and several other components have `exactOptionalPropertyTypes` strictness issues
- **Pre-existing dirty schema** — `component-schemas.generated.ts` has uncommitted changes from groupedDataView/workItemDetail, not from this session's work

## Cross-Module

- pages#473 — radial-layout migration (CLOSED)
- pages#474 — hold-to-drag pan + layout suppression (layout fix landed, pan fix open)
