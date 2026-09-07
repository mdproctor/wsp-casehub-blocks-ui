## D1: Component discovery mechanism

**Choice:** Create a `BlocksComponentRegistry` interface in blocks-ui-core that maps element names to prop interfaces, mirroring the pages `ComponentTypeRegistry` pattern
**Alternatives:**
- Scan `@customElement` decorators and extract `@property()` declarations directly — avoids a separate registry but couples the generator to Lit decorator internals and is more brittle
**Rationale:** Single source of truth for the component surface area. Matches the proven pages pattern exactly, making the generator simpler and the two systems consistent. Forces each component to declare a clean Props interface.
**Trade-offs:** Every new component must add an entry to the registry manually — a small maintenance cost
**Sources:** pages `ComponentTypeRegistry` at `packages/pages-component/src/model/type-guards.ts`, pages generator at `packages/pages-schema/scripts/generate-schemas.ts`
**Exploration:** quick
**Status:** captured

## D2: Schema package location

**Choice:** New `packages/blocks-ui-schema` package, mirroring pages-schema structure
**Alternatives:**
- Inside `packages/blocks-ui-core` — fewer packages but mixes domain types with generated schemas and adds ts-morph dev dependency to core
**Rationale:** Clean separation of concerns. Generator machinery (ts-morph, code generation) stays out of the core package. Mirrors pages-schema for cross-repo consistency.
**Trade-offs:** One more package in the workspace to maintain
**Sources:** pages `packages/pages-schema/package.json`
**Exploration:** quick
**Depends on:** D1 (registry interface lives in blocks-ui-core, schemas package reads from it)
**Status:** captured

## D3: Fix untyped configure() methods

**Choice:** Fix the ~16 untyped `configure(props: Record<string, unknown>)` methods as part of this issue — replace with typed Props interface from the registry
**Alternatives:**
- Defer to a separate issue — schemas would still be correct (generated from registry, not configure), but runtime enforcement gap remains
**Rationale:** Mechanical fix that completes the audit task from the issue. Ensures runtime behaviour matches what schemas declare. Leaving untyped configure() creates a gap where undeclared properties still pass through at runtime.
**Trade-offs:** Increases scope slightly (~16 files to touch), but each change is a simple type annotation + cast removal
**Sources:** execution-monitor.ts:186 `configure(props: Record<string, unknown>)`, audit finding of ~16 components with this pattern
**Exploration:** quick
**Depends on:** D1 (Props interfaces must exist before configure() can reference them)
**Status:** captured
