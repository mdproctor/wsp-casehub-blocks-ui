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
Discriminator type changes do NOT flow through `_onPropertyChange` —
they are handled directly by render function closures in the
EditorResolver (see §Discriminator Rendering via Closures).

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

    if (schema['x-discriminator'] && schema.oneOf) {
      return {
        kind: 'render',
        render: (ctx) => this._renderDiscriminator(ctx, schema),
      };
    }
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
returns a `{ kind: 'render' }` descriptor. The render function is a
**closure that captures `this`** (the diagram component instance), giving
it direct access to `_selectedData`, `_adapterResult`, and the CST-
preserving switch functions. This is critical because pages-property-
palette's `FieldRenderContext.onChange` is typed `(value: unknown) => void`
— it takes only a value, not a field path. The field path is baked in by
pages-property-palette when creating the context. Discriminator type
changes cannot flow through `ctx.onChange`.

`swf-diagram` does not override `_editorResolver()` — SWF schemas have
no custom editor fields or discriminated unions.

### Virtual Discriminators and Data-Schema Mismatch

The case schemas use `x-discriminator` + `oneOf` as a **presentation
construct** that does not match the data structure:

**Worker `functionType`:** The schema defines `functionType` as a property
with `x-discriminator: '_type'` and 6 `oneOf` branches (agent, flow, a2a,
mcp, sequence, external). But worker data has NO `functionType` field —
function type is determined by which key is present at the top level:
`data['agent']`, `data['a2a']`, `data['mcp']`, `data['do']`, or
`data['sequence']` via `detectFunctionType()`. The schema says
`functionType._type = 'agent'`; the data says `{ agent: { ... } }`.

**Nested discriminators:** `model._provider` and `transport._transport`
within the agent and MCP branches are also virtual — the data has
`agent.model._provider` as a key in the model object, not a
discriminator field that maps cleanly to `ctx.value`.

**Binding `on` trigger:** The schema defines `on` with
`x-discriminator: 'triggerType'` and 4 branches. The `on` field DOES
exist in the data (`data['on'] = { contextChange: {...} }`), but the
discriminator key `triggerType` does not — the active trigger is
determined by which key is present inside `on`.

Because all discriminators are virtual, a generic `renderDiscriminator`
utility that reads `ctx.value` and matches `const` values cannot work.
The render functions use **domain-aware detection functions**
(`detectFunctionType`, `detectTriggerType`, `detectMcpTransport`,
`detectModelProvider`) to determine the active branch from the actual
data structure.

### Discriminator Rendering via Closures

Discriminator type changes and sub-field edits are handled through
two separate paths:

**Type changes** (structural mutations): The render function calls the
CST-preserving switch function directly via the captured `this` reference.
These never flow through `_onPropertyChange` or `ctx.onChange`.

**Sub-field edits** (value mutations): The render function creates a
nested `<pages-property-palette>` for the active branch, with a nested
`PropertyPaletteSource` whose `onChange` prefixes the field path with the
correct YAML key. These flow through the normal `_onPropertyChange` →
`_applyPropertyEdit` path.

```typescript
// casehub-diagram — discriminator rendering for functionType
private _renderDiscriminator(
  ctx: FieldRenderContext,
  schema: FieldSchema,
): TemplateResult {
  const disc = schema['x-discriminator'] as string;

  // --- Function type discriminator ---
  if (disc === '_type') {
    const fnType = detectFunctionType(this._selectedData);
    const yamlKey = FUNCTION_TYPE_TO_YAML_KEY[fnType]; // e.g., 'agent'
    const branches = schema.oneOf as FieldSchema[];
    const activeBranch = branches.find(b =>
      b.properties?.['_type']?.const === (yamlKey ?? fnType));

    return html`
      <select @change=${(e: Event) => {
        const newType = (e.target as HTMLSelectElement).value;
        this._switchFunctionType(newType as WorkerFunctionType);
      }}>
        ${branches.map(b => html`
          <option value=${b.properties?.['_type']?.const}
            ?selected=${b === activeBranch}>${b.title}</option>
        `)}
      </select>
      ${activeBranch && yamlKey ? this._renderBranchPalette(
        activeBranch, yamlKey,
      ) : nothing}
    `;
  }

  // --- Trigger type discriminator ---
  if (disc === 'triggerType') {
    return this._renderTriggerDiscriminator(ctx, schema);
  }

  // --- Model provider discriminator ---
  if (disc === '_provider') {
    return this._renderProviderDiscriminator(ctx, schema);
  }

  // --- MCP transport discriminator ---
  if (disc === '_transport') {
    return this._renderTransportDiscriminator(ctx, schema);
  }

  return html`<span>Unknown discriminator: ${disc}</span>`;
}
```

