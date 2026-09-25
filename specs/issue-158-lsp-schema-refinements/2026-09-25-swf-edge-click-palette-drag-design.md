# SWF Edge Click Picker and Palette Drag-to-Canvas

**Issue:** casehubio/blocks-ui#173
**Branch:** issue-158-lsp-schema-refinements
**Date:** 2026-09-25

## Overview

Two interaction gaps in the SWF diagram: (1) clicking an edge does
nothing — it should show a node picker to insert a task on that edge,
and (2) palette drag-to-canvas may work at the GraphCanvas level but
has not been verified end-to-end in the SWF diagram. Additionally, file
pages issues with design sketches for the long-term mixin refactoring
that would eliminate the need for per-diagram render overrides.

## Track 1: Edge Click Picker (interim, blocks-ui only)

### Current State

GraphCanvas emits `graph:edge:click` with `{ edgeId, edgeType }`
(GraphCanvas.ts:573-578). SwfDiagram's `@pages-event` handler
(swf-diagram.ts:319-326) does not include a branch for this event.

The mixin's picker infrastructure uses `_chooserState: { x, y,
sourceNodeId? }` (diagram-base-mixin.ts:239). There is no `edgeId`
field. Two existing entry points populate this state:
- `_showPickerAtPaneClick` — sets `{ x, y }`, picker select dispatches
  `addNode` (diagram-base-mixin.ts:249-252)
- `_showPickerAtConnectEnd` — sets `{ x, y, sourceNodeId }`, picker
  select dispatches `addNode` + `addEdge` (diagram-base-mixin.ts:254-261)

### Design

Add a third entry point for edge-click insertion in SwfDiagram without
modifying the pages mixin.

**New state:** `_pendingEdgeId: string | null` on SwfDiagram. This
tracks the edge targeted for insertion, kept separate from
`_chooserState` to avoid type incompatibility with the mixin.

**Event wiring:** Add `graph:edge:click` to the `@pages-event` switch
in SwfDiagram's render (swf-diagram.ts:319-326):
- Set `_pendingEdgeId` to the clicked edgeId
- Set `_chooserState` to `{ x: lastPointerX, y: lastPointerY }` to
  show the picker

**Type filtering (D10):** Override `_chooserItems()` in SwfDiagram.
When `_pendingEdgeId` is set, call `getInsertableTypes(edgeId)` on the
edit policy and convert the returned type strings to `PaletteItem[]`
using the stencil registry. When `_pendingEdgeId` is null, delegate to
`super._chooserItems()` (existing grammar-filtered list for pane-click
and connect-end-on-empty flows).

The edit policy method filters `getCreatableTypes()` through
`canSpliceOntoEdge(type, edgeId)` to return only grammar-valid types
for that edge position. The component can further narrow this list for
UX reasons.

`getInsertableTypes` signature change: add optional `edgeId?: string`
parameter on the SWF implementation only (swf-edit-policy.ts:73-75).
The EditPolicy interface in pages stays unchanged — TypeScript allows
implementations to accept additional optional parameters. The current
implementation takes no args and returns `[]`. No existing callers
break.

**Select handler (D11):** Override `_onChooserSelect` in SwfDiagram.
When `_pendingEdgeId` is set:
1. Dispatch `{ type: 'splitEdge', edgeId: _pendingEdgeId, nodeType }`
   via `_handleMutation`
2. Clear `_pendingEdgeId`
3. Clear `_chooserState`

When `_pendingEdgeId` is null, delegate to `super._onChooserSelect`
(existing addNode/addEdge behaviour).

**Dismiss handler:** Override `_onChooserDismiss` to also clear
`_pendingEdgeId`.

### Files Changed

| File | Change |
|------|--------|
| `components/swf-diagram/src/swf-diagram.ts` | Add `_pendingEdgeId` state, `graph:edge:click` handler, override `_chooserItems`, `_onChooserSelect`, `_onChooserDismiss` |
| `packages/graph-stencil-swf/src/editing/swf-edit-policy.ts` | Implement `getInsertableTypes(edgeId?)` — filter creatable types through `canSpliceOntoEdge` |

## Track 2: Palette Drag-to-Canvas Verification

### Current State

GraphCanvas handles palette drops (GraphCanvas.ts:361-379) with
`getAddPlacement` integration. The SWF edit policy already implements
`getAddPlacement` (swf-edit-policy.ts:81-90) — returns `splitEdge`
targeting the edge before `swf-end` when the pipeline is linear.

The SWF diagram's `_handlePaletteSelect` override
(swf-diagram.ts:142-154) already calls `getAddPlacement` and dispatches
`splitEdge` for palette click-to-add.

### Design

Verify end-to-end that:
1. Dragging a palette item over an edge shows the green splice indicator
2. Dropping on an edge inserts the node at that position (splitEdge)
3. Dropping on empty canvas uses `getAddPlacement` fallback
4. The YAML is updated correctly after drag-and-drop insertion

Fix any gaps found during verification. Expected issues:
- The SWF diagram's custom `_fullRender` might not re-render splice
  indicators correctly after layout (the base mixin delegates to
  `toReactFlowGraph` which handles this, but the SWF override calls it
  differently)
- Edge highlighting during drag may require `onDragOver` event handling
  that the SWF render template doesn't wire

### Files Changed

Depends on verification — may be zero changes if it already works, or
targeted fixes in `swf-diagram.ts` render template.

## Track 3: Pages Issues with Design Sketches

File two issues on casehub-pages describing the long-term mixin changes
that would eliminate per-diagram render duplication.

### Issue A: Layout hook (`_computeLayout`)

**Problem:** DiagramBaseMixin hardcodes `computeElkLayout` in
`_fullRender` (diagram-base-mixin.ts:405). SWF must override
`_fullRender` entirely to use `computeSwfStackLayout`. This forces
copy-paste of the render pipeline.

**Design sketch:**
```typescript
// New protected method on DiagramBaseMixin
protected async _computeLayout(
  model: GraphModel,
  options: LayoutOptions
): Promise<LayoutResult> {
  return computeElkLayout(model, options);
}
```

SWF overrides only `_computeLayout` to return
`computeSwfStackLayout(model)`. The base `_fullRender` calls
`this._computeLayout(...)` instead of `computeElkLayout(...)` directly.
No other diagram changes needed.

### Issue B: Standard render template with event wiring

**Problem:** The mixin provides no `render()` — each diagram assembles
its own template with manual event wiring. SWF, case, and org diagrams
all have near-identical render methods (~70 lines each) with the same
toolbar/palette/canvas/properties/picker structure. New events
(edge-click, palette-drop) must be wired in every diagram.

**Design sketch:**
```typescript
// New protected method on DiagramBaseMixin
protected _renderDiagramShell(): TemplateResult {
  return html`
    <diagram-toolbar ...></diagram-toolbar>
    <div class="diagram-body">
      ${this._renderStencilPalette()}
      <div class="diagram-canvas" @pointerdown=${this._onCanvasPointerDown}>
        <pages-graph-canvas
          .graph=${this._reactFlowGraph}
          .editPolicy=${this._editPolicy}
          @pages-event=${this._handleCanvasEvent}
        ></pages-graph-canvas>
        ${this._renderNodePicker()}
      </div>
      ${this._renderPropertyPanel()}
    </div>
    ${this._renderConflictDialog()}
  `;
}

