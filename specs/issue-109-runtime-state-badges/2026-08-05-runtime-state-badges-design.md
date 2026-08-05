# Runtime State Badges — Design Spec

**Issue:** casehubio/blocks-ui#109
**Date:** 2026-08-05
**Branch:** issue-109-runtime-state-badges

## Problem

The engine defines 6 runtime state enums beyond what blocks-ui currently renders.
The Phase 7 runtime overlay covers TaskStatus (9 states) and MilestoneLifecycleStatus
(3 states). The remaining enums — CaseStatus (7), WorkStatus (8), SlaStatus (3),
OutcomeKind (6), GroupStatus (3), NodeState (6) — have no UI representation.
The work module additionally defines WorkItemStatus (12 states) — a UI-side composite
that includes states like ASSIGNED, IN_PROGRESS, ESCALATED not present in engine WorkStatus.

Additionally, three components (commitment-state-pill, work-item-inbox, session-list)
each implement their own ad-hoc status badge rendering with inline `_statusColors`
records. The diagram overlay in graph-stencil-case uses a fourth variant — hardcoded
`NodeDecoration` maps. All four use the same visual language (`--pages-*` CSS custom
properties, coloured pills) but share no code.

## Approach

A single generic `<status-badge>` component backed by a status registry that maps
`(domain, state)` to visual descriptors. The registry serves two output shapes:
pill rendering (for components and tables) and graph node decorations (for the
diagram overlay). Existing components retrofit to the shared badge.

## Status Model

### Types

In `blocks-ui-core/src/types/status.ts`:

```typescript
export type StateCategory = 'active' | 'info' | 'success' | 'danger'
  | 'neutral' | 'transfer' | 'warning';
// Canonical definition. commitment.ts imports from here.

export interface StatusDescriptor {
  readonly category: StateCategory;
  readonly icon: string;
  readonly label?: string;     // display override; default: the state string
  readonly pulse?: boolean;    // animates active/running states
  readonly border?: boolean;   // sole driver of graph node borders in toDecoration()
}
```

