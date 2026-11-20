# Execution Model Badge on Case Diagram

**Issue:** casehubio/casehub-mosaic#295
**Branch:** issue-295-execution-model-badge

## Overview

Add a colour-coded document-type badge to the case diagram toolbar. The type is extracted from the `name.type.yaml` filename convention (primary) with content-based detection as fallback. The type set is open — new types require only a colour map entry.

## Changes

### 1. `detectDiagramTypeFromSrc()` — new function in mosaic-core

**File:** `packages/mosaic-core/src/diagram-type.ts`

```typescript
export function detectDiagramTypeFromSrc(src: string): string | undefined
```

Extracts the type segment from `name.type.yaml` filenames. Takes a URL or path string, extracts the filename, splits on `.`, and returns the second-to-last segment if the file has at least three dot-separated parts ending in `.yaml` or `.yml`.

Examples:
- `mycase.choreography.yaml` → `choreography`
- `plan.htn.yaml` → `htn`
- `bot.agent-conf.yaml` → `agent-conf`
- `legacy.yaml` → `undefined` (no type segment)
- Inline YAML (no src) → `undefined`

Exported from `packages/mosaic-core/src/index.ts`.

### 2. Extend `detectDiagramType()` — mosaic-core

**File:** `packages/mosaic-core/src/diagram-type.ts`

Refine the existing content-based detection. The `case` return value is replaced by more specific types:

| Condition | Returns |
|-----------|---------|
| `spec.decomposition` | `htn` |
| `spec.goals` present | `goap` |
| `spec.bindings` or `spec.workers` | `choreography` |
| `document` + `do` | `swf` |
| None of the above | `unknown` |

The return type remains `string` — not a union literal — because the type set is open. Content detection only covers types with distinguishable YAML structure. Types like `agent-conf`, `avatar-conf`, `agency`, `playbook` are only detectable from filename.

**Breaking change note:** `detectDiagramType` currently returns `'case'` for standard case YAML. Changing this to `'choreography'` affects:
- `DIAGRAM_TAGS` in `packages/mosaic-core/src/diagram-tags.ts` — maps `'case'` → `'casehub-diagram'`. Add `choreography: 'casehub-diagram'` as an alias. Keep `case` for backward compatibility.
- `diagram-workbench` hardcodes `diagramType: 'case'` for the root level. Update to use the resolved type.
- `diagram-export-page` (examples) checks `type === 'case'`. Update to also accept `'choreography'`.
- `goap` needs a `DIAGRAM_TAGS` entry pointing to `'casehub-diagram'` (same component, goal-aware variant).

### 3. Badge property on toolbar

**File:** `components/casehub-diagram/src/casehub-diagram-toolbar.ts`

New property:
```typescript
@property({ type: String }) diagramType = '';
```

Rendering: a styled `<span>` pill on the left side of the toolbar, after the export buttons. Hidden when `diagramType` is empty or `unknown`.

Colour map — a `Record<string, string>` mapping known types to CSS colours using `--pages-*` tokens where available:

| Type | Colour token / fallback |
|------|------------------------|
| choreography | `--pages-accent-color` / blue |
| htn | purple |
| goap | amber |
| llm | teal |
| swf | orange |
| playbook | green |
| agent-conf | slate |
| avatar-conf | slate |
| agency | slate |
| (default) | `--pages-neutral-9` / grey |

Badge style: `font-size: 11px`, `padding: 1px 8px`, `border-radius: 10px`, white text on coloured background, uppercase label.

ARIA: `role="status"`, `aria-label="Document type: {type}"`.

### 4. Wire type through casehub-diagram

**File:** `components/casehub-diagram/src/casehub-diagram.ts`

In the `render()` method, resolve the diagram type and pass to the toolbar:

```typescript
const resolvedType = detectDiagramTypeFromSrc(this.src) ?? detectDiagramType(this._currentYaml);
```

Pass as attribute:
```html
<casehub-diagram-toolbar .diagramType=${resolvedType} ...>
```

The type is resolved on every render — no caching needed since both detection functions are pure and cheap.

### 5. Tests

**`packages/mosaic-core/src/diagram-type.test.ts`:**
- `detectDiagramTypeFromSrc` extracts type from `name.type.yaml` patterns
- `detectDiagramTypeFromSrc` returns `undefined` for single-segment filenames
- `detectDiagramTypeFromSrc` handles URL paths, query strings
- `detectDiagramType` returns `goap` for YAML with `spec.goals`
- `detectDiagramType` returns `choreography` instead of `case` for standard case YAML

**`components/casehub-diagram/src/casehub-diagram-toolbar.test.ts`:**
- Badge renders with correct text and colour for known types
- Badge hidden when `diagramType` is empty or `unknown`
- ARIA attributes present on badge

## Out of scope

- Org-wide enforcement of `name.type.yaml` naming convention — separate cross-repo concern
- Badge click interaction (drill-down, tooltip) — can be added later
- `playbook` or `llm` content-based detection — these need YAML structure conventions to be defined first; filename detection covers them now

## References

- `packages/mosaic-core/src/diagram-type.ts` — existing detection function
- `packages/mosaic-core/src/diagram-tags.ts` — `DIAGRAM_TAGS` map (needs `choreography` and `goap` aliases)
- `components/casehub-diagram/src/casehub-diagram-toolbar.ts` — toolbar component
- `components/casehub-diagram/src/casehub-diagram.ts:39` — existing `detectDiagramType` import
- `components/casehub-diagram/src/casehub-diagram.ts:799-803` — existing usage in drill-down
- `components/diagram-workbench/src/diagram-workbench.ts:114` — hardcoded `diagramType: 'case'`
- `examples/src/pages/diagram-export-page.ts:55` — checks `type === 'case'`
- decisions.md D1-D4 — design decisions captured during brainstorming
