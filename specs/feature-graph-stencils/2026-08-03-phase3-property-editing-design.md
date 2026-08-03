# Phase 3 — Property Editing

**Date:** 2026-08-03
**Issue:** #103 (Epic: Visual Diagram Editor — Domain Layer)
**Status:** Approved (revised after light design review)
**Parent spec:** `specs/2026-08-01-visual-diagram-editor-design.md` (parent workspace)
**Depends on:** Phase 2 (read-only viewer — completed)

---

## 1. Goal

Click a node in the diagram, edit its properties in a side panel. Property changes update the YAML source of truth via the CaseAdapter (CST-preserving edits) and re-render the graph. Includes undo/redo, validation, and complex property editors for oneOf/nested types.

## 2. Layout and Selection

### 2.1 Split layout

`<casehub-diagram>` becomes a split layout: canvas fills remaining space on the left, properties panel (300px fixed width) appears on the right when a node is selected. Hidden when no node is selected.

### 2.2 Selection state flow

1. `graph:node-click` from canvas → casehub-diagram stores `_selectedNodeId`
2. Look up `GraphNode` from the model by ID → pass data + schema to `<casehub-diagram-properties>`
3. Click on canvas background or press Escape → deselect → hide panel
4. `graph:selection-change` with empty selection → deselect
5. On external YAML reload (`yaml` property set from outside) → clear selection if the selected node no longer exists in the new model

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

The schema $defs are the single source for field-renderer, validation, and stencil property definitions.

## 3. casehub-diagram-properties Component

Lit element with Shadow DOM enabled (CSS encapsulation from canvas per design spec §3.1).

### 3.1 Properties

- `schema: object` — JSON Schema for the selected node type ($defs entry)
- `data: Record<string, unknown>` — the node's current property values
- `readonly: boolean` — when true, all fields are disabled (for external nodes)

### 3.2 Events

- `property-change` — `{ field: string[], value: unknown }` — emitted with `composed: true, bubbles: true` on each committed field edit (blur or Enter). `field` is an array path (e.g., `['when']` or `['outcomePolicy', 'onDecline']`) — never dot-separated strings.

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
| `type: array` of `type: string` | Newline-separated `<textarea>` (split on `\n`, join on save) |
| `type: array` of objects | Read-only `<pre>` JSON display |
| `$ref` | Resolve to referenced $def, render as nested group |
| `oneOf` (exclusive types) | Discriminated selector (see §4) |

### 3.4 Field ordering

Properties render in schema property order (Object.keys of schema.properties). Internal properties starting with `_` are hidden. `name` is always first if present.

### 3.5 Empty value handling

Clearing a field (empty string in input, unchecked checkbox):
- Optional property → remove the key from the YAML (delete, not set to empty string)
- Required property → keep the empty value and show validation error

## 4. Complex Property Editors

### 4.1 Trigger type selector (Binding.on)

Trigger has `oneOf: [contextChange, cloudEvent, schedule, scopeActivated]`. Rendered as:
- Radio button group showing the four trigger types with labels
- Only the active type's sub-form renders below
- Switching types: emits a single `property-change` for the full `on` object with the new trigger type structure. The adapter handles clearing the old trigger data in the YAML.

Sub-forms:
- **contextChange:** filter (textarea), listenLayer (input)
- **cloudEvent:** If value is a string → simple string input. If value is an object → expanded form (type, source, subject, filter). No toggle — determined by current YAML value type.
- **schedule:** cron (input) or every (input) — exclusive radio. timezone (input).
- **scopeActivated:** No fields — just the radio selection.

### 4.2 Binding target display

The Binding's oneOf target (capability/subCase/humanTask) is a structural concern — switching target type is Phase 4 (structural editing). Phase 3:
- Shows current target type as a read-only badge at the top of the panel
- Renders the target's properties as editable fields below the badge
- capability: just the capability name (string input)
- subCase: namespace, name, version, completionStrategy, etc. as a nested group
- humanTask: shows active mode (title/titleExpression/templateRef) as read-only badge, renders that mode's fields as editable. Switching between modes is Phase 4.

### 4.3 Nested object groups (outcomePolicy, executionPolicy, cbr)

Collapsible section with a header showing the object name. Inner fields rendered using the same flat-field logic. Supports one level of nesting:
- `outcomePolicy`: onDecline (enum select), onFailure (enum select), onExpired (enum select), maxRerouteAttempts (number)
- `executionPolicy`: timeoutMs (number). retries as a nested sub-group: maxAttempts (number), delayMs (number)
- `cbr`: many flat fields — topK (number), minSimilarity (number), vectorWeight (number), timing (enum), etc. features and weights as read-only JSON.

## 5. YAML Round-Trip

### 5.1 Adapter owns YAML editing

All YAML mutations go through `CaseAdapter`, not directly from the diagram component. The adapter owns YAML Document creation, path tracking, and CST-preserving edits. casehub-diagram never touches `yaml.Document` directly.

### 5.2 Adapter API extension

`toGraph()` return type extended to include path metadata:

```typescript
interface AdapterResult {
  model: GraphModel;
  yamlPaths: ReadonlyMap<string, readonly (string | number)[]>;
}

function toGraph(yaml: string): AdapterResult
```

New editing function:

```typescript
function applyPropertyEdit(
  yaml: string,
  nodePath: readonly (string | number)[],
  field: readonly (string | number)[],
  value: unknown,
): string
```

Parses the YAML into a Document internally, applies `doc.setIn([...nodePath, ...field], value)`, and returns the new YAML string via `doc.toString()`. CST-preserving — untouched sections keep original formatting.