**Nested palette for sub-field edits:**

```typescript
private _renderBranchPalette(
  branch: FieldSchema,
  yamlKey: string,
): TemplateResult {
  const subData = (this._selectedData[yamlKey] ?? {}) as Record<string, unknown>;
  const subProps = { ...branch.properties };
  delete subProps['_type']; // exclude discriminator const field

  const nestedSource: PropertyPaletteSource = {
    schema: { type: 'object', properties: subProps } as FieldSchema,
    data: subData,
    readonly: this.readonly,
    onChange: (field, value) =>
      this._onPropertyChange([yamlKey, ...field], value),
  };

  return html`
    <pages-property-palette
      .source=${nestedSource}
      .resolver=${this._editorResolver()}>
    </pages-property-palette>
  `;
}
```

The nested palette uses the **same EditorResolver** (which captures
`this`), so nested discriminators (model `_provider` within agent,
transport `_transport` within MCP) are handled recursively. Each nesting
level adds the correct YAML key prefix to `onChange`, producing the full
path: e.g., `['agent', 'model', 'modelName']`.

**No `_onPropertyChange` override needed for discriminators.**
`casehub-diagram` does NOT override `_onPropertyChange`. Discriminator
type changes bypass onChange entirely (direct switch function calls).
Sub-field edits flow through the base class `_onPropertyChange` →
`_applyPropertyEdit` with the correct YAML path. The existing four
event handlers (`_handleFunctionTypeChange`, `_handleMcpTransportChange`,
`_handleModelProviderChange`, `_handleTargetTypeChange`) are removed —
their logic moves into the render function closures.

### Binding Target Type

The binding target type (capability/subCase/humanTask) is a virtual
selector like `functionType` — the three fields are mutually exclusive
but the schema does not model them as a discriminator. The existing
`_renderTargetSelector()` in `casehub-diagram-properties.ts` detects
the target type by key presence and renders a dropdown.

In the new design, `casehub-diagram` renders the target type selector
as part of the property panel for binding nodes. When the selector value
changes, it calls `switchBindingTarget()` directly via the same closure
pattern:

```typescript
private _renderTargetSelector(): TemplateResult | typeof nothing {
  if (this._selectedType !== 'binding') return nothing;
  const current = this._currentTargetType();
  return html`
    <select @change=${(e: Event) => {
      const newTarget = (e.target as HTMLSelectElement).value;
      this._switchBindingTarget(newTarget as 'capability' | 'subCase' | 'humanTask');
    }}>
      <option value="capability" ?selected=${current === 'capability'}>Capability</option>
      <option value="subCase" ?selected=${current === 'subCase'}>SubCase</option>
      <option value="humanTask" ?selected=${current === 'humanTask'}>HumanTask</option>
    </select>
  `;
}
```

`_currentTargetType()` detects the active target by key presence, matching
the existing logic. `_switchBindingTarget` wraps `switchBindingTarget()`
with undo tracking and `_fullRender`.

Modeling the binding target type as a proper `x-discriminator` in the
binding schema (for consistency with `functionType` and `on`) is deferred
to casehubio/blocks-ui#144.

### Trigger Type Switching

The existing `trigger-editor.ts` (being removed from diagram-core) handles
trigger type switching by replacing the entire `on` value — a crude approach
that loses YAML comments. For consistency with the other CST-preserving
switch functions, a new `switchTriggerType` is added:

