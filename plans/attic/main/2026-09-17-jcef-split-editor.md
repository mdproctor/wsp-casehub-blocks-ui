# JCEF Split Editor + CI Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #161 — domain schema assembly for IntelliJ LSP plugin
**Issue group:** #158, #159, #160, #161

**Goal:** Add a JCEF-based visual diagram panel alongside the native YAML editor in the IntelliJ plugin, with one-directional sync (editor → diagram), theme sync, and CI distribution.

**Architecture:** Phase 1 of the native-first hybrid architecture (D4). The existing CaseHubYAML language + LSP text intelligence stays untouched. A `TextEditorWithPreview` split editor wraps the native YAML editor alongside a JCEF panel that renders diagram web components. Editor edits push full YAML to the diagram (debounced). Diagram → editor sync is deferred until a pages PR lands the `yaml-changed` event on `DiagramBaseMixin`. CI builds the plugin zip on main pushes.

**Tech Stack:** Kotlin (IntelliJ Platform SDK, JCEF, LSP4IJ), TypeScript/esbuild (diagram bundle), GitHub Actions (CI)

## Global Constraints

- IntelliJ Platform 2024.2+ (`sinceBuild = "242"`)
- LSP4IJ 0.20.1
- Java 21 (Kotlin JVM toolchain)
- Node.js 18+ (esbuild, diagram bundle runtime)
- esbuild 0.28+ (bundler)
- JCEF availability: graceful fallback when `JBCefApp.isSupported()` returns false
- All Kotlin → JS string transfers via `executeJavaScript()` must use `Json.encodeToString()` — no raw string interpolation
- Diagram → editor sync (bidirectional) is OUT OF SCOPE — requires pages PR for `DiagramBaseMixin.yaml-changed` event

---

## Batch 1: Diagram esbuild bundle + format tag constant

After this batch: a standalone `diagram-panel.bundle.js` builds from the diagram web components, and a shared `DIAGRAM_TAGS` constant is available in `blocks-ui-core`. The IntelliJ plugin doesn't use them yet.

### Task 1: Shared DIAGRAM_TAGS constant

**Files:**
- Modify: `packages/blocks-ui-core/src/index.ts`
- Modify: `components/diagram-workbench/src/diagram-workbench.ts`
- Test: `packages/blocks-ui-core/src/diagram-tags.test.ts` (new)

**Interfaces:**
- Produces: `DIAGRAM_TAGS: Record<string, string>` exported from `@casehubio/blocks-ui-core` — maps format key to web component tag name: `{ case: 'casehub-diagram', swf: 'swf-diagram', htn: 'htn-diagram', org: 'blocks-org-diagram' }`

- [ ] **Step 1: Write the failing test**

Create `packages/blocks-ui-core/src/diagram-tags.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { DIAGRAM_TAGS } from './diagram-tags.js';

describe('DIAGRAM_TAGS', () => {
  it('maps all four domain formats', () => {
    expect(DIAGRAM_TAGS).toEqual({
      case: 'casehub-diagram',
      swf: 'swf-diagram',
      htn: 'htn-diagram',
      org: 'blocks-org-diagram',
    });
  });

  it('does not include page format', () => {
    expect(DIAGRAM_TAGS).not.toHaveProperty('page');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd packages/blocks-ui-core vitest run src/diagram-tags.test.ts`
Expected: FAIL — module `./diagram-tags.js` not found

- [ ] **Step 3: Create the constant**

Create `packages/blocks-ui-core/src/diagram-tags.ts`:

```typescript
export const DIAGRAM_TAGS: Record<string, string> = {
  case: 'casehub-diagram',
  swf: 'swf-diagram',
  htn: 'htn-diagram',
  org: 'blocks-org-diagram',
};
```

Add the export to `packages/blocks-ui-core/src/index.ts`:

```typescript
export { DIAGRAM_TAGS } from './diagram-tags.js';
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn --cwd packages/blocks-ui-core vitest run src/diagram-tags.test.ts`
Expected: PASS

- [ ] **Step 5: Update diagram-workbench to consume the shared constant**

In `components/diagram-workbench/src/diagram-workbench.ts`, replace the local `DIAGRAM_TAGS` constant (lines 11-15):

Old:
```typescript
const DIAGRAM_TAGS: Record<string, string> = {
  swf: 'swf-diagram',
  case: 'casehub-diagram',
  htn: 'htn-diagram',
};
```

