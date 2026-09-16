## D1: JSON manifest role — declarative documentation, not runtime code

**Choice:** Create JSON manifest files as declarative documentation of the discrimination structure. Wire variantDispatchers from hand-written TypeScript schemas, not from manifest data. Defer generator integration to a future issue.
**Alternatives:**
- Generator-consumed manifests — ts-morph reads manifests to produce union-aware .generated.ts files. Higher value but larger scope (generator infrastructure changes).
- No manifests — define everything in TypeScript only. Simpler but loses the declarative documentation and cross-language potential.
**Rationale:** VariantDispatch.variants needs Map<string, z.ZodType> — Zod schemas are runtime TypeScript objects that can't be serialized to JSON. Any JSON manifest is necessarily incomplete as a runtime data source. The hand-written schemas in lsp-schemas/src/schemas/*.ts already contain the structural information needed for dispatch. JSON manifests document the discrimination structure for future generator work and cross-language consumption without blocking the immediate LSP narrowing.
**Trade-offs:** Two representations of the same knowledge (JSON declares structure, TypeScript implements it). Mitigated by a staleness test validating manifest entries against TypeScript dispatchers.
**Sources:** FormatRegistration interface in pages-lsp/dist/types.d.ts, lsp-schemas/src/schemas/case-definition.ts (hand-written), lsp-schemas/src/formats/case-definition.ts (format registration), LSP spec §Schema Registry §Domain Schema Generation Constraints
**Exploration:** deep-analysis
**Status:** captured
