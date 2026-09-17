# Domain Schema Assembly for IntelliJ LSP Plugin

**Issue:** #161
**Branch:** issue-158-lsp-schema-refinements
**Date:** 2026-09-17

## Overview

**Relationship to issue-407:** This spec is the Batch 4 design pass deferred in the [LSP Server + IDE Plugins spec](../../docs/specs/issue-407-lsp-ide-plugins/2026-09-10-lsp-ide-plugins-design.md). That spec explicitly defers "Diagram Webview" as Batch 4 with "Separate design pass covering: bidirectional sync model, webview lifecycle, theme sync, component bundling, VS Code `WebviewPanel` vs IntelliJ `JBCefBrowser` tradeoffs." Issue #161 is scoped under the same #158 LSP schema refinements umbrella and delivers the IntelliJ portion of the deferred Batch 4 work. Where this spec overlaps with issue-407 (layered architecture, format registrations, file type detection), issue-407 remains authoritative — this spec adds only the diagram webview, workbench extensibility, and CI concerns that issue-407 deferred.

**VS Code scope:** This spec covers IntelliJ only. VS Code's `WebviewPanel` has different constraints — stricter CSP, message-passing serialization, no direct DOM access — that require a separate design pass. A follow-up spec for the VS Code diagram webview will be filed as a separate issue. The diagram web components and esbuild bundling strategy are shared infrastructure; the hosting, lifecycle, and sync bridging are IDE-specific.

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
              Editor → Diagram: full YAML push (debounced)
              Diagram → Editor: bridge-computed minimal patches

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

**JCEF availability fallback:** JCEF is not available in all environments — Remote Development (Gateway, SSH, WSL), headless/test mode, and some Linux configurations with missing Chromium dependencies. The provider checks `JBCefApp.isSupported()` before creating the split editor. When JCEF is unavailable, it falls back to the native text editor alone (retaining full LSP intelligence) and shows a one-time notification explaining that the visual diagram panel requires a local IDE. This is the same pattern used by IntelliJ's Markdown plugin.

**Webview lifecycle management:** `JBCefBrowser` requires explicit lifecycle management to prevent resource leaks:

1. **Disposal:** `CaseHubDiagramPanel` implements `Disposable`. The `JBCefBrowser` instance is registered with `Disposer.register(parentDisposable, browser)` where the parent is the `FileEditor`. When the editor tab is closed, the Chromium process is terminated. Failure to dispose leaks Chromium processes.
2. **Tab visibility:** When the user switches to a different editor tab, the JCEF panel is no longer visible but continues consuming memory and CPU. A `FileEditorManagerListener.selectionChanged()` callback pauses the debounced document push timer and suspends diagram re-renders when the editor loses focus. Rendering resumes with a single full push when the tab regains focus.
3. **Split editor mode switching:** `TextEditorWithPreview` supports three modes — editor-only, split, and preview-only. When the user switches to editor-only mode, the JCEF browser is disposed to free resources. On switch back to split or preview mode, a new `JBCefBrowser` is created and initialized with the current document content. The `TextEditorWithPreview.getLayout()` callback detects mode changes.

### JCEF Diagram Panel

The diagram web components (LitElement) are bundled via esbuild into `diagram-panel.bundle.js` — the same pattern as the LSP server bundle. A small HTML shell in plugin resources loads the bundle. The Kotlin side creates a `JBCefBrowser`, loads the HTML from plugin resources via `file://` protocol.

**Bundle composition and size:** The diagram components share heavy dependencies — React, ReactFlow, ELK layout engine (via `@casehubio/graph-renderer`), Lit 3 (via `@casehubio/pages-data`), and the `yaml` library. These shared dependencies dominate the bundle. Phase 1 bundles only `casehub-diagram` (one format), so per-format splitting is not yet needed. When remaining formats are wired, a monolithic bundle including all four formats shares the common dependency tree — the format-specific stencil code is small relative to React+ReactFlow+ELK. The bundle size target is ≤3MB gzipped; if the monolithic bundle exceeds this after wiring all formats, per-format code-splitting via esbuild `splitting: true` is the fallback. Since the JCEF panel loads from local plugin resources (`file://` protocol), network transfer is not a concern — only plugin distribution size and initial parse time matter.

