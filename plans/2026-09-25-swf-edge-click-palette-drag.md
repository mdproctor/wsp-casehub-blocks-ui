# SWF Edge Click Picker and Palette Drag-to-Canvas Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #173 — SWF diagram: edge click picker and palette drag-to-canvas drop zones
**Issue group:** #158, #159, #160, #161, #172, #173

**Goal:** Wire edge-click-to-insert and verify palette drag-to-canvas in the SWF diagram, then file pages issues for long-term mixin refactoring.

**Architecture:** The SWF diagram already has picker infrastructure from DiagramBaseMixin (`_chooserState`, `_renderNodePicker`, `_onChooserSelect`). We add a `_pendingEdgeId` property on SwfDiagram to track edge-click context, override the chooser select/items/dismiss methods to dispatch `splitEdge` when an edge is targeted, and implement `getInsertableTypes` on the SWF edit policy to filter creatable types by grammar validity for the clicked edge.

**Tech Stack:** TypeScript, Lit, Vitest

## Global Constraints

- All components must have ARIA attributes (role, aria-label minimum)
- No pages mixin changes — interim fix is blocks-ui only
- `getInsertableTypes` already takes `(edge: GraphEdge, model: GraphModel)` on the EditPolicy interface — no signature change needed

---

## Batch 1: Edge click picker — policy + diagram wiring

### Task 1: Implement getInsertableTypes on SWF edit policy

**Files:**
- Modify: `packages/graph-stencil-swf/src/editing/swf-edit-policy.ts:73-75`
- Test: `packages/graph-stencil-swf/src/editing/swf-edit-policy.test.ts`

**Interfaces:**
- Consumes: `canSpliceOntoEdge(edge, node, model)` (same file, line 66-71), `CREATABLE_TYPES` (same file, line 5-18)
- Produces: `getInsertableTypes(edge: GraphEdge, model: GraphModel): StencilTypeInfo[]` — returns creatable types whose nodes can splice onto the given edge

- [ ] **Step 1: Write failing tests for getInsertableTypes**

Add to the `getInsertableTypes` describe block in `swf-edit-policy.test.ts`. Register all grammars needed for the new types. Build a linear pipeline model (start → call → end) and verify that insertable types on the call→end edge include task types but exclude non-deletable types.

```typescript
import { doGrammar } from '../stencils/do.js';
import { forkGrammar } from '../stencils/fork.js';
import { emitGrammar } from '../stencils/emit.js';
import { listenGrammar } from '../stencils/listen.js';
import { runGrammar } from '../stencils/run.js';
import { waitGrammar } from '../stencils/wait.js';
import { forGrammar } from '../stencils/for.js';
```

Add to `beforeAll`:
```typescript
registerGrammar(doGrammar);
registerGrammar(forkGrammar);
registerGrammar(emitGrammar);
registerGrammar(listenGrammar);
registerGrammar(runGrammar);
registerGrammar(waitGrammar);
registerGrammar(forGrammar);
```

Replace the existing `getInsertableTypes` describe block:
```typescript
describe('getInsertableTypes', () => {
  it('returns creatable types for a normal flow edge', () => {
    const s = node('s1', 'swf-start');
    const c = node('c1', 'swf-call');
    const e = node('e1', 'swf-end');
    const m = model([s, c, e], [
      { id: 'e1', source: 's1', target: 'c1' },
      { id: 'e2', source: 'c1', target: 'e1' },
    ]);
    const edge = m.edges.find(e => e.id === 'e2')!;
    const types = policy.getInsertableTypes(edge, m);
    const typeNames = types.map(t => t.type);
    expect(typeNames.length).toBeGreaterThan(0);
    expect(typeNames).toContain('swf-call');
    expect(typeNames).toContain('swf-set');
    expect(typeNames).toContain('swf-emit');
  });

  it('excludes non-deletable types from insertable list', () => {
    const s = node('s1', 'swf-start');
    const e = node('e1', 'swf-end');
    const m = model([s, e], [
      { id: 'e1', source: 's1', target: 'e1' },
    ]);
    const edge = m.edges[0]!;
    const types = policy.getInsertableTypes(edge, m);
    const typeNames = new Set(types.map(t => t.type));
    expect(typeNames.has('swf-start')).toBe(false);
    expect(typeNames.has('swf-end')).toBe(false);
    expect(typeNames.has('swf-entry')).toBe(false);
    expect(typeNames.has('swf-exit')).toBe(false);
    expect(typeNames.has('swf-root')).toBe(false);
  });

  it('returns all 12 creatable types for a standard edge', () => {
    const s = node('s1', 'swf-start');
    const c = node('c1', 'swf-call');
    const e = node('e1', 'swf-end');
    const m = model([s, c, e], [
      { id: 'e1', source: 's1', target: 'c1' },
      { id: 'e2', source: 'c1', target: 'e1' },
    ]);
    const edge = m.edges.find(e => e.id === 'e2')!;
    const types = policy.getInsertableTypes(edge, m);
    expect(types.length).toBe(12);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/graph-stencil-swf/src/editing/swf-edit-policy.test.ts`
