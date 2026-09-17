# Design Journal — issue-158-lsp-schema-refinements

## 2026-09-17 — Discriminator-aware schema generation

### What landed

Enhanced the ts-morph generator (`generate-domain-schemas.ts`) to produce
`z.union()` schemas at discriminated type points. CaseDefinition's `Binding`
type now generates a 3-branch union (capability / subCase / humanTask) and
`Trigger` generates a 4-branch union (contextChange / cloudEvent / schedule /
scopeActivated). Discriminator keys stay optional to avoid false diagnostics
during mid-edit states.

### Key design decisions

**D1 — No JSON manifests.** The discrimination structure lives in the Zod
schema itself (`z.union()`). No separate manifest to drift.

**D2 — ZodUnion sibling narrowing, not variantDispatchers.** The pages-lsp
completion engine already handles `ZodUnion` with sibling-based key matching.
The `variantDispatchers` field on `FormatRegistration` and `getVariantSchema()`
on `SchemaRegistry` are dead infrastructure — defined but never called from
the completion pipeline. No new dispatch maps needed.

**D3 — Generator produces unions, no hand-written schemas.** A small
TypeScript config file (~15 lines) declares which type names have key-presence
variants. The generator partitions properties automatically. Eliminates
hand-written schema maintenance entirely (except SWF, whose source types are
too flat for generation).

### Adversarial review findings

The review caught a latent narrowing bug in pages-lsp: `schemaToCompletions`
checks if ANY sibling key is in a branch's shape, but with `.extend()` common
keys appear in every branch — so narrowing never fires. The fix requires
discriminant-key detection (keys unique to exactly one branch). Tracked as a
separate pages-lsp issue. The union schemas and the narrowing fix are
independently deployable.

Also caught: `Binding` and `Trigger` are intersection types (from
`json-schema-to-typescript`), not plain objects. The discriminator check
needed to fire in both the intersection and object branches of `typeToZod()`.

Org's `RelationshipScope` was removed from scope — its three fields are NOT
mutually exclusive (a relationship can be scoped to both capability and domain).

### Side quests

- Filed casehubio/casehub-pages#451 for Zod v4 migration (pages landed it same day)
- Evaluated Zod v4 vs Valibot vs ArkType vs TypeBox — Zod remains the right choice
  for our LSP use case (ecosystem integration, schema introspection needs)

### What's next

- #161 — domain schema assembly for IntelliJ LSP plugin (active, unblocked,
  pages#423 and #424 both landed)
- Pages-lsp narrowing algorithm fix (discriminant-key detection) — separate issue
- Blocks-ui Zod v4 migration — follow-up after this branch closes
