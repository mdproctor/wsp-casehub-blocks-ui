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
    onChange: (field, value) => this._handlePropertyChange(
      new CustomEvent('property-change', { detail: { field, value } })
    ),
  };
}
```

The `onChange` routes through the existing `_handlePropertyChange` method,
which already handles discriminator detection (`functionType`,
`transportType`, `modelProvider`, `targetType`) and delegates to the
correct CST-preserving YAML editor.

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
element via `document.createElement(tag)` and wires `value`/`change`
events automatically.

`swf-diagram` does not override `_editorResolver()` — SWF schemas have
no custom editor fields.

### Stencil Palette

**Before:** `casehub-diagram-palette` is a hand-built 4-button component
that emits `palette-add` with `{ elementType }`.

**After:** `DiagramBaseMixin` renders `<pages-diagram-palette>` with items
from `_paletteTypes()` (existing abstract method). The palette emits
`pages-palette-select` with `{ item: PaletteItem }`.

`_paletteTypes()` already returns the creatable types per diagram. The
return type changes from the current format to `PaletteItem[]`:

```typescript
// casehub-diagram
protected override _paletteTypes(): PaletteItem[] {
  return [
    { type: 'binding', label: 'Binding', icon: 'link', group: 'Elements' },
    { type: 'worker', label: 'Worker', icon: 'cpu', group: 'Elements' },
    { type: 'milestone', label: 'Milestone', icon: 'flag', group: 'Markers' },
    { type: 'goal', label: 'Goal', icon: 'target', group: 'Markers' },
    { type: 'subcase', label: 'SubCase', icon: 'layers', group: 'Elements' },
  ];
}

// swf-diagram
protected override _paletteTypes(): PaletteItem[] {
  return [
    { type: 'swf-call', label: 'Call', icon: 'phone', group: 'Tasks' },
    { type: 'swf-set', label: 'Set', icon: 'edit', group: 'Tasks' },
    { type: 'swf-switch', label: 'Switch', icon: 'git-branch', group: 'Flow' },
    { type: 'swf-raise', label: 'Raise', icon: 'alert-triangle', group: 'Error' },
    { type: 'swf-try', label: 'Try', icon: 'shield', group: 'Error' },
  ];
}
```

The `casehub-diagram-palette` component is removed.

### EditPolicy Implementation

Two implementations, registered per diagram type:

**CaseEditPolicy:**
- `canConnect(source, target)`: binding → worker via capability match only
- `getCreatableTypes()`: all 5 case types (binding, worker, milestone, goal, subcase)
- `canDelete(node)`: always true for user-created nodes
- `getDeleteStrategy(node)`: `auto-join` for binding/worker (reconnect edges), `disconnect` for milestone/goal/subcase

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

```
pages-diagram-palette
  → pages-palette-select event
  → DiagramBaseMixin._handlePaletteSelect(item)
  → Creates GraphEdit.addNode { nodeType: item.type }
  → EditPolicy.getCreatableTypes() validates type is allowed
  → Domain adapter creates the YAML element (existing addElement/applySwfPropertyEdit)
  → _fullRender() updates the canvas
```

The existing `addElement()` in the case adapter and `applySwfPropertyEdit()`
in the SWF adapter become the persistence layer — called by the mixin after
EditPolicy validation, not directly by the palette.

### Removal List

| File | Reason |
|------|--------|
| `packages/diagram-core/src/form/field-renderer.ts` | Replaced by pages-property-palette EditorResolver |
| `packages/diagram-core/src/form/validation.ts` | Replaced by pages-ui-components validateField |
| `packages/diagram-core/src/form/trigger-editor.ts` | Replaced by x-discriminator in binding schema |
| `packages/diagram-core/src/form/nested-group.ts` | Replaced by pages-property-palette nested object rendering |
| `packages/diagram-core/src/form/property-form.ts` | Replaced by pages-property-palette |
| `packages/diagram-core/src/diagram-properties.ts` | Replaced by pages-property-palette (used inline in mixin) |
| `components/casehub-diagram/src/casehub-diagram-palette.ts` | Replaced by pages-diagram-palette |
| `components/casehub-diagram/src/casehub-diagram-palette.test.ts` | Tests for removed component |

Exports removed from `packages/diagram-core/src/index.ts`:
`DiagramProperties`, `renderPropertyForm`, `emitPropertyChange`,
`fieldTypeFor`, `FieldType`, `validateField`, `FieldSchema`,
`renderTriggerEditor`, `detectTriggerType`, `TriggerType`,
`renderNestedGroup`.

### Showcase Gallery Updates

Update three existing pages to demonstrate the full editing UX:

**casehub-diagram-page.ts:** Property palette visible on node selection.
Stencil palette sidebar with all 5 case types. Click-to-add a new worker
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
- Palette item generation from `_paletteTypes()`

### Integration Tests

- Node selection → property palette renders correct schema groups
- Palette select → new node created in YAML
- Discriminator change → correct CST editor invoked
- Read-only mode → palette disabled, property palette read-only

## What This Does NOT Cover

- Drag-to-canvas (pages-diagram-palette supports it but requires
  ReactFlowApp `onPaneClick` + coordinate transform via ViewportBridge —
  deferred to a follow-on)
- Edge reconnection and deletion UX (EditPolicy supports it but requires
  ReactFlowApp `onReconnect` callback wiring — deferred)
- Context menus (EditPolicy supports `getInsertableTypes` for edge splitting
  but requires right-click menu component — deferred)

## References

- casehubio/casehub-pages#373 — pages-property-palette (PropertyPaletteSource SPI)
- casehubio/casehub-pages#378 — diagram editing infrastructure (EditPolicy, GraphEdit, applyGraphEdit)
- casehubio/casehub-pages#380 — pages-diagram-palette (PaletteItem, pages-palette-select)
- casehubio/blocks-ui#136 — property schemas (registerPropertySchema, all schemas)
- packages/diagram-core/src/diagram-base-mixin.ts — _updateSelectedNode, _handlePropertyChange
- packages/diagram-core/src/form/ — old form utilities (to be removed)
- components/casehub-diagram/src/casehub-diagram-palette.ts — old palette (to be removed)
- packages/graph-stencil-case/src/adapter/yaml-editor.ts — addElement, switchFunctionType, etc.
- packages/graph-stencil-swf/src/adapter/swf-property-edit.ts — applySwfPropertyEdit
- PP-20260806-320d50 — stencil package isolation protocol
- PP-20260713-8ea1af — component customisation pattern