**Theme sync:** The diagram components use `--pages-*` CSS custom properties (`--pages-surface-color`, `--pages-border-color`, `--pages-text-color`, `--pages-accent-color`, etc.) with light-themed fallback values. Without theme sync, the diagram renders with hardcoded light colors regardless of IntelliJ's theme — making it unreadable in Darcula/dark modes.

1. **Detection:** Register a `LafManagerListener` callback. `lookAndFeelChanged()` fires when IntelliJ switches themes.
2. **Mapping:** A static table maps IntelliJ UIManager colors to `--pages-*` CSS custom properties — e.g., `UIManager.getColor("Panel.background")` → `--pages-surface-color`, `UIManager.getColor("Label.foreground")` → `--pages-text-color`. The mapping covers the ~15 CSS custom properties used by diagram components.
3. **Application:** Inject a `<style>:root { ... }` element via `executeJavaScript()` with the resolved CSS custom property values.
4. **Initial render:** The HTML shell template includes the theme CSS computed at `JBCefBrowser` creation time — before loading the diagram bundle. No flash of wrong-theme content on first load. Subsequent theme switches re-inject the style element.

Communication uses `CefMessageRouter`:
- **Kotlin → JS**: `cefBrowser.cefBrowser.executeJavaScript()` to push YAML content and theme updates
- **JS → Kotlin**: `CefMessageRouterHandler` receives messages (cursor position, edit deltas)

### Sync Protocol

Asymmetric — optimized for each direction:

**Editor → Diagram (rendering):** `DocumentListener` fires on text changes. Full YAML string pushed to JCEF via `executeJavaScript()`. Debounced (~150ms) to avoid noise during rapid typing. The diagram component accepts YAML as a property and handles re-parse/re-render internally (components already diff efficiently).

**Diagram → Editor (structural edits):** All diagram components use `DiagramBaseMixin`, which stores `_currentYaml: string` as internal state. Edit methods (`_applyPropertyEdit`, `_applyGraphEdit`, `switch*` functions) use the `yaml` library's CST API internally for CST-preserving edits but return a **complete new YAML string** — not a delta. The JCEF bridge layer captures the old and new YAML strings and computes minimal text patches using a diff algorithm (same approach as `computeMinimalChanges` in `pages-builder/diff-patch.ts`). Each edit produces a delta: `{ offset: number, length: number, newText: string }`. The bridge sends the delta to Kotlin via `CefMessageRouter`. Kotlin applies it as `document.replaceString(offset, offset + length, newText)` inside a `WriteAction`. This preserves comments, blank lines, and custom spacing — only the changed characters are modified. The diagram components remain unchanged; the diffing concern is localized to the bridge layer.

**Echo suppression:** Bidirectional sync creates an echo loop: a diagram edit sends a delta to Kotlin → Kotlin applies `document.replaceString()` → `DocumentListener.documentChanged()` fires → attempts to push the full YAML back to JCEF → `DiagramBaseMixin.updated()` resets `_undoStack`, `_redoStack`, and `_selectedNodeId`, destroying undo history and selection.

The `DiagramSyncListener` uses an **origin flag** to suppress echoes. Before applying a JCEF-originated delta, set `suppressEcho = true`. The `DocumentListener.documentChanged()` callback checks this flag — if true, skip the push and return. The flag is reset in a `finally` block after the `WriteAction` completes. All operations run on the EDT (Event Dispatch Thread), so no race condition exists. This is the standard IntelliJ pattern for bidirectional editor sync (used by Markdown, AsciiDoc, and database tool plugins).

**Diagram → Editor (cursor sync):** When the user clicks a node in the diagram, the JCEF panel sends the YAML path (e.g., `spec.workers[1].name`). Kotlin resolves the path to a document offset via the PSI tree and moves the caret.

