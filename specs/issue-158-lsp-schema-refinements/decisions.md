## D1: No JSON manifests — discrimination lives in schema structure

**Choice:** No separate JSON manifest files. The discrimination structure is expressed directly in the Zod schemas via `z.union()`. The schema IS the manifest — no separate artifact to drift.
**Alternatives:**
- JSON manifests as declarative documentation — human-readable but a second representation that rots independently of the schemas
- JSON manifests consumed by generator — highest value but deferred scope (generator infrastructure changes)
**Rationale:** Hand-written artifacts rot. The Zod union structure is machine-readable, type-checked, and is the actual runtime data — no bridging layer needed. Future generator work can introspect the union schemas if it needs discrimination metadata.
**Trade-offs:** No cross-language manifest for Java engine consumption. Acceptable: the engine does its own validation via Java types today.
**Sources:** FormatRegistration interface in pages-lsp/dist/types.d.ts, schema-navigation.js ZodUnion handling
**Exploration:** deep-analysis
**Status:** captured

## D2: ZodUnion with sibling narrowing — no variantDispatchers needed

**Choice:** Restructure flat `.passthrough()` schemas into `z.union([variantA, variantB, ...])`. The existing `schemaToCompletions` in pages-lsp already narrows ZodUnion by matching sibling keys against each option's shape. No `variantDispatchers` wiring needed.
**Alternatives:**
- Wire variantDispatchers into FormatRegistration — adds dispatch infrastructure but `navigateSchema` doesn't actually consume it (getVariantSchema exists on the interface but isn't called from completion/navigation code)
- Modify navigateSchema to consult variantDispatchers — requires pages-lsp changes, more moving parts
**Rationale:** The completion engine already handles ZodUnion correctly: when a variant key is present as a sibling, it filters union options by shape match. When no variant is chosen, it merges all options (correct initial state). `.passthrough()` doesn't interfere because `getShape()` returns only explicitly declared keys. SWF already uses this pattern and it works.
**Trade-offs:** variantDispatchers infrastructure stays unused. The registry interface already defines it — but wiring data into it adds maintenance for no immediate consumer. Can be revisited if hover/diagnostics needs explicit dispatch.
**Depends on:** D1 (no manifests means no dispatch data source anyway)
**Sources:** schema-navigation.js:273-298 (ZodUnion sibling narrowing), schema-navigation.js:167-179 (ZodUnion navigation), lsp-schemas/src/schemas/swf.ts (existing union pattern)
**Exploration:** deep-analysis
**Status:** captured

## D3: Generator produces union-aware schemas — no hand-written schema files

**Choice:** Enhance the existing `generate-domain-schemas.ts` with a discriminator config per format. The generator reads the config, splits flat TypeScript types into `z.union()` schemas at discrimination points. Generated schemas replace hand-written schemas for case, org, and htn. SWF keeps its hand-written schema (source types are flat `Record<string, unknown>[]` — CNCF task structure isn't in the TS types).
**Alternatives:**
- Hand-written schemas with staleness tests — works but rots as user noted; maintenance burden for ~200 lines of schema code that duplicates generated output
- Programmatic schema transform (import generated flat schema, runtime-split into unions) — avoids hand-written schemas but introspects Zod `_def` internals, fragile across Zod versions
**Rationale:** The generator already has `discriminatorManifest?: string` on `FormatConfig` (line 15) — designed for this but never implemented. A small TypeScript config file (~20 lines per format) maps type names to variant keys. The generator separates properties into common vs variant groups and produces `z.union([common.extend({variant}), ...])`. Generated code uses only stable public API (`z.object()`, `.extend()`, `z.union()`). The config is validated at generation time — if a variant key doesn't exist in the TypeScript type, ts-morph throws.
**Trade-offs:** Generator becomes more complex (~50 lines of new logic). Acceptable: it's build-time code, not runtime, and eliminates hand-written schema maintenance entirely. SWF remains hand-written as the one exception.
**Depends on:** D2 (union-based narrowing is what makes generated unions work for completion)
**Sources:** generate-domain-schemas.ts (existing generator with discriminatorManifest hook), ts-to-zod @discriminator JSDoc pattern (prior art — we use config file instead since source types are auto-generated from engine), Zod v4 deprecation of z.discriminatedUnion (z.union() is the stable path)
**Exploration:** deep-analysis
**Status:** captured
