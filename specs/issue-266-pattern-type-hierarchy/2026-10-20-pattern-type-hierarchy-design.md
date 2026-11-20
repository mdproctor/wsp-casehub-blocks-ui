# Design: Restructure PatternType into Hierarchical Agency Categories

**Issue:** casehubio/blocks-ui#266
**Branch:** issue-266-pattern-type-hierarchy
**Date:** 2026-10-20
**Scope:** Cross-repo (pages + blocks-ui)

## Problem

`PatternType` is a flat 8-member union (`sequence | parallel | supervisor | loop | conditional | voting | debate | htn`) mixing playbook orchestration with agentic coordination. Two parallel definitions exist with different casing (lowercase in agent-viewer, UPPERCASE in blocks-ui-core). `PatternSpec` is a monolithic 16-field bag where every field is optional on every pattern type.

The agency repo restructure (casehubio/blocks#337) organises Java implementations into 7 categories: orchestration, deliberation, perception, communication, governance, optimisation, social. The UI type system should mirror this hierarchy.

## Design

### 1. Pages — PlaybookPatternCategory

A new export from `@casehubio/yaml-core/patterns` formalizes existing step kinds as a pattern category.

```typescript
export type PlaybookPatternType =
  | 'sequence' | 'parallel' | 'conditional'
  | 'match' | 'loop' | 'try-catch';

export interface PlaybookPatternCategory {
  category: 'playbook';
  type: PlaybookPatternType;
  name?: string;
  description?: string;
}
```

Mapping from existing yaml-core step kinds:
- `block` (sequential step list) → `sequence`
- `parallel` → `parallel`
- `if-else` → `conditional`
- `match` → `match`
- `loop` decorator → `loop`
- `try-catch-finally` → `try-catch`

This is a category marker for dispatch, not a step definition. Step-level details remain in `ResolvedStep`.

**New file:** `packages/yaml-core/src/patterns.ts`
**New subpath export:** `"./patterns"` in `package.json`

### 2. blocks-ui-core — Agency Category Hierarchy

New directory `packages/blocks-ui-core/src/types/patterns/` with one file per category plus an index.

#### Category unions

```typescript
export type AgencyCategoryName =
  | 'orchestration' | 'deliberation' | 'perception'
  | 'communication' | 'governance' | 'optimisation' | 'social';

export type PatternCategoryName = 'playbook' | AgencyCategoryName;
```

#### Detailed spec interfaces (categories with active/imminent UI consumers)

**OrchestrationPatternCategory:**

```typescript
export type OrchestrationPatternType =
  'supervisor' | 'choreography' | 'saga' | 'decomposition';

export interface OrchestrationPatternCategory {
  category: 'orchestration';
  type: OrchestrationPatternType;
  name?: string;
  description?: string;
  agents?: AgentRef[];
  routing?: { type: string; guard?: string };
  decomposition?: { strategy: string };
  aggregation?: AggregationSpec;
  termination?: TerminationSpec[];
  failurePolicy?: FailurePolicy;
}
```

**DeliberationPatternCategory:**

```typescript
export type DeliberationPatternType =
  'voting' | 'debate' | 'negotiation' | 'coalition' | 'normative';

export interface DeliberationPatternCategory {
  category: 'deliberation';
  type: DeliberationPatternType;
  name?: string;
  description?: string;
  agents?: AgentRef[];
  judgment?: JudgmentSpec;
  maxRounds?: number;
  judge?: AgentRef;
  aggregation?: AggregationSpec;
  termination?: TerminationSpec[];
  failurePolicy?: FailurePolicy;
}
```

#### Minimal base spec (categories without UI consumers yet)

```typescript
export interface BasePatternCategory {
  category: AgencyCategoryName;
  type: string;
  name?: string;
  description?: string;
  agents?: AgentRef[];
  failurePolicy?: FailurePolicy;
}

export interface PerceptionPatternCategory extends BasePatternCategory {
  category: 'perception';
  type: 'summarisation' | 'narrative' | 'observation';
}

export interface CommunicationPatternCategory extends BasePatternCategory {
  category: 'communication';
  type: 'channel-execution' | 'channel-summary';
}

export interface GovernancePatternCategory extends BasePatternCategory {
  category: 'governance';
  type: 'judgment' | 'oversight' | 'attestation' | 'trust';
}

export interface OptimisationPatternCategory extends BasePatternCategory {
  category: 'optimisation';
  type: 'prompt-optimisation';
}

export interface SocialPatternCategory extends BasePatternCategory {
  category: 'social';
  type: 'memory-hygiene' | 'cognition' | 'reflection';
}
```

Per-category specs replace `BasePatternCategory` when views land.

#### Aggregate union

```typescript
import type { PlaybookPatternCategory } from '@casehubio/yaml-core/patterns';

export type PatternCategory =
  | PlaybookPatternCategory
  | OrchestrationPatternCategory
  | DeliberationPatternCategory
  | PerceptionPatternCategory
  | CommunicationPatternCategory
  | GovernancePatternCategory
  | OptimisationPatternCategory
  | SocialPatternCategory;
```

### 3. Type Deduplication

blocks-ui-core becomes the single source of truth for pattern types.

| File | Action |
|------|--------|
| `blocks-ui-core/src/types/orchestration.ts` | Remove flat `PatternType` (8 UPPERCASE members). `ExecutionModel.pattern` changes to `PatternCategoryName`. Canonical `AgentRef` stays here (keeps `readonly id`). All other orchestration types (`ExecutionState`, events, etc.) unchanged. |
| `agent-viewer/src/pattern-types.ts` | Delete. All types re-exported from blocks-ui-core. `PatternSpec` replaced by `PatternCategory` union. |
| `agent-viewer/src/index.ts` | Remove `pattern-types.js` exports. Re-export new types from `@casehubio/blocks-ui-core`. |

**AgentRef reconciliation:** Canonical definition in blocks-ui-core keeps `readonly id?: string` (optional — not all contexts have an id). `AgentRefType` unified to lowercase: `'worker' | 'human' | 'external' | 'composed' | 'channel'`.

**Supporting types retained in blocks-ui-core:** `AggregationSpec`, `TerminationSpec`, `JudgmentSpec`, `JudgmentTrigger`, `JudgmentCaller`, `FailurePolicy` — moved from agent-viewer's pattern-types.ts to blocks-ui-core's types directory if not already there. Structural differences reconciled (blocks-ui-core's `FailurePolicy` has more specific types than agent-viewer's string-based version — keep the more specific version).

### 4. SSE Normalization

A normalization function at the EventStreamController deserialization boundary maps UPPERCASE Java enum values to lowercase.

```typescript
export function normalizePatternType(wire: string): PatternCategoryName {
  return wire.toLowerCase() as PatternCategoryName;
}

export function normalizeExecutionSnapshot(raw: RawExecutionSnapshot): ExecutionSnapshot {
  return {
    ...raw,
    model: {
      ...raw.model,
      pattern: normalizePatternType(raw.model.pattern),
    },
  };
}
```

Applied once at SSE parse time. All downstream rendering code works with lowercase.

Similarly normalizes `AgentRefType`, `ExecutionState`, `AgentResultStatus`, and other UPPERCASE enum values from the wire format.

### 5. Agent-Viewer Dispatch Refactor

Two-level dispatch in `agent-viewer.ts`:

```typescript
private renderPattern(): TemplateResult {
  const p = this.data.pattern;
  if (!p) return html`<div class="empty-state">No pattern data available</div>`;

  switch (p.category) {
    case 'orchestration':
      return this.renderOrchestrationPattern(p);
    case 'deliberation':
      return this.renderDeliberationPattern(p);
    case 'playbook':
    case 'perception':
    case 'communication':
    case 'governance':
    case 'optimisation':
    case 'social':
      return html`<div class="empty-state">
        ${p.category}/${p.type} view not yet available</div>`;
  }
}
```

**Component property changes:**
- `voting-view.pattern`: `PatternSpec` → `DeliberationPatternCategory`
- `debate-view.pattern`: `PatternSpec` → `DeliberationPatternCategory`
- `AgentViewerData.pattern`: `PatternSpec` → `PatternCategory`

Component internals largely unchanged — the fields they use are present on `DeliberationPatternCategory`.

**Barrel exports:** `@casehubio/agent-viewer` stops exporting `PatternType`, `PatternSpec`, and supporting types. These come from `@casehubio/blocks-ui-core` instead. Breaking change for external consumers.

### 6. Test and Gallery Updates

- Test fixtures gain `category` field, use per-category types instead of `PatternSpec`
- Gallery YAML data updated to include `category` discriminant
- `orchestration.test.ts` updated from UPPERCASE to lowercase assertions
- `execution-monitor.ts` updated to use lowercase comparisons (SSE normalization handles the conversion)
- New type-assignment tests for each category interface

## Files Changed

### Pages
- **New:** `packages/yaml-core/src/patterns.ts` — PlaybookPatternType, PlaybookPatternCategory
- **Modified:** `packages/yaml-core/package.json` — add `"./patterns"` subpath export

### blocks-ui
- **New:** `packages/blocks-ui-core/src/types/patterns/` directory:
  - `index.ts` — barrel exports + aggregate union
  - `categories.ts` — category name unions
  - `orchestration.ts` — OrchestrationPatternCategory
  - `deliberation.ts` — DeliberationPatternCategory
  - `base.ts` — BasePatternCategory + minimal category interfaces
  - `normalize.ts` — SSE normalization functions
- **Modified:** `packages/blocks-ui-core/src/types/orchestration.ts` — remove flat PatternType, update ExecutionModel.pattern to PatternCategoryName, reconcile AgentRef
- **Modified:** `packages/blocks-ui-core/src/types/index.ts` — add patterns re-export
- **Deleted:** `packages/agent-viewer/src/pattern-types.ts`
- **Modified:** `packages/agent-viewer/src/index.ts` — re-export from blocks-ui-core
- **Modified:** `packages/agent-viewer/src/agent-viewer.ts` — two-level dispatch
- **Modified:** `packages/agent-viewer/src/voting-view.ts` — property type change
- **Modified:** `packages/agent-viewer/src/debate-view.ts` — property type change
- **Modified:** `components/execution-monitor/src/execution-monitor.ts` — lowercase comparisons
- **Modified:** Test files — fixture updates, new type tests

## Breaking Changes

- `PatternSpec` removed — replaced by `PatternCategory` discriminated union
- `PatternType` (flat 8-member union) removed — replaced by per-category type unions
- Types now exported from `@casehubio/blocks-ui-core` instead of `@casehubio/agent-viewer`
- UPPERCASE enum values in orchestration types replaced with lowercase
- `AgentRef` structural change (optional `id` field added)

## References

- casehubio/blocks-ui#266 — this issue
- casehubio/blocks#337 — agency repo 7-category restructure (source of category taxonomy)
- casehubio/blocks-ui#267 — supervisor-view (first orchestration consumer)
- casehubio/blocks-ui#268 — judgment-view (governance consumer)
- casehubio/blocks-ui#269 — negotiation-view (deliberation consumer)
- `packages/agent-viewer/src/pattern-types.ts` — current lowercase PatternType + PatternSpec
- `packages/blocks-ui-core/src/types/orchestration.ts` — current UPPERCASE PatternType + ExecutionModel
- `packages/agent-viewer/src/agent-viewer.ts:262-270` — current flat dispatch
- `packages/yaml-core/src/step/walker.ts:65-77` — ResolvedStep union (pages step kinds)
- R1-01 through R1-08 — decision review findings
