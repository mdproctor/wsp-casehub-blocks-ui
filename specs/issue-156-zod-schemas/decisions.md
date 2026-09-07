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