### Format Routing

The `CaseHubFileEditorProvider` maps file extension to diagram component:

| Extension | Diagram Component |
|-----------|-------------------|
| `.case.yaml` | `casehub-diagram` (full case editor with graph editing, property palette, undo/redo) |
| `.swf.yaml` | `swf-diagram` |
| `.htn.yaml` | `htn-diagram` |
| `.org.yaml` | `blocks-org-diagram` |
| `.page.yaml` | Page preview (via `renderPreview` callback pattern) |

This mapping mirrors the canonical `DIAGRAM_TAGS` record in `blocks-diagram-workbench` (`components/diagram-workbench/src/diagram-workbench.ts`), which already maps format → component tag for runtime routing (`{swf: 'swf-diagram', case: 'casehub-diagram', htn: 'htn-diagram'}`). The JCEF panel does NOT embed `blocks-diagram-workbench` — that component is case-centric with drill-down navigation (case → embedded SWF/HTN), designed for runtime case exploration. The JCEF panel opens individual format files directly (e.g., `.swf.yaml` shows `swf-diagram` at the top level), which the workbench's hardcoded case root level doesn't support. The format → tag mapping will be extracted to a shared constant in `blocks-ui-core` to avoid duplication between the workbench and the provider. The shared constant adds `org: 'blocks-org-diagram'` — the existing `DIAGRAM_TAGS` in `blocks-diagram-workbench` only has three entries (swf, case, htn) because the workbench doesn't handle org diagrams. The extraction is not a straight copy.

### Extensible Workbench SPI (Pages)

`pages-builder-shell` becomes format-agnostic. The format registration is minimal:

```typescript
interface WorkbenchFormatRegistration {
  formatId: string;                // Must match a FormatRegistration.formatId in the schema registry
  visualElement: string;           // Custom element tag name for the visual column
}
```

**Relationship to `FormatRegistration`:** The existing `FormatRegistration` in `@casehubio/pages-lsp` (`pages-lsp/src/types.ts`) already carries `formatId`, `extensions`, and `documentSchema` (Zod) for each format — all four domain formats are already registered in `packages/lsp-schemas/src/formats/`. `WorkbenchFormatRegistration` deliberately does NOT duplicate these fields. Instead, the workbench resolves the Zod schema and extensions from the `SchemaRegistry` using the `formatId`. This eliminates the maintenance liability of two registration interfaces with overlapping fields that could diverge.

The workbench derives everything else from the Zod schema (obtained via the registry) using Zod 4 introspection:
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

**UX rationale:** The native Structure view and JCEF diagram serve different interaction models. The Structure view provides keyboard-driven tree editing — speed search, Ctrl+F12 file structure popup, bookmarks, breadcrumbs, and IntelliJ's native cut/copy/paste with undo integration. The JCEF diagram provides mouse-driven visual editing with spatial layout awareness. Users who prefer keyboard-centric workflows (common among IntelliJ power users) can add/remove/reorganize nodes from the Structure view without switching to the diagram. The two views complement each other; neither replaces the other.

Clipboard-based structural editing following the `BuilderClipboard` pattern from pages. In IntelliJ, the tree (native Structure view) handles cut/copy/paste via IntelliJ's native clipboard and undo system. The JCEF diagram handles its own structural operations (already implemented via `DiagramBaseMixin._applyGraphEdit()` in `casehub-diagram`, `swf-diagram`, and `blocks-org-diagram`) and syncs back via the bridge layer's diff-based delta protocol (§Sync Protocol).

**Clipboard bridge:** Users can copy a structural node (e.g., a capability) from the native Structure view and paste it into the JCEF diagram's drop target, or vice versa. The bridge uses a shared clipboard format — serialized YAML fragments with a format-type header — portable across both runtimes. Validation rules (fragment type → valid paste targets) are expressed as JSON config per format, consumed by both the Kotlin and TypeScript implementations. For example, a capability fragment copied from the tree can paste into any `spec.capabilities[]` slot in either the tree or the diagram.

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