The `domain` parameter on both `registerStatus()` and the `<status-badge>` component
is `string` — the registry is open for future epics (#110, #111) to add domains
without updating a union type. Known domains are documented in the per-domain
registration tables below.

### Registry

A `Map<string, StatusDescriptor>` keyed by `${domain}:${state}`. All built-in
registrations (cross-domain defaults and per-domain entries) are static `Map`
entries populated at module parse time in the same module as the Map itself —
no side-effect imports, no initialization ordering risk. `registerStatus()` is
for external consumers adding new domains at runtime.

Lookup order:

1. `domain:state` — exact match when domain is provided (e.g., `case:WAITING`)
2. `*:state` — cross-domain default (e.g., `*:COMPLETED`). This is the only
   fallback when domain is omitted — per-domain registrations are never scanned
   without an explicit domain.
3. Fallback — `{ category: 'neutral', icon: '?' }`

Extensibility for future epics (#110 conversation states, #111 orchestration states):

```typescript
export function registerStatus(
  domain: string,
  state: string,
  descriptor: StatusDescriptor,
): void;

export function lookupStatus(
  domain: string | undefined,
  state: string,
): StatusDescriptor;
```

### Cross-domain defaults

| State | Category | Icon | Pulse | Border |
|-------|----------|------|-------|--------|
| PENDING | neutral | ○ | | |
| RUNNING | success | ▶ | yes | yes |
| COMPLETED | success | ✓ | | |
| FAULTED | danger | ! | | |
| CANCELLED | neutral | / | | |
| SUSPENDED | warning | ⏸ | | yes |

### Per-domain registrations

**CaseStatus (7 states) — `case:`**

| State | Category | Icon | Notes |
|-------|----------|------|-------|
| STARTING | info | ◐ | Distinct from PENDING — initialising |
| WAITING | warning | ⏳ | Blocked on external input |
| Others (5) | — | — | Use cross-domain defaults |

**TaskStatus (9 states) — `task:`**

| State | Category | Icon | Border |
|-------|----------|------|--------|
| DELEGATED | info | → | yes |
| REJECTED | warning | ✕ | |
| OBSOLETE | neutral | — | |
| Others (6) | — | — | — |

**WorkStatus (8 states) — `work:`**

Maps to engine `WorkStatus` — the execution-side status for orchestrated work.
Distinct from `WorkItemStatus` (UI-side composite). Used when rendering work
execution state in orchestration views; not used by work-item-inbox.

| State | Category | Icon |
|-------|----------|------|
| DECLINED | neutral | 🚫 |
| FAILED | danger | ✗ |
| EXPIRED | warning | ⌛ |
| Others (5) | — | — |

**WorkItemStatus (12 states) — `workitem:`**

Maps to `WorkItemStatus` from `blocks-ui-core/src/types/work-item.ts` — the
UI-side composite status used by work-item-inbox. This is distinct from engine
`WorkStatus` (8 states); the UI enum adds ASSIGNED, IN_PROGRESS, ESCALATED,
OBSOLETE.

| State | Category | Icon | Border |
|-------|----------|------|--------|
| ASSIGNED | info | ● | |
| IN_PROGRESS | active | ◐ | yes |
| DELEGATED | info | → | yes |
| REJECTED | warning | ✕ | |
| ESCALATED | warning | ↑ | |
| OBSOLETE | neutral | — | |
| EXPIRED | warning | ⌛ | |
| Others (5) | — | — | — |

**MilestoneLifecycleStatus (3 states) — `milestone:`**

| State | Category | Icon | Pulse | Border |
|-------|----------|------|-------|--------|
| ACTIVE | info | ◉ | yes | |
| Others (2) | — | — | | |

**OutcomeKind (6 values) — `outcome:`**

| State | Category | Icon |
|-------|----------|------|
| SUCCESS | success | ✓ |
| DECLINED | neutral | 🚫 |
| FAILED | danger | ✗ |
| EXPIRED | warning | ⌛ |
| ESCALATED | warning | ↑ |
| COMPLETED | success | ✓ |

**GroupStatus (3 values) — `group:`**

GroupStatus has no FAULTED state — in this 3-state model, REJECTED is the terminal
failure state and uses `danger` accordingly. This differs from task/workitem domains
where REJECTED (warning) and FAULTED (danger) are separate failure modes.

| State | Category | Icon |
|-------|----------|------|
| IN_PROGRESS | active | ◐ |
| COMPLETED | success | ✓ |
| REJECTED | danger | ✕ |

**SlaStatus (3 values) — `sla:`**

| State | Category | Icon | Pulse |
|-------|----------|------|-------|
| NOT_STARTED | neutral | ○ | |
| ON_TRACK | success | ✓ | |
| BREACHED | danger | ! | yes |

**NodeState (6 variants) — `node:`**

| State | Category | Icon |
|-------|----------|------|
| DISPATCHED | info | → |
| SKIPPED | neutral | ⏭ |
| FAILED | danger | ✗ |
| Others (3) | — | — |

**SessionStatus — `session:`**

| State | Category | Icon |
|-------|----------|------|
| ACTIVE | success | ▶ |
| WAITING | warning | ⏳ |
| IDLE | neutral | ○ |

**Commitment (7 states) — `commitment:`**

Migrated from existing `commitmentStateCategory()`. Same mappings, same icons:

| State | Category | Icon |
|-------|----------|------|
| OPEN | active | ⏳ |
| ACKNOWLEDGED | info | 📋 |
| FULFILLED | success | ✓ |
| FAILED | danger | ✗ |
| DECLINED | neutral | 🚫 |
| DELEGATED | transfer | ↳ |
| EXPIRED | warning | ⌛ |

## Status Badge Component

In `blocks-ui-core/src/status-badge/status-badge.ts`:

```typescript
@customElement('status-badge')
export class StatusBadge extends LitElement {
  @property({ type: String }) state?: string;
  @property({ type: String }) domain?: string;
  @property({ type: String }) size: 'sm' | 'md' = 'sm';
  @property({ type: Boolean }) showIcon = false;
}
```

When `state` is falsy (`undefined`, `null`, `''`), renders `nothing` — no DOM output.
This matches the existing `commitment-state-pill` guard and prevents misleading
fallback badges for absent data.

Renders a coloured pill using `styleMap()` and inline styles (protocol PP-20260713-8ea1af).
Looks up descriptor via `lookupStatus(domain, state)`, maps category to `CategoryStyle`
via the existing `stateCategoryStyles()`.

The retrofit normalises all pill backgrounds to scale-3 CSS custom properties (the
`CATEGORY_STYLES` standard). The existing `_statusColors` in work-item-inbox and
session-list use scale-4. For most states, this is a scale-only change within the
same colour family (e.g. `--pages-neutral-4` → `--pages-neutral-3`) — visually
subtle, slightly lighter backgrounds.

Two workitem states change colour category, not just scale:

- **REJECTED**: danger (red, `--pages-danger-4`) → warning (amber, `--pages-warning-3`).
  REJECTED is a workflow decision ("declined"), not a system failure. FAULTED carries
  the danger semantic. Aligns with task:REJECTED → warning.
- **DELEGATED**: accent (indigo, `--pages-accent-4`) → info (blue, `--pages-info-3`).
  DELEGATED is an informational transfer state, not an active/in-progress state. The
  accent family is reserved for active states (IN_PROGRESS). Aligns with
  task:DELEGATED → info.

Both category changes are intentional semantic corrections — the existing
`_statusColors` miscategorised these states relative to the platform's category
semantics. The remaining 7 workitem states stay within their colour family.

### Usage

```html
<!-- Domain-specific -->
<status-badge domain="case" state="WAITING" showIcon></status-badge>

<!-- Cross-domain default -->
<status-badge state="COMPLETED"></status-badge>

<!-- In a column renderer -->
columnRenderers.set('status', (cell) =>
  html`<status-badge domain="workitem" state=${cell.value} size="sm" showIcon></status-badge>`
);
```

## Diagram Decoration Output

A pure function in `graph-stencil-case/src/runtime/decoration.ts` — NOT in
blocks-ui-core. `NodeDecoration` is defined in `@casehubio/graph-core`, which
blocks-ui-core does not depend on. Placing `toDecoration()` in blocks-ui-core
would couple the generic component library to the graph rendering system.
The conversion from `StatusDescriptor` → `NodeDecoration` is thin and properly
belongs in the graph-aware package.

```typescript
import { lookupStatus } from '@casehubio/blocks-ui-core';
import type { NodeDecoration } from '@casehubio/graph-core';

export function toDecoration(domain: string, state: string): NodeDecoration;
```

This adds `@casehubio/blocks-ui-core` as a dependency of `graph-stencil-case`
(correct direction: specific depends on generic). blocks-ui-core exports
`lookupStatus()` from the registry; graph-stencil-case consumes it.

Maps category to badge colour via a `BADGE_COLORS` record local to
graph-stencil-case — NOT `stateCategoryStyles()`. The graph badge needs
a single vibrant colour as background (with white text overlay), whereas
`stateCategoryStyles()` returns pill-appropriate CSS variable pairs (light
background + dark text at scale-3). The graph renderer renders NodeDecoration
colours as React inline styles on DOM elements (via `stencil-wrapper.tsx`),
so CSS variables would technically resolve, but the scale-3 pill colours are
semantically wrong for the badge context.

```typescript
const BADGE_COLORS: Record<StateCategory, string> = {
  active:   '#6366f1',   // accent-9: vibrant indigo
  info:     '#3b82f6',   // info-9: vibrant blue
  success:  '#22c55e',   // success-9: vibrant green
  danger:   '#ef4444',   // danger-9: vibrant red
  neutral:  '#9ca3af',   // neutral-8: medium grey
  transfer: '#3b82f6',   // same as info
  warning:  '#eab308',   // warning-9: vibrant amber
};
```

These match the existing raw hex values in `MILESTONE_STATUS_DECORATIONS` and
all `TASK_STATUS_DECORATIONS` entries except REJECTED — which changes from
`#f97316` (Tailwind orange-500, ad-hoc) to `#eab308` (amber-500, warning
category standard). This follows from REJECTED's reclassification to the
`warning` category.

Border rendering uses the `border` flag from `StatusDescriptor`. States with
`border: true` get a `border: { style: 'solid', color }` in the decoration.
This is independent of active/terminal classification — PENDING is active
for aggregation purposes but borderless (visually quiet for not-yet-started
work, per Phase 7 §5.3). Similarly, milestone ACTIVE uses pulse but no
border. `pulse` is forwarded independently — it controls animation, not
border presence.

Replaces `TASK_STATUS_DECORATIONS`, `MILESTONE_STATUS_DECORATIONS`, and
`UNKNOWN_DECORATION` in `graph-stencil-case/src/runtime/badge-mappings.ts`.
The runtime-adapter calls `toDecoration('task', status)` instead of indexing
static records.

`TERMINAL_SEVERITY` and `ACTIVE_WORST_PRIORITY` remain in badge-mappings as
aggregation logic — they're about which status "wins" when multiple plan items
exist, not about rendering.

### Case-level overlay

`CaseRuntimeState` in `graph-stencil-case/src/runtime/types.ts` is extended
with `caseStatus?: string` (CaseStatus from the engine: STARTING, RUNNING,
WAITING, SUSPENDED, COMPLETED, FAULTED, CANCELLED). When present, the
diagram toolbar renders `<status-badge domain="case" state=${caseStatus}>`
next to the mode toggle and staleness indicator.

This is NOT a NodeDecoration — the case has no corresponding node in the plan
graph. The badge is rendered directly by `casehub-diagram-toolbar`, reading
`caseStatus` from the `runtimeState` property. `toDecorations()` does not
emit a case entry; the toolbar handles it as a first-class display concern.

## Retrofit Plan

### work-item-inbox

Remove static `_statusColors` record (lines 121–131). Replace inline status
column renderer with `<status-badge domain="workitem">`. The `workitem` domain
maps to `WorkItemStatus` (12 states) — distinct from engine `WorkStatus` (8 states).
Priority badges stay — priority is not a status domain. Remove dead CSS rules
for `.status-pill.status-obsolete`, `.status-pill.status-expired`, and
`.status-pill.status-escalated` (lines 430–443) — these reference an unused
class pattern (the column renderer uses inline styles, not CSS classes).

### session-list

Remove `_statusColors` record (lines 51–55). Replace inline status column
renderer with `<status-badge domain="session">`.

### commitment-state-pill

Keep registered and working. Internally delegates to `<status-badge domain="commitment">`.
Mark as deprecated in JSDoc — new code uses `status-badge` directly.

### graph-stencil-case badge-mappings

Remove `TASK_STATUS_DECORATIONS`, `MILESTONE_STATUS_DECORATIONS`,
`UNKNOWN_DECORATION`. Replace with `toDecoration()` calls from the new
`decoration.ts` within graph-stencil-case itself. Keep `TERMINAL_SEVERITY`
and `ACTIVE_WORST_PRIORITY` as aggregation logic.

### sla-indicator

No retrofit. The full sla-indicator component handles countdown/escalation.
The `sla` domain registration provides a simpler badge for table columns.

### case-explorer

No code change now. The `<status-badge domain="case">` component is available
for column renderers when status columns are added to entity-list.

## What Changes (structural moves)

- `stateCategoryStyles()`, `CategoryStyle`, and `CATEGORY_STYLES` move from
  `blocks-ui-core/src/commitment-pill/styles.ts` to
  `blocks-ui-core/src/styles/category.ts`. These are cross-cutting concerns —
  both `status-badge` and `commitment-state-pill` depend on them, so leaving
  them in commitment-pill would invert the dependency direction (generic
  importing from specific). The `src/styles/` directory already exists
  (holds `animations.ts`). `commitment-state-pill` imports from the new
  location. Old re-export from `commitment-pill/styles.ts` is removed.

- `StateCategory` type — moves from `types/commitment.ts` to `types/status.ts`
  (canonical location for status model types). `commitment.ts` imports from
  `status.ts`.

## What Doesn't Change

- CSS custom property names — all `--pages-*` tokens unchanged.
- `commitment-state-pill` tag — continues to work, deprecated not removed.

## Testing

- Registry: unit tests for lookup order (exact → default → fallback),
  `registerStatus()` override behaviour, all built-in domains resolve.
- Component: rendering tests for each size/domain/showIcon combination.
  Snapshot or string-match on rendered HTML.
- Decoration: unit tests for `toDecoration()` producing valid `NodeDecoration`
  for all registered domains. Verify border-flagged states produce borders.
- Retrofit: existing tests for work-item-inbox, session-list pass unchanged
  (DOM structure equivalent). Update snapshot tests — background colours shift
  from scale-4 to scale-3 as part of the intentional normalisation.
- Backward compat: `commitment-state-pill` renders identically before and after.
