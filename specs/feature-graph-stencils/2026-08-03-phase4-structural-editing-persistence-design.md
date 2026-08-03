# Phase 4 — Structural Editing + Persistence

**Date:** 2026-08-03
**Issue:** #103 (Epic: Visual Diagram Editor — Domain Layer)
**Status:** Approved (revised after light design review)
**Parent spec:** `specs/2026-08-01-visual-diagram-editor-design.md` (parent workspace)
**Depends on:** Phase 3 (property editing — completed)

---

## 1. Goal

Add, remove, and restructure case definition elements via the diagram editor. Save and load YAML via pluggable persistence backends, starting with GitHub. Includes a palette for adding nodes, delete with dependency checks, binding target type switching (deferred from Phase 3), a toolbar with save/dirty indicator, and conflict resolution.

## 2. Architectural Principle: YAML-First Mutations

All structural mutations go through YAML, not the GraphModel. This is consistent with Phase 3's property editing — `applyPropertyEdit` mutates YAML, then `toGraph()` re-derives the graph.

graph-core's `addNode/removeNode/replaceNode` operate on `GraphModel`, but we never persist GraphModel — we persist YAML. The structural editing path is:

```
User action → YAML mutation (CST-preserving) → toGraph() → toReactFlowGraph() → computeElkLayout() → render
```

Unlike property edits (which skip re-layout), structural edits change graph topology and MUST trigger full re-layout via `computeElkLayout()`.

## 3. YAML Structural Operations

All new functions live in `yaml-editor.ts` alongside `applyPropertyEdit`. The adapter file (`case-adapter.ts`) remains read-only (YAML→graph direction).

### 3.1 addElement

```typescript
function addElement(
  yaml: string,
  elementType: 'binding' | 'worker' | 'milestone' | 'goal',
  defaults?: Record<string, unknown>,
): string
```

Appends a new element to the appropriate YAML array. Uses `parseDocument()` for CST preservation. Creates the array if it doesn't exist in the YAML.

**Array mapping:**

| Element type | YAML path |
|-------------|-----------|
| binding | `spec.bindings` |
| worker | `spec.workers` |
| milestone | `spec.milestones` |
| goal | `spec.goals` |

**Default properties per type** (minimum valid structure):

| Type | Generated defaults |
|------|-------------------|
| Binding | `{ name: 'binding-N', capability: '' }` |
| Worker | `{ name: 'worker-N', capabilities: [] }` |
| Milestone | `{ name: 'milestone-N' }` |
| Goal | `{ name: 'goal-N', kind: 'success' }` |

Name uniqueness: scans existing elements of that type, generates `{type}-{N}` where N is the first unused suffix. The `defaults` parameter merges over generated defaults.

SubCase is NOT directly addable — it's created as a binding target via `switchBindingTarget`.

### 3.2 removeElement

```typescript
function removeElement(
  yaml: string,
  nodePath: readonly (string | number)[],
): string
```

Removes the element at `nodePath` (e.g., `['spec', 'bindings', 2]`) from its parent YAML array. The `nodePath` comes from `AdapterResult.yamlPaths`. CST-preserving.

No edge cleanup needed — edges are derived by `toGraph()` on re-parse. Removing a Worker whose capabilities were referenced by bindings means those bindings produce `external:` nodes on the next `toGraph()` call (expected behavior — external workers are normal in CaseHub).

### 3.3 switchBindingTarget

```typescript
function switchBindingTarget(
  yaml: string,
  bindingPath: readonly (string | number)[],
  targetType: 'capability' | 'subCase' | 'humanTask',
): string
```

Clears all three target fields (`capability`, `subCase`, `humanTask`) from the binding, then sets the new target with defaults:

| Target type | Default value |
|------------|---------------|
| capability | `''` (empty string — user fills in via property panel) |
| subCase | `{ namespace: '', name: '' }` |
| humanTask | `{ title: '' }` |