// _handleCanvasEvent dispatches ALL graph events
protected _handleCanvasEvent(e: CustomEvent): void {
  const topic = e.detail?.topic;
  switch (topic) {
    case 'graph:node:click': ...
    case 'graph:edge:click': this._handleEdgeClick(e); break;
    case 'graph:selection:change': ...
    case 'graph:pane:click': ...
    case 'graph:connect:end-on-empty': ...
    // all events wired by default
  }
}

// Subclasses override specific handlers, not the template
protected _handleEdgeClick(e: CustomEvent): void {
  // default: show picker with splitEdge — subclasses can override
}
```

Each diagram's `render()` calls `this._renderDiagramShell()` and adds
only domain-specific elements (e.g. SWF drill-down handler, org chip
bar toggles).

## Testing

### Edge Click Picker
- Unit: `getInsertableTypes(edgeId)` returns correct subset for linear
  pipeline, branching graph, and empty graph
- Unit: `_onChooserSelect` with `_pendingEdgeId` dispatches `splitEdge`
- Integration: click edge → picker appears → select type → node
  inserted at correct position in YAML
- Edge case: click edge on readonly diagram → no picker
- Edge case: click edge where no types are insertable → picker shows
  empty or doesn't appear

### Palette Drag
- Integration: drag palette item over edge → splice indicator visible
- Integration: drop on edge → node inserted via splitEdge
- Integration: drop on empty canvas → getAddPlacement fallback

### Regression
- Existing pane-click picker still works
- Existing connect-end-on-empty picker still works
- Existing palette click-to-add still works

## References

- swf-diagram.ts:319-326 — current event handler switch
- swf-diagram.ts:142-154 — _handlePaletteSelect with getAddPlacement
- swf-edit-policy.ts:66-90 — canSpliceOntoEdge, getInsertableTypes, getCreatableTypes, getAddPlacement
- diagram-base-mixin.ts:239-337 — picker infrastructure (_chooserState, _showPickerAt*, _chooserItems, _onChooserSelect, _renderNodePicker)
- GraphCanvas.ts:573-578 — graph:edge:click emission
- GraphCanvas.ts:361-379 — palette drop handler with getAddPlacement
- D10: Edge-click type filtering (decisions.md)
- D11: Picker positioning on edge click (decisions.md)
- casehubio/blocks-ui#170 — stack-column layout (related, separate issue)