New:
```typescript
import { DIAGRAM_TAGS } from '@casehubio/blocks-ui-core';
```

- [ ] **Step 6: Run diagram-workbench tests**

Run: `yarn --cwd components/diagram-workbench vitest run`
Expected: PASS — existing tests work with the imported constant

- [ ] **Step 7: Commit**

```bash
git add packages/blocks-ui-core/src/diagram-tags.ts packages/blocks-ui-core/src/diagram-tags.test.ts packages/blocks-ui-core/src/index.ts components/diagram-workbench/src/diagram-workbench.ts
git commit -m "refactor: extract DIAGRAM_TAGS to blocks-ui-core for sharing

Adds 'org: blocks-org-diagram' entry not present in the diagram-workbench
original. diagram-workbench now imports from blocks-ui-core.

Refs #161"
```

### Task 2: Esbuild diagram bundle + HTML shell

**Files:**
- Create: `packages/lsp-schemas/build-diagram-bundle.js`
- Create: `packages/lsp-schemas/src/diagram-shell.html`
- Modify: `packages/lsp-schemas/package.json` (add `build:diagram` script)
- Test: build output verification (inline)

**Interfaces:**
- Consumes: `@casehubio/blocks-ui-casehub-diagram` (web component auto-registers on import)
- Produces: `packages/lsp-schemas/dist/diagram-panel.bundle.js` — self-contained browser bundle; `packages/lsp-schemas/src/diagram-shell.html` — HTML shell that loads the bundle

- [ ] **Step 1: Create the esbuild script**

Create `packages/lsp-schemas/build-diagram-bundle.js`:

```javascript
import { build } from 'esbuild';

await build({
  entryPoints: ['src/diagram-entry.ts'],
  bundle: true,
  format: 'esm',
  platform: 'browser',
  target: 'es2022',
  outfile: 'dist/diagram-panel.bundle.js',
  sourcemap: 'linked',
  define: {
    'process.env.NODE_ENV': '"production"',
  },
});
```

- [ ] **Step 2: Create the diagram entry point**

Create `packages/lsp-schemas/src/diagram-entry.ts`:

```typescript
import '@casehubio/blocks-ui-casehub-diagram';
import { DIAGRAM_TAGS } from '@casehubio/blocks-ui-core';

interface YamlPushMessage {
  type: 'yaml-push';
  yaml: string;
  format: string;
}

interface ThemeMessage {
  type: 'theme';
  css: string;
}

type BridgeMessage = YamlPushMessage | ThemeMessage;

let activeElement: HTMLElement | null = null;
let pushing = false;

function createDiagramElement(tag: string): HTMLElement {
  const el = document.createElement(tag);
  el.setAttribute('mode', 'readonly');
  document.getElementById('diagram-root')!.innerHTML = '';
  document.getElementById('diagram-root')!.appendChild(el);
  return el;
}

(window as any).updateYaml = (yaml: string, format: string) => {
  const tag = DIAGRAM_TAGS[format];
  if (!tag) return;

  if (!activeElement || activeElement.tagName.toLowerCase() !== tag) {
    activeElement = createDiagramElement(tag);
  }

  pushing = true;
  (activeElement as any).yaml = yaml;
  pushing = false;
};

(window as any).updateTheme = (css: string) => {
  let style = document.getElementById('theme-vars');
  if (!style) {
    style = document.createElement('style');
    style.id = 'theme-vars';
    document.head.appendChild(style);
  }
  style.textContent = css;
};

(window as any).getDiagramTags = () => DIAGRAM_TAGS;
```

- [ ] **Step 3: Create the HTML shell**

Create `packages/lsp-schemas/src/diagram-shell.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style id="theme-vars">
    :root {
      --pages-surface-color: #ffffff;
      --pages-border-color: #d1d5db;
      --pages-text-color: #1f2937;
      --pages-accent-color: #3b82f6;
      --pages-muted-color: #6b7280;
    }
  </style>
  <style>
    body { margin: 0; padding: 0; overflow: hidden; background: var(--pages-surface-color); }
    #diagram-root { width: 100vw; height: 100vh; }
  </style>
</head>
<body>
  <div id="diagram-root"></div>
  <script type="module" src="diagram-panel.bundle.js"></script>
</body>
</html>
```

- [ ] **Step 4: Add build script to package.json**

In `packages/lsp-schemas/package.json`, add to `scripts`:

