# Domain Schema Assembly for IntelliJ LSP Plugin

**Issue:** #161
**Branch:** issue-158-lsp-schema-refinements
**Date:** 2026-09-17

## Overview

Extend the CaseHub YAML IntelliJ plugin beyond pure LSP text intelligence to include visual diagram editing, make the web-based YAML workbench extensible for domain formats, and add CI distribution.

Three concerns:

1. **IntelliJ visual editor** — split editor with native YAML editor + JCEF visual diagram panel, synced bidirectionally
2. **Extensible YAML workbench** (pages dependency) — `pages-builder-shell` accepts format registrations; domain formats plug in visual renderers
3. **CI pipeline** — plugin zip as a downloadable GitHub Actions artifact

Target architecture: native-first hybrid. Maximize IntelliJ-native UI (text editor, tree, properties). JCEF only for the visual diagram where no native equivalent exists.

## Architecture

### Current State

The IntelliJ plugin (`plugins/intellij-casehub/`) is a thin LSP client shell:

- `CaseHubYamlLanguage` — custom language dialect extending YAML
- `CaseHubYamlFileType` — routes `*.page.yaml`, `*.case.yaml`, `*.swf.yaml`, `*.htn.yaml`, `*.org.yaml`
- `CaseHubLspServerDescriptor` — extracts and launches the bundled Node.js LSP server via LSP4IJ
- LSP server (`packages/lsp-schemas/`) — schema registry with format registrations for all five formats, esbuild-bundled as `server-node.bundle.cjs`

This provides: schema-driven completion, diagnostics, hover, rename, find references, go-to-definition. No visual rendering.

The web-based workbench (`pages-builder-shell` in pages) provides tree + CodeMirror editor + visual preview for `*.page.yaml` only. Domain formats (case/swf/htn/org) have diagram web components in blocks-ui but no workbench integration.

### Target State

```
IntelliJ Plugin
┌─────────────────┬──────────────────────┬──────────────────┐
│  Structure View  │  Native YAML Editor  │  JCEF Diagram    │
│  (native tree)   │  (CaseHubYAML + LSP) │  (web component) │
│                  │                      │                  │
│  PSI-derived     │  Existing, untouched │  Format-specific │
│  domain-aware    │                      │  visual renderer │
└─────────────────┴──────────────────────┴──────────────────┘
                            ↕ sync
              Editor → Diagram: full YAML push
              Diagram → Editor: CST delta patches

Web Workbench (pages-builder-shell)
┌──────────┬──────────────────┬──────────────────┐
│  Tree    │  CodeMirror       │  Visual Editor   │
│  (Zod-   │  Editor           │  (pluggable per  │
│  derived)│                   │   format)        │
└──────────┴──────────────────┴──────────────────┘
         Format registration: Zod schema + visual component tag
```

### Split Editor — TextEditorWithPreview

IntelliJ's `TextEditorWithPreview` API provides a split pane: native text editor on the left, custom preview panel on the right. Used by Markdown, AsciiDoc, and similar plugins.

A `CaseHubFileEditorProvider` detects file type by extension, wraps the native YAML editor, and creates a `JBCefBrowser` panel rendering the appropriate diagram web component. The provider is registered in `plugin.xml` for CaseHubYAML files.

### JCEF Diagram Panel

The diagram web components (LitElement) are bundled via esbuild into `diagram-panel.bundle.js` — the same pattern as the LSP server bundle. A small HTML shell in plugin resources loads the bundle. The Kotlin side creates a `JBCefBrowser`, loads the HTML from plugin resources via `file://` protocol.

Communication uses `CefMessageRouter`:
- **Kotlin → JS**: `cefBrowser.cefBrowser.executeJavaScript()` to push YAML content
- **JS → Kotlin**: `CefMessageRouterHandler` receives messages (cursor position, CST edit deltas)

### Sync Protocol

Asymmetric — optimized for each direction:

**Editor → Diagram (rendering):** `DocumentListener` fires on text changes. Full YAML string pushed to JCEF via `executeJavaScript()`. Debounced (~150ms) to avoid noise during rapid typing. The diagram component accepts YAML as a property and handles re-parse/re-render internally (components already diff efficiently).

**Diagram → Editor (structural edits):** The diagram uses the `yaml` library's CST API to compute minimal text patches. Each edit produces a delta: `{ offset: number, length: number, newText: string }`. The JCEF panel sends the delta to Kotlin via `CefMessageRouter`. Kotlin applies it as `document.replaceString(offset, offset + length, newText)` inside a `WriteAction`. This preserves comments, blank lines, and custom spacing — only the changed characters are modified.

**Diagram → Editor (cursor sync):** When the user clicks a node in the diagram, the JCEF panel sends the YAML path (e.g., `spec.workers[1].name`). Kotlin resolves the path to a document offset via the PSI tree and moves the caret.

### Format Routing

