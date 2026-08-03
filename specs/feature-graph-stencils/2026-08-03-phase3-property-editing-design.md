# Phase 3 — Property Editing

**Date:** 2026-08-03
**Issue:** #103 (Epic: Visual Diagram Editor — Domain Layer)
**Status:** Approved
**Parent spec:** `specs/2026-08-01-visual-diagram-editor-design.md` (parent workspace)
**Depends on:** Phase 2 (read-only viewer — completed)

---

## 1. Goal

Click a node in the diagram, edit its properties in a side panel. Property changes update the YAML source of truth via CST-preserving edits and re-render the graph. Includes undo/redo, validation, and complex property editors for oneOf/nested types.

## 2. Layout and Selection

### 2.1 Split layout

`<casehub-diagram>` becomes a split layout: canvas fills remaining space on the left, properties panel (300px fixed width) appears on the right when a node is selected. Hidden when no node is selected.

### 2.2 Selection state flow

1. `graph:node-click` from canvas → casehub-diagram stores `_selectedNodeId`
2. Look up `GraphNode` from the model by ID → pass data + schema to `<casehub-diagram-properties>`
3. Click on canvas background or press Escape → deselect → hide panel
4. `graph:selection-change` with empty selection → deselect

### 2.3 Schema resolution

casehub-diagram stores the full parsed JSON Schema (loaded once from the YAML schema $defs). A `getSchemaForType()` function maps node type to $defs key:

| Node type | Schema $defs key |
|-----------|-----------------|
| `binding` | `Binding` |
| `worker` | `Worker` |
| `milestone` | `Milestone` |
| `goal` | `Goal` |
| `subcase` | `SubCase` |
| `external` | (no schema — read-only display) |

## 3. casehub-diagram-properties Component

Lit element with Shadow DOM enabled (CSS encapsulation from canvas per design spec §3.1).

### 3.1 Properties

- `schema: object` — JSON Schema for the selected node type ($defs entry)
- `data: Record<string, unknown>` — the node's current property values
- `readonly: boolean` — when true, all fields are disabled (for external nodes)

### 3.2 Events

- `property-change` — `{ field: string, value: unknown }` — emitted on each committed field edit (blur or Enter)

### 3.3 Two-tier rendering

| Schema property type | Renders as |
|---------------------|-----------|
| `type: string` | `<input type="text">` |
| `type: string` + `enum` | `<select>` with options |
| `type: integer` / `type: number` | `<input type="number">` with min/max |
| `type: boolean` | `<input type="checkbox">` |
| `type: string` where description contains "JQ" or "expression" | `<textarea rows="3">` |
| `type: object` with known properties | Nested group with collapsible header — recurse one level |
| `type: object` with `additionalProperties` only | Read-only `<pre>` JSON display |
| `type: array` of `type: string` | Comma-separated input (split/join on save) |
| `type: array` of objects | Read-only `<pre>` JSON display |
| `$ref` | Resolve to referenced $def, render as nested group |
| `oneOf` (exclusive types) | Discriminated selector (see §4) |

### 3.4 Field ordering

Properties render in schema property order (Object.keys of schema.properties). Internal properties starting with `_` are hidden. `name` is always first if present.

## 4. Complex Property Editors

### 4.1 Trigger type selector (Binding.on)

Trigger has `oneOf: [contextChange, cloudEvent, schedule, scopeActivated]`. Rendered as:
- Radio button group showing the four trigger types with labels
- Only the active type's sub-form renders below
- Switching types: clears the old trigger data, initializes the new type with empty values, emits property-change for the full `on` object

Sub-forms:
- **contextChange:** filter (textarea), listenLayer (input)
- **cloudEvent:** Short form (string input for type match) or expanded form (type, source, subject, filter). Toggle between forms.
- **schedule:** cron (input) or every (input) — exclusive radio. timezone (input).
- **scopeActivated:** No fields — just the radio selection.

### 4.2 Binding target display

The Binding's oneOf target (capability/subCase/humanTask) is a structural concern — switching target type is Phase 4 (structural editing). Phase 3:
- Shows current target type as a read-only badge at the top of the panel
- Renders the target's properties as editable fields below the badge
- capability: just the capability name (string input)
- subCase: namespace, name, version, completionStrategy, etc. as a nested group
- humanTask: title/titleExpression/templateRef (show active mode), candidateGroups, outcomes, etc.

### 4.3 Nested object groups (outcomePolicy, executionPolicy, cbr)