Single undo unit. Topology-changing operation — triggers full re-layout (switching from capability to subCase may create/remove SubCase nodes and change edges).

## 4. Palette Component

`<casehub-diagram-palette>` — Lit element with Shadow DOM enabled.

### 4.1 Properties

- `disabled: boolean` — disables all add buttons (e.g., during readonly mode)

### 4.2 Events

- `palette-add` — `{ elementType: 'binding' | 'worker' | 'milestone' | 'goal' }` — `composed: true, bubbles: true`

### 4.3 Visual

Vertical list of four items, each showing:
- A small shape icon matching the stencil visual (rounded rect for binding, rect for worker, diamond for milestone, hexagon for goal)
- The type label below the icon

Click to add. No drag-and-drop (deferred per parent spec). Fixed width (56px — icon-only sidebar). Hidden when `disabled` or readonly mode.

SubCase is not listed — created as a binding target.

## 5. Binding Target Type Switching

Phase 3 renders the binding target type as a read-only badge at the top of the properties panel (spec §4.2). Phase 4 makes it editable.

### 5.1 Properties panel change

The read-only badge becomes a `<select>` dropdown with three options: Capability, SubCase, HumanTask. The current target type is pre-selected.

### 5.2 Event

`target-type-change` — `{ targetType: 'capability' | 'subCase' | 'humanTask' }` — `composed: true, bubbles: true`

### 5.3 casehub-diagram handling

1. Look up `yamlPaths.get(selectedNodeId)` → binding's YAML path
2. Call `switchBindingTarget(currentYaml, bindingPath, newTargetType)` → new YAML
3. Push previous YAML to undo stack
4. Full re-layout (topology change)
5. Update properties panel with new data from the re-parsed model

### 5.4 Scope boundary

HumanTask mode switching (title vs titleExpression vs templateRef) remains read-only — that's a deeper structural concern deferred beyond Phase 4.

## 6. Delete with Dependency Checks

Delete key or Backspace while a node is selected triggers removal. The handler MUST check `e.target` — skip deletion if the active element is an `<input>`, `<textarea>`, or `[contenteditable]` to avoid conflicting with text editing in the properties panel.

### 6.1 Connected nodes (has edges in graph model)

Show confirmation via `blocks-confirm-dialog` before removing:

| Node type | Warning message |
|-----------|----------------|
| Worker with inbound edges | "Worker '{name}' has {N} binding(s) dispatching to its capabilities. Those bindings will reference external capabilities after removal." |
| Binding with outbound edge | "Binding '{name}' connects to {target}. The connection will be removed." |
| Milestone/Goal with edges | "Remove {type} '{name}'? It has {N} connection(s)." |

### 6.2 Unconnected nodes

Remove immediately with no confirmation. Undo available via Ctrl+Z.

### 6.3 Non-deletable nodes

`external:` nodes are synthetic — created by the adapter for unresolvable capability references. They are not deletable. They disappear automatically when the referencing binding is removed or its capability resolved.

### 6.4 After removal

Clear selection, push previous YAML to undo stack, full re-layout.

## 7. casehub-diagram Layout Integration

### 7.1 Layout change

The layout expands from the current two-pane to a full editor layout:

```
┌──────────────────────────────────────────────────────┐
│  toolbar (save, dirty indicator)                      │
├──────┬───────────────────────────────┬───────────────┤
│      │                               │               │
│  P   │                               │  properties   │
│  A   │        canvas                 │  panel        │
│  L   │                               │  (on select)  │
│  E   │                               │               │
│  T   │                               │               │
│  T   │                               │               │
│  E   │                               │               │
│      │                               │               │
├──────┴───────────────────────────────┴───────────────┤
```

- **Toolbar**: Fixed height strip at top. Always visible.
- **Palette**: Fixed width (56px). Left side. Hidden when readonly.
- **Canvas**: Fills remaining space. Same as current.
- **Properties panel**: 300px right side. Appears on node selection. Same as current plus target type selector.

