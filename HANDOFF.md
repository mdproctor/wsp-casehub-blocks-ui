# HANDOFF — casehub-blocks-ui

## Last Session

Brainstormed, designed, and implemented #161 — JCEF split editor for
IntelliJ plugin. The plugin now shows a native YAML editor alongside a
JCEF panel rendering case definition diagrams when opening `.case.yaml`
files. Nodes, edges, and ELK layout render correctly. Theme sync maps
IntelliJ dark/light to `--pages-*` CSS vars. Editor edits push YAML to
diagram in real time (debounced 150ms).

Fixed a pre-existing regression in pages (`PagesGraphCanvas` — pages#452):
the component only supported data-source mode but casehub-diagram passes
nodes/edges directly. Also fixed Vite `?raw` CSS imports in the esbuild
IIFE bundle — ReactFlow's base CSS was missing, causing invisible edges.

CI workflow added for plugin distribution (zip artifact on main push).

## Immediate Next Step

Fix bundled stencil palette and property panel in the IIFE diagram bundle.
The `<pages-diagram-palette>` web component renders empty in the bundle —
its content elements aren't initializing. Same issue likely affects the
property palette. The examples gallery (Vite dev server) works because
imports resolve individually; the IIFE bundle needs all page component
registrations to be included and initialized.

Debug approach: check if `pages-diagram-palette` custom element is
registered in the IIFE context, verify its dependencies are bundled,
and trace why its render produces empty content.

Also: selection highlight offset (a few pixels top-left) needs CSS
investigation.

## Cross-Module

- pages#452: PagesGraphCanvas direct property pass-through — committed
  to pages main, needs push to remote.
- pages workbench extensibility SPI — design spec written, pages issue
  to be filed for `WorkbenchFormatRegistration` interface refactor.
- DiagramBaseMixin `yaml-changed` event — needed for diagram→editor sync
  (Phase 2). Requires pages PR to add getter/setter on `_currentYaml`.

## References

- Spec: `specs/issue-158-lsp-schema-refinements/2026-09-17-domain-schema-assembly-design.md`
- Plan: `plans/2026-09-17-jcef-split-editor.md`
- Decisions: `specs/issue-158-lsp-schema-refinements/decisions.md`
- Pages issue: casehubio/casehub-pages#452