Collapsible section with a header showing the object name. Inner fields rendered using the same flat-field logic. Supports one level of nesting:
- `outcomePolicy`: onDecline (enum select), onFailure (enum select), onExpired (enum select), maxRerouteAttempts (number)
- `executionPolicy`: timeoutMs (number). retries as a nested sub-group: maxAttempts (number), delayMs (number)
- `cbr`: many flat fields — topK (number), minSimilarity (number), vectorWeight (number), timing (enum), etc. features and weights as read-only JSON.

## 5. YAML Round-Trip

### 5.1 YAML Document management

casehub-diagram stores the YAML as a `yaml.Document` (CST-preserving) alongside the string. The Document is created once on initial YAML load and updated in-place for property edits.

### 5.2 Node path tracking

During `toGraph()`, record the YAML path for each node in a metadata map:
```
yamlPaths: Map<string, (string | number)[]>
```
Key: node ID. Value: path into the YAML document (e.g., `['spec', 'bindings', 2]`).

This is stored separately from node properties — not in GraphNode.properties (which are domain data, not infrastructure metadata).

### 5.3 Edit cycle

1. Properties panel emits `property-change` with `{ field: 'when', value: '.ocrResult != null' }`
2. casehub-diagram looks up `_yamlPaths.get(selectedNodeId)` → `['spec', 'bindings', 2]`
3. Calls `doc.setIn(['spec', 'bindings', 2, 'when'], '.ocrResult != null')`
4. New YAML string = `doc.toString()`
5. Push previous YAML string onto undo stack
6. Re-run `toGraph()` → `toReactFlowGraph()` → `computeElkLayout()` → update canvas
7. Update properties panel data from the new graph model (reflects any derived changes)

### 5.4 applyPropertyEdit utility

```typescript
function applyPropertyEdit(
  doc: yaml.Document,
  nodePath: (string | number)[],
  field: string,
  value: unknown,
): string
```

Handles type coercion: number inputs produce strings from the DOM — coerce back to numbers when the schema says `type: integer` or `type: number`. Boolean checkboxes produce booleans directly.

For nested edits (e.g., `outcomePolicy.onDecline`), the field path is dot-separated: `'outcomePolicy.onDecline'` → `setIn([...nodePath, 'outcomePolicy', 'onDecline'], value)`.

## 6. Undo/Redo

YAML-snapshot model (design spec §2.6):

- `_undoStack: string[]` — previous YAML strings, max depth 50 (oldest dropped)
- `_redoStack: string[]` — cleared on any new edit
- Ctrl+Z: pop undo → push current to redo → re-parse and re-render from popped YAML
- Ctrl+Shift+Z: pop redo → push current to undo → re-parse and re-render
- Stack cleared on external YAML reload (e.g., `yaml` property set from outside)
- Managed by casehub-diagram (composition root)

## 7. Validation

On field blur, validate the field value against the JSON Schema property definition:

| Condition | Message |
|-----------|---------|
| Required field empty | "Required" |
| Number below minimum | "Must be at least {min}" |
| Number above maximum | "Must be at most {max}" |
| String shorter than minLength | "Must be at least {n} characters" |
| String violates pattern | "Invalid format" |

Errors display inline below the field with red text. They do not block editing — advisory only.

## 8. File Structure

```
packages/graph-stencil-case/
  src/
    adapter/
      case-adapter.ts           ← modified: return yamlPaths alongside GraphModel
      yaml-editor.ts            ← applyPropertyEdit utility
      yaml-editor.test.ts

components/casehub-diagram/
  src/
    casehub-diagram.ts          ← modified: split layout, selection, undo/redo, Document management
    casehub-diagram-properties.ts  ← new: property panel component
    casehub-diagram-properties.test.ts
    form/
      field-renderer.ts         ← schema property → form field mapping
      trigger-editor.ts         ← Trigger oneOf editor
      nested-group.ts           ← collapsible nested object group
      validation.ts             ← JSON Schema field validation
      validation.test.ts
```

## 9. Testing Strategy

1. **yaml-editor** — unit tests: setIn a property on a parsed YAML document, verify toString() preserves formatting for untouched sections, verify the changed property has the new value
2. **validation** — unit tests per validation rule (required, min, max, pattern, minLength)
3. **field-renderer** — unit tests: given a schema property, verify the correct field type is returned
4. **trigger-editor** — unit tests: given trigger data, verify correct sub-form renders; verify switching types produces correct property-change event
5. **casehub-diagram-properties** — unit tests: given schema + data, verify correct fields render; verify property-change event on edit
6. **casehub-diagram undo/redo** — unit tests: push edit, undo restores previous, redo re-applies
7. **Integration** — end-to-end: load YAML → select binding → edit `when` field → verify YAML updated → verify graph re-rendered with new value