### 7.2 New event handlers

| Event | Source | Action |
|-------|--------|--------|
| `palette-add` | palette | `addElement()` → undo push → full re-layout |
| `target-type-change` | properties panel | `switchBindingTarget()` → undo push → full re-layout |
| `toolbar-save` | toolbar | `backend.write()` → handle result |
| Delete/Backspace | keyboard | Dependency check → confirm if needed → `removeElement()` → undo push → clear selection → full re-layout |
| Ctrl+S | keyboard | Same as `toolbar-save` |

### 7.3 Structural edits and undo

Structural edits follow the same undo/redo pattern as property edits:
- Push previous YAML string to undo stack before mutation
- Undo restores the previous YAML and triggers full re-layout
- Redo stack cleared on any new edit
- Max depth 50 (same as property edits)

The only difference from property edits is that structural edits call `_fullRender()` (with re-layout) instead of `_updateWithoutLayout()`.

**Undo/redo always uses `_fullRender()`:** Since undo/redo can restore YAML from either a property edit or a structural edit, and the undo stack doesn't track which type of edit produced each entry, all undo/redo operations use `_fullRender()` for correctness. The performance cost of occasional unnecessary re-layout on property-edit undo is negligible compared to the incorrectness of skipping re-layout on structural-edit undo.

### 7.4 Async render guard

`_fullRender()` is async (ELK layout). Rapid structural edits (e.g., clicking palette twice quickly) can cause overlapping renders where the second completes with stale intermediate state.

Guard: `_renderInProgress: boolean` flag. While true, new structural edits still mutate `_currentYaml` and push to the undo stack (YAML mutations are synchronous), but `_fullRender()` is deferred. When the in-flight render completes, if `_currentYaml` has changed since the render started, trigger another `_fullRender()` with the current YAML. This ensures the final rendered state always matches `_currentYaml`.

The guard also prevents save while a render is in progress — the save flow checks `_renderInProgress` and defers until render completes.

## 8. Persistence

### 8.1 PersistenceBackend SPI (already exists in graph-core)

```typescript
interface PersistenceBackend {
  read(uri: string): Promise<ReadResult>;
  write(uri: string, yaml: string, expectedVersion: string): Promise<WriteResult>;
}
```

`InMemoryBackend` already implemented in graph-core.

### 8.2 GitHubBackend

Lives in `packages/graph-stencil-case/src/persistence/github-backend.ts`. Domain-agnostic implementation — promotable to graph-core (pages) when mature.

```typescript
interface GitHubBackendConfig {
  token: string;
  owner: string;
  repo: string;
  branch?: string;        // default: 'main'
  commitMessage?: string;  // default: 'Update case definition'
}

class GitHubBackend implements PersistenceBackend {
  constructor(config: GitHubBackendConfig);
  read(uri: string): Promise<ReadResult>;
  write(uri: string, yaml: string, expectedVersion: string): Promise<WriteResult>;
}
```

The `uri` parameter is the file path within the repo (e.g., `cases/document-processing.yaml`).

**read():**
1. `GET /repos/{owner}/{repo}/contents/{uri}?ref={branch}` with `Authorization: Bearer {token}`
2. Decode base64 `content` field
3. Return `{ status: 'ok', yaml: decoded, version: response.sha }`
4. On 404 → `{ status: 'not_found', uri }`

**write():**
1. `PUT /repos/{owner}/{repo}/contents/{uri}` with `{ message, content: base64(yaml), sha: expectedVersion, branch }`
2. On 200/201 → `{ status: 'ok', version: response.content.sha }`
3. On 409 → `GET` current file to fetch SHA → `{ status: 'conflict', currentVersion: currentSha }`
4. For new files (no `expectedVersion`) → PUT without `sha` field

### 8.3 casehub-diagram persistence integration

**New properties:**

| Property | Type | Description |
|----------|------|-------------|
| `backend` | `PersistenceBackend \| null` | Optional. Null = playground mode (no save). |
| `uri` | `string` | Document path for the backend |