```json
"build:diagram": "node build-diagram-bundle.js"
```

Add `@casehubio/blocks-ui-casehub-diagram` and `@casehubio/blocks-ui-core` to `dependencies`:

```json
"@casehubio/blocks-ui-casehub-diagram": "workspace:*",
"@casehubio/blocks-ui-core": "workspace:*"
```

- [ ] **Step 5: Build the bundle and verify output**

Run: `yarn --cwd packages/lsp-schemas build:diagram`
Expected: `dist/diagram-panel.bundle.js` and `dist/diagram-panel.bundle.js.map` are created

Run: `ls -lh packages/lsp-schemas/dist/diagram-panel.bundle.js`
Expected: file exists (size will vary, target ≤3MB)

- [ ] **Step 6: Commit**

```bash
git add packages/lsp-schemas/build-diagram-bundle.js packages/lsp-schemas/src/diagram-entry.ts packages/lsp-schemas/src/diagram-shell.html packages/lsp-schemas/package.json
git commit -m "feat(lsp-schemas): esbuild diagram bundle + HTML shell for JCEF

Bundles casehub-diagram web component as a browser ESM bundle.
HTML shell provides the JCEF loading surface with default light theme
CSS custom properties.

Refs #161"
```

## Batch 2: IntelliJ split editor

After this batch: opening a `.case.yaml` file in IntelliJ shows a split editor with the native YAML editor on the left and the casehub-diagram rendering on the right. Theme syncs with IntelliJ's look-and-feel. Editor edits update the diagram in real time.

### Task 3: JCEF diagram panel + theme sync + sync listener

**Files:**
- Create: `plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/CaseHubDiagramPanel.kt`
- Create: `plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/DiagramSyncListener.kt`
- Create: `plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/DiagramThemeSync.kt`
- Test: manual verification (JCEF requires a running IDE)

**Interfaces:**
- Consumes: `diagram-shell.html` + `diagram-panel.bundle.js` extracted from plugin resources
- Produces: `CaseHubDiagramPanel` — a `Disposable` JPanel wrapping JBCefBrowser; `DiagramSyncListener` — DocumentListener that pushes YAML to the panel; `DiagramThemeSync` — maps IntelliJ theme colors to CSS custom properties

- [ ] **Step 1: Create DiagramThemeSync**

Create `plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/DiagramThemeSync.kt`:

```kotlin
package io.casehub.intellij

import com.intellij.openapi.editor.colors.EditorColorsManager
import com.intellij.ui.JBColor
import kotlinx.serialization.json.Json
import javax.swing.UIManager
import java.awt.Color

object DiagramThemeSync {

    private val CSS_MAPPINGS = mapOf(
        "Panel.background" to "--pages-surface-color",
        "Label.foreground" to "--pages-text-color",
        "Component.borderColor" to "--pages-border-color",
        "Component.focusColor" to "--pages-accent-color",
        "Label.disabledForeground" to "--pages-muted-color",
        "Tree.background" to "--pages-panel-bg",
        "Table.stripeColor" to "--pages-stripe-color",
        "Actions.Red" to "--pages-error-color",
        "Actions.Yellow" to "--pages-warning-color",
        "Actions.Green" to "--pages-success-color",
    )

    fun buildThemeCss(): String {
        val lines = CSS_MAPPINGS.mapNotNull { (uiKey, cssVar) ->
            val color = UIManager.getColor(uiKey) ?: return@mapNotNull null
            "$cssVar: ${toHex(color)};"
        }
        return ":root { ${lines.joinToString(" ")} }"
    }

    fun buildThemeInjectionJs(): String {
        val css = buildThemeCss()
        val encoded = Json.encodeToString(css)
        return "window.updateTheme($encoded)"
    }

    fun installListener(panel: CaseHubDiagramPanel): com.intellij.ide.ui.LafManagerListener {
        val listener = com.intellij.ide.ui.LafManagerListener { panel.pushTheme() }
        com.intellij.openapi.application.ApplicationManager.getApplication()
            .messageBus.connect(panel)
            .subscribe(com.intellij.ide.ui.LafManagerListener.TOPIC, listener)
        return listener
    }

    private fun toHex(c: Color): String =
        "#%02x%02x%02x".format(c.red, c.green, c.blue)
}
```

- [ ] **Step 2: Create DiagramSyncListener**

Create `plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/DiagramSyncListener.kt`:

```kotlin
package io.casehub.intellij

import com.intellij.openapi.editor.Document
import com.intellij.openapi.editor.event.DocumentEvent
import com.intellij.openapi.editor.event.DocumentListener
import kotlinx.serialization.json.Json
import java.util.Timer
import java.util.TimerTask

class DiagramSyncListener(
    private val panel: CaseHubDiagramPanel,
    private val format: String,
) : DocumentListener {

    @Volatile
    var suppressEcho = false

    private var debounceTimer: Timer? = null
    private val debounceMs = 150L
    var paused = false

    override fun documentChanged(event: DocumentEvent) {
        if (suppressEcho || paused) return
        scheduleYamlPush(event.document)
    }

    private fun scheduleYamlPush(document: Document) {
        debounceTimer?.cancel()
        debounceTimer = Timer("diagram-sync", true).also { timer ->
            timer.schedule(object : TimerTask() {
                override fun run() {
                    val yaml = document.text
                    panel.pushYaml(yaml, format)
                }
            }, debounceMs)
        }
    }

    fun dispose() {
        debounceTimer?.cancel()
        debounceTimer = null
    }
}
```

- [ ] **Step 3: Create CaseHubDiagramPanel**

Create `plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/CaseHubDiagramPanel.kt`:

```kotlin
package io.casehub.intellij

import com.intellij.openapi.Disposable
import com.intellij.openapi.util.Disposer
import com.intellij.ui.jcef.JBCefBrowser
import com.intellij.ui.jcef.JBCefApp
import kotlinx.serialization.json.Json
import java.awt.BorderLayout
import java.io.File
import java.nio.file.Files
import java.nio.file.Path
import java.nio.file.StandardCopyOption
import javax.swing.JLabel
import javax.swing.JPanel
import javax.swing.SwingConstants

class CaseHubDiagramPanel(parent: Disposable) : JPanel(BorderLayout()), Disposable {

    private var browser: JBCefBrowser? = null

    init {
        Disposer.register(parent, this)

        if (!JBCefApp.isSupported()) {
            add(JLabel(
                "Visual diagram requires a local IDE (not available in Remote Development).",
                SwingConstants.CENTER
            ), BorderLayout.CENTER)
        } else {
            val targetDir = extractDiagramResources()
            val htmlFile = targetDir.resolve("diagram-shell.html")
            val cefBrowser = JBCefBrowser(htmlFile.toUri().toString())
            browser = cefBrowser
            add(cefBrowser.component, BorderLayout.CENTER)
            Disposer.register(this, cefBrowser)
        }
    }

    fun pushYaml(yaml: String, format: String) {
        val b = browser ?: return
        val encodedYaml = Json.encodeToString(yaml)
        val encodedFormat = Json.encodeToString(format)
        b.cefBrowser.executeJavaScript(
            "window.updateYaml($encodedYaml, $encodedFormat)",
            "", 0
        )
    }

    fun pushTheme() {
        val b = browser ?: return
        b.cefBrowser.executeJavaScript(DiagramThemeSync.buildThemeInjectionJs(), "", 0)
    }

    private fun extractDiagramResources(): Path {
        val targetDir = Path.of(System.getProperty("java.io.tmpdir"), "casehub-diagram")
        Files.createDirectories(targetDir)

        for (name in listOf("diagram-shell.html", "diagram-panel.bundle.js", "diagram-panel.bundle.js.map")) {
            val resource = javaClass.getResourceAsStream("/diagram/$name") ?: continue
            resource.use { input ->
                Files.copy(input, targetDir.resolve(name), StandardCopyOption.REPLACE_EXISTING)
            }
        }
        return targetDir
    }

    override fun dispose() {
        browser = null
    }
}
```

- [ ] **Step 4: Commit**

```bash
git add plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/CaseHubDiagramPanel.kt plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/DiagramSyncListener.kt plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/DiagramThemeSync.kt
git commit -m "feat(intellij): JCEF diagram panel + sync listener + theme sync

CaseHubDiagramPanel wraps JBCefBrowser with Disposable lifecycle.
DiagramSyncListener debounces YAML pushes with echo suppression.
DiagramThemeSync maps IntelliJ UIManager colors to --pages-* CSS vars.
All Kotlin→JS transfers use Json.encodeToString for sanitization.

Refs #161"
```

### Task 4: Split editor provider + Gradle wiring