The `intellij-plugin` job has no additional path filters of its own — it runs whenever the CI workflow triggers. The existing CI workflow already triggers on `packages/**`, `components/**`, and `examples/**`, which covers all sources that affect the plugin bundle (diagram components, graph-stencil packages, lsp-schemas). Adding `plugins/intellij-casehub/**` to the workflow-level path triggers ensures Kotlin-only changes also trigger CI. The `needs: build-and-test` dependency ensures the full TypeScript build passes before the Gradle build starts. Artifact upload is conditional on `github.ref == 'refs/heads/main'`; PR builds verify the Gradle build succeeds but do not upload artifacts.

## Delivery Phases

### Phase 1 — JCEF split editor + CI (blocks-ui only)

No pages dependency. Can proceed immediately.

1. `CaseHubFileEditorProvider` — `TextEditorWithPreview` with native YAML editor + JCEF panel, JCEF availability fallback, `Disposable` lifecycle
2. `diagram-panel.bundle.js` — esbuild bundle of diagram web components + HTML shell
3. `CaseHubDiagramPanel` — Kotlin JCEF wrapper with `CefMessageRouter` bridge, tab visibility handling, split mode disposal
4. Sync: `DocumentListener` → debounced YAML push to JCEF, echo suppression via origin flag
5. Theme sync: `LafManagerListener` → CSS custom property injection, initial theme in HTML shell
6. Format routing: file extension → diagram component (shared `DIAGRAM_TAGS` constant)
7. Gradle `copyDiagramBundle` task (mirrors existing `copyServerBundle`)
8. CI job: build plugin zip, upload as artifact on main push
9. Start with one format (`.case.yaml` → `casehub-diagram`) to prove the integration, then wire remaining formats

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
- `WorkbenchFormatRegistration` interface: `formatId` (must match a `FormatRegistration.formatId` in the schema registry), `visualElement` (custom element tag name). Schema and extensions are resolved from the `SchemaRegistry` — no duplication.
- Refactor `pages-builder-shell` to consume registrations instead of hardcoding `PageDocument`/`dashboardSchema`
- Page format becomes first consumer: `pageFormat` registration
- Tree derivation from Zod schema (array → collection, object → leaf)
- Property form derivation from Zod schema (string/enum/number/boolean/datetime → typed editors)
- Generalized `BuilderClipboard` fragment rules — array membership determines valid targets
- All existing page builder behavior must be preserved (regression test)

**Consumer:** blocks-ui domain formats (case/swf/htn/org) will register via this SPI

**Delivery:** This is substantial cross-repo work — refactoring `pages-builder-shell` from hardcoded page-specific logic to a generic format SPI while preserving all existing behavior. The pages issue should include its own design spec with adversarial review in the pages repo, not just an issue body. This spec defines the contract (`WorkbenchFormatRegistration` interface and Zod derivation strategy); the pages spec defines the implementation approach and migration plan.

**Not in scope:** IntelliJ integration (handled by blocks-ui), domain-specific visual renderers (already exist in blocks-ui)

## Files to Create/Modify

### Phase 1

| File | Change |
|------|--------|
| `plugins/intellij-casehub/src/main/kotlin/.../CaseHubFileEditorProvider.kt` | **New** — TextEditorWithPreview provider |
| `plugins/intellij-casehub/src/main/kotlin/.../CaseHubDiagramPanel.kt` | **New** — JCEF wrapper + CefMessageRouter bridge + Disposable lifecycle |
| `plugins/intellij-casehub/src/main/kotlin/.../DiagramSyncListener.kt` | **New** — DocumentListener with debounced YAML push + echo suppression |
| `plugins/intellij-casehub/src/main/kotlin/.../DiagramThemeSync.kt` | **New** — LafManagerListener + UIManager → CSS custom property mapping |
| `plugins/intellij-casehub/src/main/resources/diagram/diagram-shell.html` | **New** — HTML shell for JCEF (includes initial theme CSS) |
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
