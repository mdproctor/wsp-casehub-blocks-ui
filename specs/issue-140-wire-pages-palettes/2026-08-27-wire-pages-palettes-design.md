# Wire Pages Palette Components and Make All Diagrams Editable

**Date:** 2026-08-27
**Issue:** casehubio/blocks-ui#140
**Status:** Draft

## Summary

Consume three pages capabilities — `pages-property-palette` (#373),
`pages-diagram-palette` (#380), and `EditPolicy` SPI (#378) — across
all blocks-ui diagram types. Replace the hand-built property form and
stencil palette with pages components. Implement `CaseEditPolicy` and
`SwfEditPolicy` for validated graph mutations. Remove the old `form/`
directory from diagram-core.

## Architecture

### Dependency Update

Add `pages-property-palette` and `pages-diagram-palette` to the workspace.
Publish pages SNAPSHOT (`yarn build && mvn install` in casehub-pages) so
blocks-ui's `yarn install` picks up both packages. Add to
`packages/diagram-core/package.json` dependencies.

### Property Palette Migration

**Before:** `DiagramBaseMixin._updateSelectedNode()` sets `_selectedSchema`
and `_selectedData`. `DiagramProperties` component renders via
`renderPropertyForm()` from `form/property-form.ts`.

**After:** `DiagramBaseMixin._updateSelectedNode()` sets `_selectedSchema`
and `_selectedData` (unchanged). A new `_propertyPaletteSource` getter
constructs a `PropertyPaletteSource` object bridging to the existing
state. `DiagramBaseMixin._renderPropertyPanel()` renders
`<pages-property-palette>` with `.source` and `.resolver`.

```typescript
protected get _propertyPaletteSource(): PropertyPaletteSource | undefined {
  if (!this._selectedNodeId) return undefined;
  return {
    schema: this._selectedSchema as FieldSchema,
    data: this._selectedData,
    readonly: this.readonly,
    onChange: (field, value) => this._onPropertyChange(field, value),
  };
}
```

The `onChange` routes through a new `_onPropertyChange(field, value)`
method on DiagramBaseMixin. The base implementation wraps undo tracking
and delegates to `_applyPropertyEdit` for regular field changes.
Subclasses override to intercept discriminator changes and route them
to specialised CST-preserving YAML editors (see §Discriminator Routing).

### EditorResolver

`DiagramBaseMixin` provides a default `_editorResolver()` method returning
`undefined` (no custom editors). Subclasses override to wire domain-specific
editors:

```typescript
// DiagramBaseMixin (default — not abstract)
protected _editorResolver(): EditorResolver | undefined {
  return undefined;
}

// casehub-diagram override
protected override _editorResolver(): EditorResolver {
  return (schema) => {
    const tag = schema['x-editor-component'] as string | undefined;
    if (tag) return { kind: 'tag', tag };
    return undefined;
  };
}
```

The resolver checks `x-editor-component` on the schema and returns a
`{ kind: 'tag', tag }` descriptor. `pages-property-palette` creates the
element via `document.createElement(tag)` and listens for `change`
events. Custom editors that currently emit `value-changed` must be
updated to emit standard `change` events to match this contract (see
§Custom Editor Event Contract).

For discriminated unions (`x-discriminator` + `oneOf`), the resolver
returns a `{ kind: 'render' }` descriptor with a render function that
handles the full discriminator UX. This keeps all discriminator
complexity in the EditorResolver — pages-property-palette does not
need native `x-discriminator` support:

```typescript
// casehub-diagram override — handles both custom editors and discriminators
protected override _editorResolver(): EditorResolver {
  return (schema) => {
    const tag = schema['x-editor-component'] as string | undefined;
    if (tag) return { kind: 'tag', tag };

    if (schema['x-discriminator'] && schema.oneOf) {
      return {
        kind: 'render',
        render: (ctx) => renderDiscriminator(ctx, schema),
      };
    }
    return undefined;
  };
}
```

`renderDiscriminator` is a shared utility in diagram-core that:
1. Reads the `x-discriminator` field name and `oneOf` branches
2. Determines the active branch by matching data against each branch's
   `const` value for the discriminator field
3. Renders a type selector dropdown from branch titles
4. Renders a nested `<pages-property-palette>` for the active branch's
   sub-schema
5. Routes discriminator changes through `ctx.onChange` with the
   discriminator field path (e.g. `['functionType', '_type']`)

`swf-diagram` does not override `_editorResolver()` — SWF schemas have
no custom editor fields or discriminated unions.

### Discriminator Routing

`DiagramBaseMixin._onPropertyChange(field, value)` handles regular
field changes via `_applyPropertyEdit`. `casehub-diagram` overrides
this to intercept discriminator changes before they reach the generic
editor:

```typescript
// casehub-diagram
protected override _onPropertyChange(
  field: (string | number)[], value: unknown,
): void {
  const key = String(field[field.length - 1]);
  if (key === '_type' && field[0] === 'functionType') {
    this._switchFunctionType(value as WorkerFunctionType);
    return;
  }
  if (key === '_transport') {
    this._switchMcpTransport(value as McpTransportType);
    return;
  }
  if (key === '_provider') {
    this._switchModelProvider(value as ModelProviderKey);
    return;
  }
  if (field[0] === 'targetType') {
    this._switchBindingTarget(value as string);
    return;
  }
  super._onPropertyChange(field, value);
}
```

Each `_switch*` method wraps the existing CST-preserving YAML editor
(`switchFunctionType`, `switchMcpTransport`, `switchModelProvider`,
`switchBindingTarget`) with undo tracking and `_fullRender`.

This approach handles all four discriminator levels in the worker schema
(`functionType._type` → 6 branches, `model._provider` → 5 providers,
`transport._transport` → stdio/http) and the binding target type, using
the same specialized YAML mutation functions that `casehub-diagram.ts`
already has (`_handleFunctionTypeChange`, `_handleMcpTransportChange`,
etc.).

### casehub-diagram-properties.ts Migration

The existing `casehub-diagram-properties.ts` (177 lines) contains:
- Function type detection and sub-form rendering (`_renderFunctionTypeSection`)
- Binding target type selector (`_renderTargetSelector`)
- Schema filtering to remove function-type keys (`_filteredSchema`)
- Sub-form delegation (renderAgentForm, renderA2AForm, renderMcpForm, etc.)
- Five specialised events (target-type-change, function-type-change,
  mcp-transport-change, model-provider-change, prompt-editor-open)

All of this is replaced by the combination of:
1. `pages-property-palette` — generic field rendering
2. EditorResolver discriminator support — type selector + sub-schema swap
3. Discriminator routing in `_onPropertyChange` — YAML mutation dispatch
4. `x-editor-component` editors — blocks-prompt-editor, blocks-env-map-editor, etc.

The component is added to the Removal List. Its sub-form renderers
(renderAgentForm, renderA2AForm, etc.) in graph-stencil-case are also
removed — the worker schema's `oneOf` branches drive rendering directly.

### Stencil Palette

**Before:** `casehub-diagram-palette` is a hand-built 4-button component
that emits `palette-add` with `{ elementType }`.

**After:** `DiagramBaseMixin` renders `<pages-diagram-palette>` with items
from `_paletteTypes()` (existing abstract method). The palette emits
`pages-palette-select` with `{ item: PaletteItem }`.

`_paletteTypes()` is replaced by `_paletteItems()`, which derives items
from `EditPolicy.getCreatableTypes()` rather than hardcoding. The
`defaultEditPolicy()` in pages' graph-renderer already reads from the
stencil registry via `getAllStencils()`, so new stencils automatically
appear in the palette after registration.

```typescript
// DiagramBaseMixin — default implementation
protected _paletteItems(): PaletteItem[] {
  const policy = this._editPolicy();
  if (!policy) return [];
  return policy.getCreatableTypes(null, this._adapterResult?.model ?? emptyModel)
    .map(s => ({ type: s.type, label: s.label, icon: s.icon, group: s.group }));
}
```

Subclasses can override `_editPolicy()` to provide domain-specific
filtering. For case diagrams, `CaseEditPolicy.getCreatableTypes()`
filters to the 4 creatable types: binding, worker, milestone, goal.

Note: `subcase` is NOT a creatable type. Subcases are binding target
references — they appear as graph nodes when a binding's target is set
to `subCase`, but cannot be independently created via the palette. The
subcase stencil is registered for rendering only.

For SWF diagrams, `SwfEditPolicy.getCreatableTypes()` filters to user-
creatable task types: call, set, switch, raise, try.

The `casehub-diagram-palette` component is removed.

### EditPolicy Implementation

Two implementations, registered per diagram type:

**CaseEditPolicy:**
- `canConnect(source, target)`: binding → worker via capability match only
- `getCreatableTypes()`: 4 creatable types (binding, worker, milestone, goal — NOT subcase, which is a binding target reference)
- `canDelete(node)`: always true for user-created nodes; subcase nodes are non-deletable (they disappear when the binding target changes)
- `getDeleteStrategy(node)`: `auto-join` for binding/worker (reconnect edges), `disconnect` for milestone/goal

**SwfEditPolicy:**
- `canConnect(source, target)`: any task → any task (flow edges); switch → case targets
- `getCreatableTypes()`: all SWF task types from `registerSwfStencils()`
- `canDelete(node)`: true except start/end boundary nodes
- `getDeleteStrategy(node)`: `auto-join` (reconnect flow edges around deleted node)

Each policy lives in its stencil package (per PP-20260806-320d50 isolation):
- `packages/graph-stencil-case/src/editing/case-edit-policy.ts`
- `packages/graph-stencil-swf/src/editing/swf-edit-policy.ts`

`DiagramBaseMixin` receives the policy via a new protected method
`_editPolicy()` (default returns `undefined` — read-only diagrams).
Subclasses override to provide their domain policy.

### Add-Node Flow

The mutation path is YAML-first. `_currentYaml` is the source of truth;
the `GraphModel` is derived from YAML via `_adaptYaml()` and has no
reverse serializer. `GraphEdit`/`applyGraphEdit` (from pages#378)
operate on the in-memory GraphModel — they are not used in the add-node
flow because the YAML mutation IS the operation.

```
pages-diagram-palette
  → pages-palette-select event
  → DiagramBaseMixin._handlePaletteSelect(item)
  → Validates type via _editPolicy().getCreatableTypes()
  → Delegates to abstract _addElement(type) on the subclass
  → Domain adapter creates the YAML element
  → _fullRender() updates the canvas
```

`_addElement(type: string)` is a new abstract method on DiagramBaseMixin.
Each subclass implements the domain-specific YAML mutation:

```typescript
// casehub-diagram
protected override _addElement(type: string): void {
  this._currentYaml = addElement(
    this._currentYaml,
    type as 'binding' | 'worker' | 'milestone' | 'goal',
  );
}

// swf-diagram
protected override _addElement(type: string): void {
  this._currentYaml = addSwfTask(this._currentYaml, type);
}
```

### SWF YAML Mutation

`addSwfTask()` is a new function in `swf-yaml-editor.ts` that inserts
a new named task entry under the `do:` block. Each SWF task type has a
type-specific YAML template:

```typescript
const SWF_TASK_DEFAULTS: Record<string, (n: number) => Record<string, unknown>> = {
  'swf-call': (n) => ({ call: 'http:get', with: {} }),
  'swf-set': (n) => ({ set: {} }),
  'swf-switch': (n) => ({ switch: [{ when: '.condition == true', then: 'continue' }] }),
  'swf-raise': (n) => ({ raise: { error: { type: 'error', status: 500, title: 'Error' } } }),
  'swf-try': (n) => ({ try: { call: 'http:get' }, catch: { as: 'error' } }),
};

export function addSwfTask(yaml: string, taskType: string): string {
  // Insert new named task at end of do: block with type-specific defaults
}
```

### Custom Editor Event Contract

`pages-property-palette` listens for standard `change` events on
tag-based custom editors (line 264 in the source). The existing custom
editors emit `value-changed` instead — this is a contract mismatch.

**Fix:** Update the two editors that emit events to use `change`:
- `blocks-env-map-editor`: change `value-changed` → `change`
- `blocks-prompt-editor`: change `value-changed` → `change`

Read-only editors (`blocks-sequence-editor`, `blocks-swf-link`,
`blocks-json-editor`) emit no events and need no changes.

### Removal List

| File | Reason |
|------|--------|
| `packages/diagram-core/src/form/field-renderer.ts` | Replaced by pages-property-palette EditorResolver |
| `packages/diagram-core/src/form/validation.ts` | Replaced by pages-ui-components validateField |
| `packages/diagram-core/src/form/trigger-editor.ts` | Replaced by x-discriminator in binding schema |
| `packages/diagram-core/src/form/nested-group.ts` | Replaced by pages-property-palette nested object rendering |
| `packages/diagram-core/src/form/property-form.ts` | Replaced by pages-property-palette |
| `packages/diagram-core/src/diagram-properties.ts` | Replaced by pages-property-palette (used inline in mixin) |
| `components/casehub-diagram/src/casehub-diagram-properties.ts` | Replaced by pages-property-palette + EditorResolver discriminator support (see §casehub-diagram-properties.ts Migration) |
| `components/casehub-diagram/src/casehub-diagram-properties.test.ts` | Tests for removed component |
| `components/casehub-diagram/src/casehub-diagram-palette.ts` | Replaced by pages-diagram-palette |
| `components/casehub-diagram/src/casehub-diagram-palette.test.ts` | Tests for removed component |
| Inline prompt dialog in `casehub-diagram.ts` | The `<dialog id="prompt-editor-dialog">` block (lines 270-296), `_promptEditorOpen`/`_promptEditorValue` state, and `_handlePromptEditor*` methods are removed. `blocks-prompt-editor` with `x-editor-component` provides the editing surface. |

Exports removed from `packages/diagram-core/src/index.ts`:
`DiagramProperties`, `renderPropertyForm`, `emitPropertyChange`,
`fieldTypeFor`, `FieldType`, `validateField`, `FieldSchema`,
`renderTriggerEditor`, `detectTriggerType`, `TriggerType`,
`renderNestedGroup`.

Sub-form renderers removed from `packages/graph-stencil-case/src/`:
`renderAgentForm`, `renderA2AForm`, `renderMcpForm`,
`renderSequenceForm`, `renderUnknownForm` — replaced by worker schema
`oneOf` branches rendered via EditorResolver discriminator support.

### Showcase Gallery Updates

Update three existing pages to demonstrate the full editing UX:

**casehub-diagram-page.ts:** Property palette visible on node selection.
Stencil palette sidebar with all 4 case types. Click-to-add a new worker
and see its schema-driven properties.

**swf-diagram-page.ts:** Property palette with x-group annotations visible.
Stencil palette with SWF task types. Add a new call task.

**diagram-workbench-page.ts:** Both palettes visible in the split-pane layout.
Property palette updates when switching between case (left) and SWF (right)
panes.

## Testing

### Unit Tests

- `PropertyPaletteSource` adapter: schema/data bridge, onChange routing
- `EditorResolver`: x-editor-component → tag descriptor mapping
- `CaseEditPolicy`: canConnect rules, creatable types, delete strategies
- `SwfEditPolicy`: flow edge validation, boundary node protection
- Palette item generation from `_paletteItems()`

### Integration Tests

- Node selection → property palette renders correct schema groups
- Palette select → new node created in YAML
- Discriminator change → correct CST editor invoked
- Read-only mode → palette disabled, property palette read-only

## What This Does NOT Cover

Each deferred item is captured as a GitHub issue:

- **Drag-to-canvas** (casehubio/blocks-ui#141) — pages-diagram-palette
  is click-to-add only; drag-to-canvas requires ReactFlowApp `onPaneClick`
  + coordinate transform via ViewportBridge. Issue #140's acceptance
  criteria updated from "drag-to-add" to "click-to-add" accordingly.
- **Edge reconnection and deletion UX** (casehubio/blocks-ui#142) —
  EditPolicy supports it but requires ReactFlowApp `onReconnect` callback
  wiring.
- **Context menus** (casehubio/blocks-ui#143) — EditPolicy supports
  `getInsertableTypes` for edge splitting but requires right-click menu
  component.

## References

- casehubio/casehub-pages#373 — pages-property-palette (PropertyPaletteSource SPI)
- casehubio/casehub-pages#378 — diagram editing infrastructure (EditPolicy, GraphEdit, applyGraphEdit)
- casehubio/casehub-pages#380 — pages-diagram-palette (PaletteItem, pages-palette-select)
- casehubio/blocks-ui#136 — property schemas (registerPropertySchema, all schemas)
- packages/diagram-core/src/diagram-base-mixin.ts — _updateSelectedNode, _handlePropertyChange
- packages/diagram-core/src/form/ — old form utilities (to be removed)
- components/casehub-diagram/src/casehub-diagram-palette.ts — old palette (to be removed)
- packages/graph-stencil-case/src/adapter/yaml-editor.ts — addElement, switchFunctionType, etc.
- packages/graph-stencil-swf/src/adapter/swf-yaml-editor.ts — applySwfPropertyEdit, addSwfTask
- PP-20260806-320d50 — stencil package isolation protocol
- PP-20260713-8ea1af — component customisation pattern