**Files:**
- Create: `plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/CaseHubFileEditorProvider.kt`
- Modify: `plugins/intellij-casehub/src/main/resources/META-INF/plugin.xml`
- Modify: `plugins/intellij-casehub/build.gradle.kts`
- Test: Gradle build verification

**Interfaces:**
- Consumes: `CaseHubDiagramPanel`, `DiagramSyncListener`, `DiagramThemeSync`, `CaseHubYamlFileType`
- Produces: Split editor in IntelliJ when opening CaseHubYAML files

- [ ] **Step 1: Create CaseHubFileEditorProvider**

Create `plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/CaseHubFileEditorProvider.kt`:

```kotlin
package io.casehub.intellij

import com.intellij.openapi.fileEditor.*
import com.intellij.openapi.fileEditor.impl.text.TextEditorProvider
import com.intellij.openapi.project.DumbAware
import com.intellij.openapi.project.Project
import com.intellij.openapi.vfs.VirtualFile
import com.intellij.ui.jcef.JBCefApp

class CaseHubFileEditorProvider : FileEditorProvider, DumbAware {

    override fun getEditorTypeId(): String = "casehub-yaml-diagram"

    override fun getPolicy(): FileEditorPolicy = FileEditorPolicy.HIDE_DEFAULT_EDITOR

    override fun accept(project: Project, file: VirtualFile): Boolean {
        val name = file.name
        return name.endsWith(".case.yaml") ||
               name.endsWith(".swf.yaml") ||
               name.endsWith(".htn.yaml") ||
               name.endsWith(".org.yaml")
    }

    override fun createEditor(project: Project, file: VirtualFile): FileEditor {
        val textEditor = TextEditorProvider.getInstance().createEditor(project, file) as TextEditor

        if (!JBCefApp.isSupported()) {
            return textEditor
        }

        val format = when {
            file.name.endsWith(".case.yaml") -> "case"
            file.name.endsWith(".swf.yaml") -> "swf"
            file.name.endsWith(".htn.yaml") -> "htn"
            file.name.endsWith(".org.yaml") -> "org"
            else -> return textEditor
        }

        val diagramPanel = CaseHubDiagramPanel(textEditor)
        val syncListener = DiagramSyncListener(diagramPanel, format)

        val editor = textEditor.editor
        editor.document.addDocumentListener(syncListener)

        val splitEditor = TextEditorWithPreview(
            textEditor,
            CaseHubDiagramEditor(diagramPanel, textEditor),
            "CaseHub YAML",
            TextEditorWithPreview.Layout.SHOW_EDITOR_AND_PREVIEW,
        )

        diagramPanel.pushYaml(editor.document.text, format)
        diagramPanel.pushTheme()
        DiagramThemeSync.installListener(diagramPanel)

        return splitEditor
    }
}

class CaseHubDiagramEditor(
    private val panel: CaseHubDiagramPanel,
    parent: TextEditor,
) : FileEditor by parent {

    override fun getComponent() = panel

    override fun getPreferredFocusedComponent() = panel

    override fun getName() = "Diagram"
}
```

- [ ] **Step 2: Register the provider in plugin.xml**

Add to `plugins/intellij-casehub/src/main/resources/META-INF/plugin.xml`, inside the `<extensions defaultExtensionNs="com.intellij">` block:

```xml
<fileEditorProvider
        implementation="io.casehub.intellij.CaseHubFileEditorProvider"/>
```

- [ ] **Step 3: Add Gradle dependencies and copy tasks**

In `plugins/intellij-casehub/build.gradle.kts`, add the `kotlinx-serialization` dependency for `Json.encodeToString`:

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
    // ... existing dependencies
}
```

Add the serialization plugin:

```kotlin
plugins {
    id("org.jetbrains.kotlin.jvm") version "2.1.21"
    id("org.jetbrains.kotlin.plugin.serialization") version "2.1.21"
    id("org.jetbrains.intellij.platform")
}
```

Add the diagram bundle copy task alongside the existing `copyServerBundle`:

```kotlin
val copyDiagramBundle = tasks.register<Copy>("copyDiagramBundle") {
    from("../../packages/lsp-schemas/dist/diagram-panel.bundle.js")
    from("../../packages/lsp-schemas/dist/diagram-panel.bundle.js.map")
    from("../../packages/lsp-schemas/src/diagram-shell.html")
    into(layout.buildDirectory.dir("resources/main/diagram"))
}