The `CaseHubFileEditorProvider` maps file extension to diagram component:

| Extension | Diagram Component |
|-----------|-------------------|
| `.case.yaml` | `casehub-diagram` (read-only case flow) |
| `.swf.yaml` | `swf-diagram` |
| `.htn.yaml` | `blocks-dag-viewer` |
| `.org.yaml` | `org-diagram` |
| `.page.yaml` | Page preview (via `renderPreview` callback pattern) |

This mapping is a static config in the provider — one entry per format.

### Extensible Workbench SPI (Pages)

`pages-builder-shell` becomes format-agnostic. The format registration is minimal:

```typescript
interface WorkbenchFormatRegistration {
  formatId: string;
  schema: z.ZodType;              // Zod schema — tree, properties, fragment rules derived at runtime
  visualElement: string;           // Custom element tag name for the visual column
  extensions: string[];            // File extensions this format handles
}
```

The workbench derives everything else from the Zod schema via Zod 4 introspection:
- **Tree structure**: `z.array()` → expandable collection node, `z.object()` → leaf with named keys
- **Property forms**: `z.string()` → text input, `z.enum([...])` → dropdown, `z.number().min().max()` → number with range, `z.boolean()` → toggle, `z.string().datetime()` → date picker
- **Fragment rules**: Array membership determines valid paste targets — a node serialized from `spec.capabilities[0]` can paste into any `spec.capabilities[]` slot

The existing `pageFormat` becomes the first consumer of this SPI, proving the abstraction works. Page-specific logic (PageDocument, dashboard schema, component catalog) moves into a `pageFormat` registration.

### Native Structure View (Phase 2)

A custom `PsiStructureViewFactory` for CaseHubYAML. Derives tree structure from the YAML PSI tree directly (no Zod access in Kotlin). Provides domain-aware node presentation:

- Case definitions: Capabilities, Workers, Bindings as top-level nodes
- SWF: Task list with type badges
- Org: Units with parent-child hierarchy
- HTN: Compound/primitive task tree

### Native Property Panel (Phase 2)

Schema-driven property editing in IntelliJ's native UI. The LSP server can expose schema metadata via a custom LSP request — given a YAML path, return the Zod schema fragment describing that node's properties. Kotlin renders the form using IntelliJ's `DialogPanel` DSL with typed editors (combo boxes, spinners, date pickers).

### Structural Editing (Phase 3)

Clipboard-based structural editing following the `BuilderClipboard` pattern from pages. In IntelliJ, the tree (native Structure view) handles cut/copy/paste via IntelliJ's native clipboard and undo system. The JCEF diagram handles its own structural operations and syncs back via CST delta patches (D7).

Validation rules (fragment type → valid targets) are expressed as data (JSON config per format) consumed by both the Kotlin and TypeScript implementations. The clipboard format is serialized YAML fragments — portable across both runtimes.

### CI Pipeline

A new GitHub Actions job in `.github/workflows/ci.yml`:

```yaml
intellij-plugin:
  runs-on: ubuntu-latest
  needs: build-and-test    # lsp-schemas bundle must build first
  steps:
    - Checkout
    - Setup Java 21
    - Setup Node.js 22 + Corepack
    - yarn install + yarn build (produces lsp-schemas bundle)
    - yarn --cwd packages/lsp-schemas build:bundle
    - Gradle build in plugins/intellij-casehub/
    - Upload plugins/intellij-casehub/build/distributions/*.zip as artifact
```

Triggers: main pushes only (paths include `plugins/intellij-casehub/**` and `packages/lsp-schemas/**`). PRs verify the Gradle build succeeds but do not upload artifacts.

## Delivery Phases

### Phase 1 — JCEF split editor + CI (blocks-ui only)

No pages dependency. Can proceed immediately.

1. `CaseHubFileEditorProvider` — `TextEditorWithPreview` with native YAML editor + JCEF panel
2. `diagram-panel.bundle.js` — esbuild bundle of diagram web components + HTML shell
3. `CaseHubDiagramPanel` — Kotlin JCEF wrapper with `CefMessageRouter` bridge
4. Sync: `DocumentListener` → debounced YAML push to JCEF
5. Format routing: file extension → diagram component
6. Gradle `copyDiagramBundle` task (mirrors existing `copyServerBundle`)
7. CI job: build plugin zip, upload as artifact on main push
8. Start with one format (`.case.yaml` → `casehub-diagram`) to prove the integration, then wire remaining formats

### Phase 2 — Extensible workbench + native IntelliJ panels (pages dependency)

Requires pages to land the workbench SPI first.

**Pages side (separate issue):**
1. Define `WorkbenchFormatRegistration` interface
2. Refactor `pages-builder-shell` to accept format registrations
3. Extract page-specific logic into a `pageFormat` registration
4. Schema-driven tree derivation from Zod introspection
5. Schema-driven property form derivation from Zod introspection
6. Generalize `BuilderClipboard` fragment rules to work with any Zod schema

