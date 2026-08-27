# Decisions — #140 Wire Pages Palettes

## D1: Remove old form utilities

**Choice:** Remove `form/` directory from diagram-core (field-renderer, validation, trigger-editor, nested-group, property-form) and `diagram-properties` component. pages-property-palette replaces all of it.
**Alternatives:**
- Deprecate only — adds noise, no consumer needs them after migration
- Keep both — confusing for contributors, dead code
**Rationale:** pages-property-palette covers every case the old form utilities handled, plus grouping, validation, custom editors, and advanced toggle. No consumer will need the old path after this branch.
**Trade-offs:** Any external consumer (unlikely — pre-release) using diagram-properties directly would break.
**Sources:** packages/diagram-core/src/form/, pages-property-palette SPI
**Exploration:** quick
**Status:** captured

## D2: EditorResolver location

**Choice:** DiagramBaseMixin exposes a protected method `_editorResolver()` that subclasses override. The mixin passes the resolver to pages-property-palette. Domain-specific editors stay in their stencil packages per PP-20260806-320d50 (stencil isolation).
**Alternatives:**
- Shared global resolver in diagram-core — violates stencil isolation, forces diagram-core to know about domain editors
- Per-component inline — duplicates wiring across casehub-diagram and swf-diagram
**Rationale:** Protected method override follows the existing DiagramBaseMixin pattern (_adaptYaml, _applyPropertyEdit, _paletteTypes, _emptyTemplate). Each subclass wires its own editors without cross-package imports.
**Trade-offs:** Each new diagram type must implement _editorResolver(). Minor boilerplate.
**Sources:** diagram-base-mixin.ts, PP-20260806-320d50
**Exploration:** quick
**Status:** captured

## D3: Add-node flow through EditPolicy

**Choice:** Palette fires `pages-palette-select` → DiagramBaseMixin creates `GraphEdit.addNode` → EditPolicy validates → `applyGraphEdit` executes → domain adapter handles YAML mutation.
**Alternatives:**
- Direct addElement — bypasses EditPolicy, validation rules in two places
- Hybrid — incremental but inconsistent; two code paths for creation vs connection
**Rationale:** EditPolicy is the single source of truth for what mutations are valid. Routing all edits (add, connect, delete) through it ensures consistency. The existing addElement() becomes the domain adapter's response to a GraphEdit, not the entry point.
**Trade-offs:** More wiring than direct addElement. Worth it for consistency.
**Sources:** graph-renderer/src/editing/types.ts (EditPolicy, GraphEdit), casehub-diagram addElement()
**Exploration:** quick
**Depends on:** D2 (resolver on mixin means palette is also wired via mixin)
**Status:** captured

## D4: Showcase strategy

**Choice:** Update existing showcase pages (casehub-diagram-page, swf-diagram-page, diagram-workbench-page) to demonstrate property palette, stencil palette, and edge creation in context.
**Alternatives:**
- Dedicated editor page — adds a new page but readers see features in isolation
- Both — more coverage but more maintenance
**Rationale:** The editing UX makes sense in context of the full diagram, not isolated. Updating existing pages shows the real workflow.
**Trade-offs:** Existing pages get more complex. Acceptable — they're showcase pages.
**Sources:** examples/src/pages/casehub-diagram-page.ts, swf-diagram-page.ts, diagram-workbench-page.ts
**Exploration:** quick
**Status:** captured