**New internal state:**

| State | Type | Description |
|-------|------|-------------|
| `_version` | `string` | Last version from backend (optimistic concurrency) |
| `_savedYaml` | `string` | YAML at last save/load point |
| `_saving` | `boolean` | True while a save is in flight |

**Load flow** (on `backend` + `uri` both set):
1. If `_saving` is true, ignore (don't load while a save is in flight)
2. `try { backend.read(uri) }` → handle `ReadResult`. On fetch/network error: show error toast, keep current state.
3. `ok` → set `_currentYaml`, `_savedYaml`, `_version`, clear undo/redo, full render
4. `not_found` → start with empty case definition template, set `_version = ''`
5. `parse_error` → show error
6. `schema_error` → load anyway (warnings advisory), store version

**Save flow** (Ctrl+S or toolbar Save):
1. If no backend, not dirty, `_saving` is true, or `_renderInProgress` is true → no-op
2. Set `_saving = true`, update toolbar
3. `try { backend.write(uri, _currentYaml, _version) }` → handle `WriteResult`. On fetch/network error: show error toast, set `_saving = false`.
4. `ok` → update `_version`, set `_savedYaml = _currentYaml`, set `_saving = false`
5. `conflict` → set `_saving = false`, show conflict dialog

**New file creation:** When `_version` is `''` (from a `not_found` load or a brand new document), the GitHubBackend interprets `expectedVersion = ''` as "create new file" and omits the `sha` field from the PUT request body.

**Dirty tracking:** Dirty state is derived: `_currentYaml !== _savedYaml`. This correctly handles undo past a save point — if the user saves, then undoes back to the pre-save state, dirty becomes true again because the current YAML differs from what was last saved. No explicit `_dirty` flag needed.

### 8.4 Conflict resolution dialog

On `write()` returning `conflict`, show a conflict resolution dialog. `blocks-confirm-dialog` supports confirm/cancel (two actions). Conflict resolution needs three actions, so use a dedicated `<casehub-diagram-conflict-dialog>` inline element (not a new file — a private render method within casehub-diagram) with three buttons:

| Action | Label | Behavior |
|--------|-------|----------|
| Overwrite | "Save anyway" | `write(uri, _currentYaml, conflict.currentVersion)` |
| Reload | "Discard my changes" | `read(uri)` → re-render, clear undo/redo |
| Cancel | "Keep editing" | Dismiss, continue editing without saving |

### 8.5 Toolbar component

`<casehub-diagram-toolbar>` — Lit element with Shadow DOM.

**Properties:**
- `dirty: boolean` — shows unsaved indicator (dot next to Save). Computed by casehub-diagram as `_currentYaml !== _savedYaml`.
- `saving: boolean` — shows spinner while save in progress. Set by casehub-diagram's `_saving` state.
- `hasBackend: boolean` — when false, Save button hidden (playground mode)

**Events:**
- `toolbar-save` — `composed: true, bubbles: true`

## 9. File Structure

```
packages/graph-stencil-case/
  src/
    adapter/
      yaml-editor.ts                ← modified: addElement, removeElement, switchBindingTarget
      yaml-editor.test.ts           ← modified: tests for new functions
    persistence/
      github-backend.ts             ← NEW
      github-backend.test.ts        ← NEW
    index.ts                        ← modified: export new functions + GitHubBackend

components/casehub-diagram/
  src/
    casehub-diagram.ts              ← modified: layout, palette/toolbar/persistence/delete
    casehub-diagram-palette.ts      ← NEW
    casehub-diagram-palette.test.ts ← NEW
    casehub-diagram-toolbar.ts      ← NEW
    casehub-diagram-toolbar.test.ts ← NEW
    casehub-diagram-properties.ts   ← modified: binding target type selector
```

## 10. Testing Strategy

1. **yaml-editor — addElement**: Add each element type, verify element appears in correct array with required defaults, verify CST preservation for untouched sections, verify name uniqueness when duplicates exist
2. **yaml-editor — removeElement**: Remove by path, verify element gone, verify remaining elements intact, verify CST preservation
3. **yaml-editor — switchBindingTarget**: Switch capability→subCase, verify old field deleted and new field set with defaults. Switch subCase→humanTask. Verify round-trip via `toGraph()` produces correct topology change.
4. **GitHubBackend**: Mock `fetch` — test read (ok, not_found), write (ok, conflict), verify base64 encoding/decoding, SHA threading, conflict detection flow (409 → GET → conflict result)
5. **casehub-diagram-palette**: Verify 4 items render, verify `palette-add` event with correct `elementType` on click, verify `disabled` prevents interaction
6. **casehub-diagram-toolbar**: Verify Save button reflects dirty/saving state, verify `toolbar-save` event, verify hidden when `hasBackend` false
7. **casehub-diagram — structural edits**: Load YAML → palette add → verify node appears in graph and YAML. Select node → delete → verify removed. Verify undo restores. Verify re-layout runs on structural edits.
8. **casehub-diagram — persistence**: Mock backend → verify load flow (read → render), save flow (write → version update → dirty cleared), conflict flow (dialog → overwrite/reload/cancel)
9. **casehub-diagram — delete with dependencies**: Worker with bindings → verify warning dialog appears. Unconnected node → verify immediate removal. External node → verify not deletable.
10. **casehub-diagram-properties — target switching**: Select binding → switch target type → verify `target-type-change` event with correct type. Verify panel re-renders with new target fields.
11. **Async render guard**: Trigger two rapid palette adds → verify final rendered state matches the YAML with both nodes added. Verify save is blocked while render is in progress.
12. **Dirty-on-undo**: Save → undo → verify dirty is true. Save → edit → undo → verify dirty matches whether current YAML equals saved YAML.
13. **Network errors**: Mock fetch to throw → verify error toast shown on save failure, verify error toast shown on load failure, verify `_saving` reset to false after error.
14. **Delete key in text input**: Focus a text input in properties panel → press Delete → verify node NOT removed. Click canvas background → press Delete → verify node removed.

## 11. Review Findings Addressed

| Finding | Resolution |
|---------|-----------|
| Async race on `_fullRender` (Robustness-R1-01, Cross-cutting-R1-01/02) | §7.4: Render guard with deferred re-render. |
| Undo of structural edits (Robustness-R1-02) | §7.3: Undo/redo always uses `_fullRender()`. |
| Delete key conflicts with text input (Robustness-R1-07) | §6: Check `e.target` before deletion. |
| Network error handling (Coherence-R1-03, Cross-cutting-R1-04) | §8.3: try/catch on backend calls, error toast. |
| Dirty tracking on undo (Coherence-R1-04, Cross-cutting-R1-03) | §8.3: Derived from `_currentYaml !== _savedYaml`. |
| Saving state flow (Coherence-R1-05) | §8.3: `_saving` flag, §8.5: toolbar reflects it. |
| New file expectedVersion (Coherence-R1-02) | §8.3: Empty string `''` = create new. |
| Conflict dialog three actions (Robustness-R1-06) | §8.4: Dedicated inline dialog, not blocks-confirm-dialog. |
| Testing gaps (Cross-cutting-R1-05) | §10: Items 11-14 added. |
| PersistenceBackend SPI mismatch (Coherence-R1-01, Structure-R1-03, Robustness-R1-04) | False positive — spec matches graph-core implementation (verified against source). |
| InMemoryBackend doesn't exist (Structure-R1-04, Robustness-R1-05) | False positive — exists in graph-core `persistence.ts`. |
| God component (Structure-R1-02, Cross-cutting-R1-02) | casehub-diagram is the composition root — decomposed sub-components (palette, toolbar, properties) are already separate. |
| YAML-first orphans graph-core ops (Structure-R1-06) | By design — §2 explains why. |
