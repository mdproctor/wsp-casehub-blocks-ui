# HANDOFF — casehub-blocks-ui

## Last Session

Agent setup wizard — #216 filter pill sync fix + #211 agent-catalog component.

### Completed

- **#216 closed** (37c5418) — all six framework filter pills now sync on archetype selection. MBTI, Enneagram, DISC, Belbin added to `_selectArchetype()`. SDI non-matching pills dimmed when archetype selected. 8 tests.

- **#211 closed** (fb6afda landed on main) — agent-catalog component:
  - Evolved `FullAgentDescriptor` in blocks-ui-core with personality, manifest, preferredAlias, profession, role fields
  - Extracted `initProfile` from avatar-step into shared `profile-derivation.ts`
  - Template data derived from PROFESSION_PRESETS (120+ templates, 5 featured)
  - Grouped-by-role layout with profession pill filters, fuzzy search, featured picks
  - Inline detail expansion with personality badges, alias tier, summary text
  - `catalog:template:selected` event with FullAgentDescriptor
  - 16 component tests + 5 data tests + 3 profile derivation tests
  - Demo harness at `components/agent-catalog/demo.html`

- **#217 created** — agent-personality-workbench: tabbed catalog + wizard with shared profile panel. The catalog and avatar wizard are two views of the same personality state.

### Design Decisions

- Catalog templates reuse PROFESSION_PRESETS (no separate taxonomy)
- Alias-driven manifest defaults per task type (reasoning-heavy vs fast-response)
- "From scratch" removed from catalog — belongs at host level as mode toggle
- Duplicate `PersonalityProfile` type in blocks-ui-core and avatar-step (cleanup tracked)

## What's Next

1. **#217** — agent-personality-workbench (catalog + wizard tabs, shared profile)
2. **#212** — agent-profile (character sheet with inline editing)
3. **#213** — agent-relationship-editor (table + arc diagram + ego subgraph)

## Known Issues

- **Pre-existing typecheck errors** — channel-activity, org-diagram, trust-workbench, blocks-dag-viewer have strictness issues
- **Duplicate PersonalityProfile type** — defined in both blocks-ui-core/types/agent.ts and avatar-step/avatar-step.ts (structurally identical, will drift)

## Cross-Module

- pages#474 — hold-to-drag pan fix (open)