Handles type coercion: number inputs produce strings from the DOM — coerce back to numbers when the schema says `type: integer` or `type: number`. Boolean checkboxes produce booleans directly. Empty optional fields → delete the key from the YAML node.

### 5.3 Edit cycle

1. Properties panel emits `property-change` with `{ field: ['when'], value: '.ocrResult != null' }`
2. casehub-diagram looks up `yamlPaths.get(selectedNodeId)` → `['spec', 'bindings', 2]`
3. Calls `applyPropertyEdit(currentYaml, ['spec', 'bindings', 2], ['when'], '.ocrResult != null')` → new YAML string
4. Push previous YAML string onto undo stack
5. Re-run `toGraph()` → `toReactFlowGraph()` → update canvas node data **without re-layout** (property edits don't add/remove nodes — reuse existing positions)
6. Update properties panel data from the new graph model
7. If `toGraph()` throws (invalid YAML after edit) → revert to previous YAML, show error toast, do not update undo stack

### 5.4 No re-layout on property edits

Property edits change node data, not graph topology. Skip `computeElkLayout()` on property edits — reuse the existing node positions. Only re-layout when nodes are added/removed (Phase 4).

This also eliminates the async race condition: without the async layout call, the edit cycle is synchronous (`applyPropertyEdit` → `toGraph` → `toReactFlowGraph` → update). Rapid edits can't overlap.

## 6. Undo/Redo

YAML-snapshot model (design spec §2.6):

- `_undoStack: string[]` — previous YAML strings, max depth 50 (oldest dropped)
- `_redoStack: string[]` — cleared on any new edit
- Ctrl+Z: pop undo → push current to redo → re-parse via `toGraph()` and re-render (no re-layout)
- Ctrl+Shift+Z: pop redo → push current to undo → re-parse via `toGraph()` and re-render (no re-layout)
- Stack cleared on external YAML reload (`yaml` property set from outside)
- On undo/redo, if the selected node still exists in the new model → keep selection and update panel. If it doesn't → clear selection.
- Managed by casehub-diagram (composition root)

Note: `applyPropertyEdit()` creates a fresh `yaml.Document` internally each call — no persistent Document state to get out of sync on undo.

## 7. Validation

On field blur, validate the field value against the JSON Schema property definition:

| Condition | Message |
|-----------|---------|
| Required field empty | "Required" |
| Number below minimum | "Must be at least {min}" |
| Number above maximum | "Must be at most {max}" |
| String shorter than minLength | "Must be at least {n} characters" |
| String violates pattern | "Invalid format" |

Errors display inline below the field with red text. They do not block editing — advisory only. Required fields show an asterisk (*) label indicator.

## 8. File Structure

```
packages/graph-stencil-case/
  src/
    adapter/
      case-adapter.ts           ← modified: toGraph returns AdapterResult with yamlPaths
      yaml-editor.ts            ← applyPropertyEdit utility
      yaml-editor.test.ts

components/casehub-diagram/
  src/
    casehub-diagram.ts          ← modified: split layout, selection, undo/redo
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

1. **yaml-editor** — unit tests: applyPropertyEdit on a parsed YAML, verify CST preservation for untouched sections, verify the changed property has the new value, verify type coercion, verify empty optional field deletion
2. **validation** — unit tests per validation rule (required, min, max, pattern, minLength)
3. **field-renderer** — unit tests: given a schema property, verify the correct field type is returned
4. **trigger-editor** — unit tests: given trigger data, verify correct sub-form renders; verify switching types produces correct property-change event with composed:true
5. **casehub-diagram-properties** — unit tests: given schema + data, verify correct fields render; verify property-change event on edit crosses Shadow DOM
6. **casehub-diagram undo/redo** — unit tests: push edit, undo restores previous, redo re-applies, selection cleared if node removed
7. **Integration** — end-to-end: load YAML → select binding → edit `when` field → verify YAML updated → verify graph re-rendered with new value without re-layout

## 10. Review Findings Addressed

| Finding | Resolution |
|---------|-----------|
| Adapter bypass (Structure-R1-01, Cross-R1-01/02) | §5.1: All YAML mutations through CaseAdapter. applyPropertyEdit in adapter, not diagram. |
| Async race (Robustness-R1-02) | §5.4: Skip re-layout on property edits — edit cycle is synchronous. |
| Shadow DOM event crossing (Robustness-R1-05) | §3.2: composed:true, bubbles:true on property-change. |
| Undo Document sync (Robustness-R1-04, Cross-R1-03) | §6: applyPropertyEdit creates fresh Document per call. No persistent Document state. |
| toGraph signature (Robustness-R1-03) | §5.2: Returns AdapterResult { model, yamlPaths }. |
| Dot-separated paths (Robustness-R1-08) | §3.2, §5.2: Array paths throughout. |
| Comma-separated arrays (Coherence-R1-08) | §3.3: Newline-separated textarea instead. |
| Selection on YAML reload (Robustness-R1-06) | §2.2 step 5, §6: Clear selection if node gone. |
| Re-layout skip (Coherence-R1-06) | §5.4: Property edits reuse existing positions. |
| Required field indicators (Coherence-R1-09) | §7: Asterisk label indicator. |
| cloudEvent toggle (Coherence-R1-10) | §4.1: Determined by current value type, no toggle. |
| Empty value handling (Robustness-R1-10) | §3.5: Optional → delete key, required → keep + validate. |
| humanTask mode (Robustness-R1-11) | §4.2: Active mode as read-only badge, switching is Phase 4. |
| Error recovery (Robustness-R1-07, Cross-R1-06) | §5.3 step 7: Revert on toGraph failure, show error. |
