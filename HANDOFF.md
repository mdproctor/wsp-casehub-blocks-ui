# Handoff — Slot 201: Agent Setup Wizard

## Orientation

You are in slot 201, branch `issue-166-agent-setup-wizard`, working on
blocks-ui#166 — an epic for four independent web components that make
LLM and agent setup trivial.

**This work has completed full design.** The spec, decisions, and
implementation plan are all written, reviewed, and ready to execute.
You should NOT brainstorm or redesign — go straight to implementation.

## What to build

Four standalone LitElement web components + one package:

| Component | Tag | What it does |
|-----------|-----|-------------|
| **Manifest editor** | `<agent-manifest-editor>` | LLM provider/model/credential config with presets, auto-detection, guided setup |
| **Agent catalog** | `<agent-catalog>` | Template browsing with pill filters, popular section, from-scratch wizard |
| **Agent profile** | `<agent-profile>` | Rich "character sheet" with disposition radar, capabilities, inline editing |
| **Relationship editor** | `<agent-relationship-editor>` | Table-based relationship management with arc diagram and ego diagram views |
| **Avatar 2D** | `<agent-avatar>` (package) | DiceBear Avataaars generation from disposition axes |

Plus type additions to `packages/blocks-ui-core/`.

## Key files to read first

Read these in order — they give you everything you need:

1. **Implementation plan** (execute this):
   `wksp/plans/2026-09-20-agent-setup-wizard.md`
   — 7 batches, 13 tasks, TDD steps, exact code, exact file paths

2. **Design spec** (reference when implementing):
   `wksp/specs/agent-setup-wizard/2026-09-20-agent-setup-wizard-design.md`
   — full component specs, data contracts, type mirrors, test strategy

3. **Design decisions** (reference when making judgment calls):
   `wksp/specs/agent-setup-wizard/decisions.md`
   — 8 decisions + 2 constraints, reviewed

## Architecture decisions to know

- **No DataSourceMixin.** These components consume object trees, not
  tabular data. They extend LitElement directly with `endpoint` and
  `data` properties. Follow the `gdpr-erasure-action` pattern.

- **Disposition model simplified.** Java `AgentDisposition` supports
  multi-term weighted values per axis. The wizard deliberately
  simplifies to single-term for ergonomics, with "advanced" toggle for
  power users. Avatar uses `primaryTerm()`. See constraint C4 in spec.

- **Type mirrors from Java.** The TS types in blocks-ui-core mirror
  actual Java records from `casehub-platform-agent-config-core` and
  `casehub-eidos-api`. They were verified against decompiled bytecode.
  `Manifest.providers` and `.models` are arrays, NOT maps.

- **Credential backends.** Three types matching `CredentialRef` sealed
  interface: `env:VAR_NAME`, `file:/path`, `ref:name`. NOT "direct
  encrypted" or "vault URI."

- **2D avatars complement 3D.** `packages/avatar/` has a 3D TalkingHead
  system for live interaction. The new `agent-avatar-2d` package is for
  static profile icons. They don't replace each other.

- **Ego diagram is experimental.** The relationship editor's ego diagram
  view reuses the existing graph-stencil-org renderer. It needs an
  `extractEgoSubgraph()` function (Task 12). If ELK layout is poor for
  star topologies, the view gets replaced later.

## Execution approach

Use `executing-plans` or `subagent-driven-development` to work through
the plan. Each batch is a safe wrap point. The recommended flow:

1. Run `work` — it will detect the scaffolded state and resume
2. Read the implementation plan
3. Execute task by task, following TDD (write test → verify fail →
   implement → verify pass → commit)
4. Each batch ends with all tests passing and a clean build

**Batches 1-2 must be sequential** (everything depends on core types
and avatar). **Batches 3-6 are independent** — manifest editor, catalog,
profile, and relationship editor can be built in any order.

## Existing patterns to follow

- **Component scaffold:** see `components/service-card/` for package.json,
  tsconfig, test patterns
- **Non-DataSourceMixin component:** see `components/gdpr-erasure-action/`
- **Event topics:** see `packages/blocks-ui-core/src/types/events.ts`
  (colon-delimited, `emitPagesEvent`)
- **Type files:** see `packages/blocks-ui-core/src/types/` (one file per
  domain, barrel re-export from index.ts)
- **ARIA tests:** see `components/commitment-viz/` for `aria-label`
  assertion patterns
- **Org agent stencil:** see `packages/graph-stencil-org/src/stencils/org-agent.ts`
  for disposition pill rendering (match visual style)

## Cross-repo context (read-only — do NOT commit to these)

These repos are at their canonical paths for reference:
- `/Users/mdproctor/claude/casehub/eidos` — AgentDescriptor, profiles
- `/Users/mdproctor/claude/casehub/platform` — Manifest, ManifestLoader
- `/Users/mdproctor/claude/casehub/neocortex` — memory seeding reference
- `/Users/mdproctor/claude/casehub/examples/wacky-manor` (in slot 196)
  — briefing→memory transition pattern

## Out of scope

Do NOT implement these — they are follow-ups:
- Composition shell (combining the 4 components)
- Full neocortex seeding API
- Resource configuration (calendar, RAG, mailbox)
- 3D avatar changes
- Custom ego diagram renderer (only if ELK experiment fails)

## Progress

No implementation work has been done. Starting from scratch on the
feature branch. The branch is clean — based on main.