Expected: FAIL — `getInsertableTypes` returns `[]`

- [ ] **Step 3: Implement getInsertableTypes**

In `swf-edit-policy.ts`, replace the `getInsertableTypes` method body:

```typescript
getInsertableTypes(edge: GraphEdge, model: GraphModel): StencilTypeInfo[] {
  return CREATABLE_TYPES.filter(info => {
    const candidate = { id: '__probe__', type: info.type, properties: {} };
    return policy.canSpliceOntoEdge?.(edge, candidate, model) ?? true;
  });
},
```

This creates a probe node for each creatable type and checks it against `canSpliceOntoEdge`. The probe node only needs `id` and `type` — `canSpliceOntoEdge` checks `NON_DELETABLE` and self-targeting, both type-based.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/graph-stencil-swf/src/editing/swf-edit-policy.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/graph-stencil-swf/src/editing/swf-edit-policy.ts packages/graph-stencil-swf/src/editing/swf-edit-policy.test.ts
git commit -m "feat(graph-stencil-swf): implement getInsertableTypes with grammar filtering

Refs #173

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Wire edge-click picker in SwfDiagram

**Files:**
- Modify: `components/swf-diagram/src/swf-diagram.ts`
- Test: `components/swf-diagram/src/swf-diagram.test.ts`

**Interfaces:**
- Consumes: `getInsertableTypes(edge, model)` from Task 1, `_chooserState` / `_renderNodePicker` / `_onChooserSelect` / `_chooserItems` / `_onChooserDismiss` from DiagramBaseMixin
- Produces: Edge click → picker → splitEdge mutation flow in SwfDiagram

- [ ] **Step 1: Write failing test for edge-click handler**

Add a new describe block in `swf-diagram.test.ts`. The test verifies that dispatching a `graph:edge:click` pages-event sets `_pendingEdgeId` and opens the chooser. Check the existing test file for fixture patterns first.

Read `components/swf-diagram/src/swf-diagram.test.ts` to understand the test setup before writing. The test needs to:

1. Create a SwfDiagram element
2. Set its `src` to a simple workflow YAML (start → call → end)
3. Wait for render
4. Dispatch a `pages-event` CustomEvent with `detail: { topic: 'graph:edge:click', payload: { edgeId: 'e2', edgeType: 'flow' } }`
5. Assert `(el as any)._pendingEdgeId === 'e2'`
6. Assert `(el as any)._chooserState !== null`

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run components/swf-diagram/src/swf-diagram.test.ts`
Expected: FAIL — `_pendingEdgeId` is undefined

- [ ] **Step 3: Add _pendingEdgeId state and event handler**

In `swf-diagram.ts`, add the state property near the top of the class:

```typescript
@state() private _pendingEdgeId: string | null = null;
```

Add the import for `state` from `lit/decorators.js` if not already present (it should be — the class uses `@state()` already).

In the `@pages-event` handler (lines 319-326), add the edge-click branch:

```typescript
if (topic === 'graph:edge:click') {
  if (!this.readonly) {
    this._pendingEdgeId = e.detail?.payload?.edgeId ?? null;
    if (this._pendingEdgeId) {
      this._chooserState = { x: this._lastPointerX, y: this._lastPointerY };
    }
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn vitest run components/swf-diagram/src/swf-diagram.test.ts`
Expected: PASS

- [ ] **Step 5: Write failing test for chooser select with splitEdge**

Test that when `_pendingEdgeId` is set and a chooser item is selected, the resulting YAML has the new node inserted at the edge position (not appended at the end).

```typescript
it('edge-click select dispatches splitEdge mutation', async () => {
  // Setup: create diagram with start → call → end
  // Set _pendingEdgeId to the call→end edge
  // Dispatch pages-palette-select event with a task type
  // Assert: YAML now has the new node between call and end
  // Assert: _pendingEdgeId is cleared
  // Assert: _chooserState is null
});
```

- [ ] **Step 6: Implement chooser overrides**

Override `_chooserItems` to filter by insertable types when an edge is targeted:

```typescript
protected override _chooserItems(): PaletteItem[] {
  if (!this._pendingEdgeId || !this._adapterResult) {
    return super._chooserItems();
  }
  const edge = this._adapterResult.model.edges.find(e => e.id === this._pendingEdgeId);
  if (!edge) return super._chooserItems();
  const policy = this._editPolicy();
  const insertable = policy.getInsertableTypes(edge, this._adapterResult.model);
  return insertable.map(info => ({ type: info.type, label: info.label, icon: info.icon }));
}
```

Override `_onChooserSelect` to dispatch `splitEdge` when an edge is targeted:

```typescript
protected override _onChooserSelect = (e: Event): void => {
  if (this._pendingEdgeId) {
    const detail = (e as CustomEvent).detail;
    const nodeType = detail?.item?.type as string | undefined;
    if (nodeType) {
      this._handleMutation({ type: 'splitEdge', edgeId: this._pendingEdgeId, insertNodeType: nodeType });
    }
    this._pendingEdgeId = null;
    this._chooserState = null;
    return;
  }
  super._onChooserSelect(e);
};
```

Override `_onChooserDismiss` to clear edge state:

```typescript
protected override _onChooserDismiss = (): void => {
  this._pendingEdgeId = null;
  this._chooserState = null;
};
```

Add `PaletteItem` to the imports — it comes from `@casehubio/pages-diagram-palette` (same package the mixin imports from). Check the existing imports in the file and add if not present.

- [ ] **Step 7: Run tests to verify they pass**

Run: `yarn vitest run components/swf-diagram/src/swf-diagram.test.ts`
Expected: PASS

- [ ] **Step 8: Run full test suite**

Run: `yarn test`
Expected: PASS — no regressions

- [ ] **Step 9: Commit**

```bash
git add components/swf-diagram/src/swf-diagram.ts components/swf-diagram/src/swf-diagram.test.ts
git commit -m "feat(swf-diagram): wire edge-click picker with splitEdge mutation

Clicking an edge shows the node picker filtered by getInsertableTypes.
Selecting a type dispatches splitEdge to insert the node at that position.

Refs #173

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 2: Palette drag verification + pages issues

### Task 3: Verify palette drag-to-canvas end-to-end

**Files:**
- Test: `components/swf-diagram/src/swf-diagram.test.ts`
- Possibly modify: `components/swf-diagram/src/swf-diagram.ts` (if gaps found)

**Interfaces:**
- Consumes: `getAddPlacement` from swf-edit-policy, `_handlePaletteSelect` override in swf-diagram, GraphCanvas drop handler
- Produces: Verified drag-to-canvas flow, test coverage

- [ ] **Step 1: Write integration test for palette click-to-add with splitEdge**

Test that clicking a palette item on a linear pipeline (start → call → end) inserts before end via `getAddPlacement`:

```typescript
it('palette select uses getAddPlacement to insert before end', async () => {
  // Setup: create diagram with start → call → end (linear)
  // Dispatch pages-palette-select with type 'swf-set'
  // Assert: YAML has set node between call and end (splitEdge path)
});
```

- [ ] **Step 2: Run test to verify current behaviour**

Run: `yarn vitest run components/swf-diagram/src/swf-diagram.test.ts`
Expected: PASS (palette select with getAddPlacement already works per swf-diagram.ts:142-154)

- [ ] **Step 3: Write test for palette drag-drop event**

Test that a `graph:palette:drop` event doesn't break anything (it's informational — fires after mutation):

```typescript
it('graph:palette:drop event does not throw', async () => {
  // Setup: create diagram
  // Dispatch pages-event with topic 'graph:palette:drop' and payload { nodeType: 'swf-call', x: 100, y: 100 }
  // Assert: no error, diagram still renders
});
```

- [ ] **Step 4: Run tests**

Run: `yarn vitest run components/swf-diagram/src/swf-diagram.test.ts`
Expected: PASS

- [ ] **Step 5: Run full test suite and typecheck**

Run: `yarn test && yarn typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add components/swf-diagram/src/swf-diagram.test.ts
git commit -m "test(swf-diagram): verify palette drag-to-canvas and click-to-add paths

Refs #173

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 4: File pages issues with design sketches

**Files:**
- No code files — GitHub issue creation only

**Interfaces:**
- Produces: Two issues on casehub-pages with design sketches

- [ ] **Step 1: File Issue A — Layout hook (_computeLayout)**

```bash
gh issue create --repo casehubio/casehub-pages --title "DiagramBaseMixin: add _computeLayout hook for subclass layout strategies" --body "$(cat <<'EOF'
## Problem

DiagramBaseMixin hardcodes `computeElkLayout` in `_fullRender` (diagram-base-mixin.ts:405). Subclasses that need a different layout strategy (e.g. SWF stack-column layout) must override `_fullRender` entirely, copy-pasting the render pipeline.

## Design Sketch

Add a protected method that subclasses override:

```typescript
protected async _computeLayout(
  model: GraphModel,
  options: LayoutOptions
): Promise<LayoutResult> {
  return computeElkLayout(model, options);
}
```

`_fullRender` calls `this._computeLayout(...)` instead of `computeElkLayout(...)` directly. SWF overrides only `_computeLayout` to return `computeSwfStackLayout(model)`. No other changes needed.

## Impact

Eliminates full `_fullRender` overrides in swf-diagram, and any future diagram that needs a custom layout strategy.

## Related

- casehubio/blocks-ui#173 — interim workaround (full `_fullRender` override)
- casehubio/blocks-ui#170 — stack-column layout improvements
EOF
)"
```

- [ ] **Step 2: File Issue B — Standard render template with event wiring**

```bash
gh issue create --repo casehubio/casehub-pages --title "DiagramBaseMixin: standard render template with default event wiring" --body "$(cat <<'EOF'
## Problem