**Blocks-ui side (after pages lands):**
1. Register domain formats with the workbench SPI (case, swf, htn, org)
2. Custom `PsiStructureViewFactory` for domain-aware tree in IntelliJ
3. Native property panel via custom LSP request + `DialogPanel` DSL
4. Wire diagram web components as visual column in the web workbench

### Phase 3 — Structural editing (blocks-ui, after phase 2)

1. IntelliJ: structural clipboard operations in native Structure view
2. Fragment validation rules as shared JSON config per format
3. Clipboard bridge between IntelliJ's native clipboard and JCEF diagram
4. Insert mode / guided paste targets in native tree

## Cross-Repo Coordination

### Pages Issue Spec

File a pages issue for the workbench extensibility SPI with this contract:

**Title:** feat: extensible workbench — format registration SPI for builder-shell

**Scope:**
- `WorkbenchFormatRegistration` interface: `formatId`, `schema` (Zod), `visualElement` (tag name), `extensions`
- Refactor `pages-builder-shell` to consume registrations instead of hardcoding `PageDocument`/`dashboardSchema`
- Page format becomes first consumer: `pageFormat` registration
- Tree derivation from Zod schema (array → collection, object → leaf)
- Property form derivation from Zod schema (string/enum/number/boolean/datetime → typed editors)
- Generalized `BuilderClipboard` fragment rules — array membership determines valid targets
- All existing page builder behavior must be preserved (regression test)

**Consumer:** blocks-ui domain formats (case/swf/htn/org) will register via this SPI

**Not in scope:** IntelliJ integration (handled by blocks-ui), domain-specific visual renderers (already exist in blocks-ui)

## Files to Create/Modify

### Phase 1

| File | Change |
|------|--------|
| `plugins/intellij-casehub/src/main/kotlin/.../CaseHubFileEditorProvider.kt` | **New** — TextEditorWithPreview provider |
| `plugins/intellij-casehub/src/main/kotlin/.../CaseHubDiagramPanel.kt` | **New** — JCEF wrapper + CefMessageRouter bridge |
| `plugins/intellij-casehub/src/main/kotlin/.../DiagramSyncListener.kt` | **New** — DocumentListener with debounced YAML push |
| `plugins/intellij-casehub/src/main/resources/diagram/diagram-shell.html` | **New** — HTML shell for JCEF |
| `plugins/intellij-casehub/src/main/resources/META-INF/plugin.xml` | Add FileEditorProvider registration |
| `plugins/intellij-casehub/build.gradle.kts` | Add `copyDiagramBundle` task |
| `packages/lsp-schemas/build-diagram-bundle.js` | **New** — esbuild script for diagram components |
| `.github/workflows/ci.yml` | Add `intellij-plugin` job |

### Phase 2 (blocks-ui side only)

| File | Change |
|------|--------|
| `plugins/intellij-casehub/src/main/kotlin/.../CaseHubStructureViewFactory.kt` | **New** — domain-aware Structure view |
| `plugins/intellij-casehub/src/main/kotlin/.../CaseHubPropertyPanel.kt` | **New** — schema-driven property editor |
| `packages/lsp-schemas/src/formats/*.ts` | Add workbench registration alongside existing FormatRegistration |

## Testing Strategy

- Unit tests for format routing (extension → component mapping)
- Unit tests for CST delta generation (YAML edit → minimal patch)
- Integration test: JCEF panel loads and renders diagram from YAML input
- Gradle build verification in CI (plugin zip is valid)
- Manual verification: open `.case.yaml` in IntelliJ → split editor shows diagram
- Regression: existing LSP features (completion, diagnostics, hover) unaffected

## References

- `plugins/intellij-casehub/` — existing IntelliJ plugin (LSP shell)
- `packages/lsp-schemas/` — domain format schemas and LSP server
- `packages/lsp-schemas/build-bundle.js` — existing esbuild pattern
- `pages/packages/pages-builder/src/shell/builder-shell.ts` — existing workbench (to be made extensible)
- `pages/docs/specs/issue-439-dock-workbench-polish/2026-09-17-structural-editing-design.md` — structural editing clipboard spec
- `components/casehub-diagram/`, `components/swf-diagram/`, `components/org-diagram/` — diagram web components
- `packages/graph-stencil-swf/` — `applySwfPropertyEdit` (CST-preserving edit pattern)
- `pages/packages/pages-builder/src/shell/diff-patch.ts` — `computeMinimalChanges` (diff-patch pattern)
- IntelliJ Platform SDK: `TextEditorWithPreview`, `JBCefBrowser`, `CefMessageRouter`, `PsiStructureViewFactory`
- D1-D3 (prior session) — schema generation decisions
- D4-D7 — architecture, workbench SPI, JCEF bundling, sync protocol