tasks.named("processResources") {
    dependsOn(copyServerBundle)
    dependsOn(copyDiagramBundle)
}
```

- [ ] **Step 4: Build the Gradle project to verify compilation**

Run: `yarn --cwd packages/lsp-schemas build:diagram` (ensure bundle exists)

Then build the Gradle project:
```bash
/Users/mdproctor/claude/casehub/blocks-ui/plugins/intellij-casehub/gradlew -p /Users/mdproctor/claude/casehub/blocks-ui/plugins/intellij-casehub build
```
Expected: BUILD SUCCESSFUL — plugin zip created at `build/distributions/intellij-casehub-0.1.0.zip`

- [ ] **Step 5: Commit**

```bash
git add plugins/intellij-casehub/src/main/kotlin/io/casehub/intellij/CaseHubFileEditorProvider.kt plugins/intellij-casehub/src/main/resources/META-INF/plugin.xml plugins/intellij-casehub/build.gradle.kts
git commit -m "feat(intellij): split editor — YAML + JCEF diagram panel

TextEditorWithPreview shows native YAML editor alongside JCEF diagram.
Format routing: .case.yaml → casehub-diagram, .swf/.htn/.org planned.
JCEF fallback returns native editor when JBCefApp.isSupported() is false.
Theme sync injects --pages-* CSS vars from IntelliJ's LookAndFeel.

Refs #161"
```

## Batch 3: CI pipeline

After this batch: CI builds the IntelliJ plugin zip and uploads it as a GitHub Actions artifact on main pushes. PRs verify the build succeeds.

### Task 5: GitHub Actions workflow for plugin build

**Files:**
- Modify: `.github/workflows/ci.yml`
- Test: verify workflow syntax

**Interfaces:**
- Consumes: `packages/lsp-schemas/build:diagram` (diagram bundle), Gradle build (plugin zip)
- Produces: `intellij-casehub-*.zip` artifact on main push

- [ ] **Step 1: Add plugin paths to CI triggers**

In `.github/workflows/ci.yml`, add `plugins/intellij-casehub/**` to both `push.paths` and `pull_request.paths`:

```yaml
paths:
  - 'packages/**'
  - 'components/**'
  - 'plugins/intellij-casehub/**'
  - 'examples/**'
  - 'package.json'
  - 'yarn.lock'
  - '.yarnrc.yml'
  - 'tsconfig*.json'
  - '.github/workflows/ci.yml'
```

- [ ] **Step 2: Add the intellij-plugin job**

Add after the `build-and-test` job:

```yaml
  intellij-plugin:
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          server-id: github
          server-username: GITHUB_ACTOR
          server-password: GITHUB_TOKEN

      - name: Resolve pages packages from Maven
        run: mvn -f npm-packages/pom.xml initialize
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'yarn'

      - name: Enable Corepack
        run: corepack enable

      - name: Install dependencies
        run: yarn install

      - name: Build packages
        run: yarn build

      - name: Build diagram bundle
        run: yarn --cwd packages/lsp-schemas build:diagram

      - name: Build LSP server bundle
        run: yarn --cwd packages/lsp-schemas build:bundle

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4

      - name: Build IntelliJ plugin
        run: ./gradlew build
        working-directory: plugins/intellij-casehub

      - name: Upload plugin artifact
        if: github.ref == 'refs/heads/main'
        uses: actions/upload-artifact@v4
        with:
          name: intellij-casehub-plugin
          path: plugins/intellij-casehub/build/distributions/*.zip
          retention-days: 30
```

- [ ] **Step 3: Validate workflow syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))" && echo "VALID"`
Expected: VALID

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: build IntelliJ plugin + upload artifact on main push

New intellij-plugin job depends on build-and-test. Builds diagram bundle,
LSP server bundle, and Gradle plugin. Uploads zip artifact on main push
only (PRs verify build but don't upload).

Refs #161"
```

## References

- `specs/issue-158-lsp-schema-refinements/2026-09-17-domain-schema-assembly-design.md` — design spec
- `plugins/intellij-casehub/` — existing IntelliJ plugin
- `packages/lsp-schemas/build-bundle.js` — existing esbuild pattern
- `components/diagram-workbench/src/diagram-workbench.ts:11-15` — existing DIAGRAM_TAGS
- `components/casehub-diagram/` — diagram web component
- `.github/workflows/ci.yml` — existing CI workflow
- D4-D7 — architecture decisions
- GitHub #161 — focal issue