The mixin provides no `render()` — each diagram assembles its own template with manual event wiring. SWF, case, and org diagrams all have near-identical render methods (~70 lines each) with the same toolbar/palette/canvas/properties/picker structure. New events (edge-click, palette-drop) must be wired in every diagram.

## Design Sketch

Add `_renderDiagramShell()` to the mixin with `_handleCanvasEvent` dispatching all graph events:

```typescript
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

protected _handleCanvasEvent(e: CustomEvent): void {
  const topic = e.detail?.topic;
  switch (topic) {
    case 'graph:node:click': this._handleNodeClick(e); break;
    case 'graph:edge:click': this._handleEdgeClick(e); break;
    case 'graph:selection:change': this._handleSelectionChange(e); break;
    case 'graph:pane:click': this._showPickerAtPaneClick(); break;
    case 'graph:connect:end-on-empty': this._showPickerAtConnectEnd(e.detail?.payload); break;
  }
}

protected _handleEdgeClick(e: CustomEvent): void {
  // default: show picker with splitEdge
}
```

Each diagram's `render()` calls `this._renderDiagramShell()` and adds only domain-specific elements.

## Impact

- New graph events are wired once in the mixin, not per-diagram
- Diagram components shrink to adapter + edit policy + stencils
- Consistent event handling across all diagram types

## Related

- casehubio/blocks-ui#173 — interim per-diagram event wiring
EOF
)"
```

- [ ] **Step 3: Record issue numbers and commit**

Note the issue numbers returned by `gh issue create`. No code to commit for this task — just verify the issues were created successfully.

```bash
gh issue list --repo casehubio/casehub-pages --limit 2 --state open --search "DiagramBaseMixin"
```

## References

- [2026-09-25-swf-edge-click-palette-drag-design.md] — design spec this plan implements
- [swf-edit-policy.ts:66-90] — canSpliceOntoEdge, getInsertableTypes, getAddPlacement
- [swf-diagram.ts:142-154] — _handlePaletteSelect override
- [swf-diagram.ts:319-326] — @pages-event handler switch
- [diagram-base-mixin.ts:239-337] — picker infrastructure
- [GraphCanvas.ts:573-578] — graph:edge:click emission
- [GraphCanvas.ts:361-379] — palette drop handler
- [types.ts:20-28] — EditPolicy interface (getInsertableTypes already has edge param)
- [GitHub #173] — focal issue
- [D10, D11] — decisions (decisions.md)