```typescript
// yaml-editor.ts — new function
const TRIGGER_KEYS = ['contextChange', 'cloudEvent', 'schedule', 'scopeActivated'];
const TRIGGER_DEFAULTS: Record<string, unknown> = {
  contextChange: {},
  cloudEvent: {},
  schedule: {},
  scopeActivated: {},
};

export function switchTriggerType(
  yaml: string,
  bindingPath: readonly (string | number)[],
  newType: string,
): string {
  const doc = parseDocument(yaml);
  const onPath = [...bindingPath, 'on'];
  const on = doc.getIn(onPath) as YAMLMap;
  for (const key of TRIGGER_KEYS) {
    if (on.has(key)) on.delete(key);
  }
  on.set(newType, doc.createNode(TRIGGER_DEFAULTS[newType]));
  return doc.toString();
}
```

`detectTriggerType` migrates from `diagram-core/src/form/trigger-editor.ts`
(being removed) to `graph-stencil-case/src/worker-function/detect.ts`
alongside the other detection functions. The type alias `TriggerType` moves
with it.

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
2. EditorResolver with closure-captured render functions — discriminator
   type selectors + nested sub-palettes (see §Discriminator Rendering
   via Closures)
3. Direct switch function calls from render closures — CST-preserving
   YAML mutations without routing through `_onPropertyChange`
4. `x-editor-component` editors — blocks-prompt-editor, blocks-env-map-editor, etc.
5. Binding target type selector — rendered directly by casehub-diagram
   (see §Binding Target Type)

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

**`_paletteTypes()` migration:** The existing `_paletteTypes()` method
and all its overrides are removed. Three call sites must be updated:

1. `DiagramBaseMixin._handleKeydown` (line 378): guards Delete/Backspace
   on `this._paletteTypes().length > 0`. Updated to check
   `this._editPolicy() != null` — if an edit policy exists, editing
   (including deletion) is enabled.
2. `casehub-diagram._paletteTypes()` returning `PALETTE_TYPES` — removed.
   Palette items now derive from `CaseEditPolicy.getCreatableTypes()`.
3. `swf-diagram._paletteTypes()` returning `[]` — removed. Palette items
   now derive from `SwfEditPolicy.getCreatableTypes()`.

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
| `packages/diagram-core/src/form/trigger-editor.ts` | `detectTriggerType` and `TriggerType` migrate to `graph-stencil-case/src/worker-function/detect.ts`; `renderTriggerEditor` removed — replaced by EditorResolver discriminator rendering |
| `packages/diagram-core/src/form/nested-group.ts` | Replaced by pages-property-palette nested object rendering |
| `packages/diagram-core/src/form/property-form.ts` | Replaced by pages-property-palette |
| `packages/diagram-core/src/diagram-properties.ts` | Replaced by pages-property-palette (used inline in mixin) |
| `components/casehub-diagram/src/casehub-diagram-properties.ts` | Replaced by pages-property-palette + EditorResolver discriminator support (see §casehub-diagram-properties.ts Migration) |
| `components/casehub-diagram/src/casehub-diagram-properties.test.ts` | Tests for removed component |
| `components/casehub-diagram/src/casehub-diagram-palette.ts` | Replaced by pages-diagram-palette |
| `components/casehub-diagram/src/casehub-diagram-palette.test.ts` | Tests for removed component |
| Inline prompt dialog in `casehub-diagram.ts` | The `<dialog id="prompt-editor-dialog">` block (lines 270-296), `_promptEditorOpen`/`_promptEditorValue` state, and `_handlePromptEditor*` methods are removed. `blocks-prompt-editor` with `x-editor-component` provides the editing surface. |
| Event handlers in `casehub-diagram.ts` | `_handleFunctionTypeChange`, `_handleMcpTransportChange`, `_handleModelProviderChange`, `_handleTargetTypeChange` — replaced by direct switch function calls from EditorResolver render closures (see §Discriminator Rendering via Closures). |

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
- `EditorResolver`: x-editor-component → tag descriptor, x-discriminator → render descriptor
- `switchTriggerType`: CST-preserving trigger type switching
- Discriminator render closures: type detection, nested palette construction
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
- packages/graph-stencil-case/src/adapter/yaml-editor.ts — addElement, switchFunctionType, switchTriggerType, etc.
- packages/graph-stencil-swf/src/adapter/swf-yaml-editor.ts — applySwfPropertyEdit, addSwfTask
- PP-20260806-320d50 — stencil package isolation protocol
- PP-20260713-8ea1af — component customisation pattern
