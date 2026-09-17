# Design: Discriminator-Aware Schema Generation for Narrowed LSP Completion

**Issue:** casehubio/blocks-ui#158
**Date:** 2026-09-17
**Branch:** issue-158-lsp-schema-refinements

## Overview

Enhance the `generate-domain-schemas.ts` generator to produce `z.union()` schemas at discriminated type points, and fix the `schemaToCompletions` narrowing algorithm in pages-lsp to use discriminant-key detection, enabling the completion engine to narrow suggestions based on which variant key is present in the YAML.

## Problem

The generated Zod schemas for CaseDefinition, Org, and HTN produce flat `z.object()` types where mutually exclusive optional keys all appear on the same object. The completion engine shows all properties simultaneously — typing `capability:` inside a binding still shows `subCase` and `humanTask` as suggestions.

## Solution

Two coordinated changes:

1. **Generator enhancement (lsp-schemas):** The generator reads a per-format discriminator config that declares which TypeScript type names have key-presence variants. At those types, it splits properties into common and variant groups, producing `z.union([common.extend({variant1}), common.extend({variant2}), ...])` instead of a flat object.

2. **Narrowing algorithm fix (pages-lsp):** The existing `schemaToCompletions` ZodUnion narrowing checks if ANY sibling key is in a branch's shape. With `.extend()`, common keys appear in every branch, so all branches always match and narrowing never fires. The algorithm must be enhanced to detect discriminant keys — sibling keys that are unique to exactly one branch.

### Why not variantDispatchers?

Issue #158 proposes wiring `FormatRegistration.variantDispatchers`. However, the issue's claim that "the `navigateSchema()` walker in pages-lsp already consults `variantDispatchers`" is factually incorrect — `navigateSchema` handles ZodUnion by trying each option sequentially (`schema-navigation.ts:185-192`), and `handleCompletion` never calls `getVariantSchema`. The `variantDispatchers` field and `SchemaRegistry.getVariantSchema()` are dead infrastructure — defined but never invoked from the completion pipeline.

Fixing the narrowing algorithm is architecturally cleaner:
- The schema structure IS the discrimination data — no separate map needed
- The fix works generically for any ZodUnion with discriminant keys, including the hand-written SWF schema (which has the same latent narrowing bug with shared `then`/`if` keys)
- No additional FormatRegistration fields, registry changes, or per-format wiring needed

## Discriminator Config

A TypeScript config file per format, referenced by `FormatConfig.discriminatorManifest` (field already exists on line 15 of the generator):

```typescript
// packages/lsp-schemas/scripts/discriminators/case-definition.ts

export interface DiscriminatorRule {
  strategy: 'key-presence';
  discriminatorKeys: string[];
}

export const discriminators: Record<string, DiscriminatorRule> = {
  Binding: {
    strategy: 'key-presence',
    discriminatorKeys: ['capability', 'subCase', 'humanTask'],
  },
  Trigger: {
    strategy: 'key-presence',
    discriminatorKeys: ['contextChange', 'cloudEvent', 'schedule', 'scopeActivated'],
  },
};
```

Each entry maps a **TypeScript type name** (as seen by ts-morph) to its discriminator keys. The remaining properties on that type become common fields shared across all variants. Shared keys are derived automatically — no explicit `sharedKeys` field needed.

### Relationship to existing infrastructure

The generator already declares (lines 18-24):
```typescript
interface DiscriminatorRule {
  strategy: 'key-presence';
  discriminatorKeys: string[];
  sharedKeys: string[];
}
type DiscriminatorManifest = Record<string, DiscriminatorRule>;
```

And `discriminators/case-definition.json` exists with path-based keys (`spec.bindings[]`). This JSON file and the existing interface are dead code — the `discriminatorManifest` field on `FormatConfig` is declared but no FORMATS entry sets it.

This design replaces the dead code:
- **TypeScript over JSON** — type-safe, catches config errors at compile time
- **Type-name keys over path-based keys** — the generator walks types by name, not by YAML path
- **Derived shared keys** — all non-discriminator-key properties are automatically common; no manual `sharedKeys` maintenance
- **Field alignment:** `discriminatorKeys` matches the existing interface name (renamed from the original draft's `variantKeys`)
- **`sharedKeys` dropped** — redundant with automatic derivation
- The existing `DiscriminatorRule` and `DiscriminatorManifest` interfaces at lines 18-24 of `generate-domain-schemas.ts` are **replaced** by the new `DiscriminatorRule` from the TypeScript config (exported from the config file, imported by the generator)
- The existing `case-definition.json` (which also has an incorrect key: `trigger` instead of `capability`) is **deleted**

### CaseDefinition Discriminators

| TypeScript Type | Discriminator Keys | Strategy |
|----------------|-------------------|----------|
| `Binding` | `capability`, `subCase`, `humanTask` | key-presence |
| `Trigger` | `contextChange`, `cloudEvent`, `schedule`, `scopeActivated` | key-presence |

The issue body lists 5 discriminated unions (adding WorkerFunction, McpTransport, ModelProvider), but these types aren't part of the CaseDefinition YAML schema type tree. They exist in `graph-stencil-case/src/worker-function/types.ts`, which is a separate type hierarchy not reachable from the `CaseHub` root type that the generator walks. Tracked in casehubio/blocks-ui#TBD-worker-function-discriminators.

### Org Discriminators

None. `RelationshipScope` has three optional fields (`capabilityName`, `domain`, `custom`) but they are NOT mutually exclusive — a relationship can be scoped to both a capability and a domain simultaneously. The TypeScript interface (`graph-stencil-org/src/types.ts:28-32`) has all three fields independently optional with no exclusivity constraint. Converting to a union would prevent valid multi-scope relationships.

### HTN Discriminators

None identified in the current type tree. HTN types (`HtnTask`, `HtnMethod`) don't have key-presence unions.

### SWF

No generator change. SWF continues using its hand-written schema (`lsp-schemas/src/schemas/swf.ts`) which already has a proper `z.union([callTaskSchema, setTaskSchema, switchTaskSchema, ...])` with discriminant keys (`call`, `set`, `switch`, `raise`, `emit`, `wait`, `listen`). The enhanced narrowing algorithm benefits SWF too — the hand-written schema has shared `then`/`if` keys that cause the same latent narrowing bug that this design fixes.

## Generator Enhancement

### Current Flow

```
TypeScript interface → ts-morph → typeToZod() → flat z.object({...})
```

### Enhanced Flow

```
TypeScript interface + discriminator config → ts-morph → typeToZod() →
  if type name in config: z.union([common.extend({v1}), common.extend({v2}), ...])
  else: z.object({...}) (unchanged)
```

### Implementation in `typeToZod()`

**Intersection type consideration:** The CaseDefinition types (`Binding`, `Trigger`) are intersection types, not plain object types — `json-schema-to-typescript` emits `export type Binding = {...} & Binding1` where `Binding1 = { [k: string]: unknown | undefined }` is a passthrough index-signature type. In `typeToZod()`, intersection types enter the `type.isIntersection()` branch (line 109) before the `type.isObject()` branch (line 131). Both branches produce the same `z.object({...})` output (the intersection handler merges properties and filters index signatures), but the discriminator check must fire in both.

The discriminator check is extracted into a shared helper called from both the intersection handler and the object handler. When either handler resolves the type's properties, the helper:

1. **Extracts the symbol name** via `type.getSymbol() || type.getAliasSymbol()` (for intersection type aliases like `Binding`, `getAliasSymbol()` returns the alias name).
2. **Checks the discriminator config** — if the symbol name exists as a key, the helper takes over; otherwise falls through to flat `z.object({...})`.
3. **Partitions properties** into common (not in `discriminatorKeys`) and discriminator (in `discriminatorKeys`).
4. **Generates a common base schema** from the common properties: `z.object({ name: z.string(), on: ..., when: ... })`.
5. **Generates per-variant schemas** by extending the common base with each discriminator key and its type: `commonSchema.extend({ capability: z.string().optional() })`.
6. **Emits `z.union([...])`** wrapping all variant schemas.

Discriminator keys stay **optional** on their variant schema — matching the source TypeScript type where they are optional. The other discriminator keys are absent (not optional — absent). Keeping the active discriminator key optional is critical because `computeDiagnostics` in pages-lsp calls `format.documentSchema.safeParse()` on every document change (`diagnostics.ts:40`). Making the key required would cause all union branches to fail for bindings mid-edit (no variant key chosen yet), producing false diagnostic errors.

Optional keys do NOT affect narrowing: the enhanced discriminant-key detection checks key presence in the shape (`k in optShape`), not whether the key's Zod type is required. An optional `capability` still appears in branch 1's shape, still counts as a discriminant (present in exactly 1 branch), and still enables narrowing.

### Validation and completion semantics

The document schema serves **both** validation (via `safeParse()` in `computeDiagnostics`) **and** completion (via `navigateSchema` + `schemaToCompletions`). The union design must be correct for both paths:

- **`safeParse` (diagnostics):** With optional discriminator keys, a binding without any variant key matches all branches (all discriminator keys are optional → missing is OK). `z.union` takes the first match. No false diagnostic errors. A binding WITH `capability: 'cap1'` also matches all branches (since `subCase` and `humanTask` are optional on their branches too), but Zod takes the first match — correct behavior.
- **Completion narrowing (`schemaToCompletions`):** Discriminant-key detection narrows based on shape key presence, independent of optionality. When `capability` is a sibling, only branch 1 has it in its shape → narrow to branch 1.
- **Schema navigation (`navigateSchema`):** ZodUnion handling tries each option and returns the first match. Common and variant fields remain navigable regardless of which variant is active.
- **Transient editing states:** `switchBindingTarget` in `yaml-editor.ts` removes the old variant key before setting the new one. During this transition, no variant key is present. With optional discriminator keys, `safeParse` succeeds → no flash of diagnostic errors.

### Generated Output Example

Before (current — flat):
```typescript
export const bindingSchema = z.object({
  name: z.string().optional(),
  on: triggerSchema,
  when: z.string().optional(),
  capability: z.string().optional(),
  subCase: subCaseSchema.optional(),
  humanTask: humanTaskSchema.optional(),
  conflictResolverStrategy: z.enum([...]).optional(),
  // ... other common fields
});
```

After (union-aware):
```typescript
const bindingCommon = z.object({
  name: z.string().optional(),
  on: triggerSchema,
  when: z.string().optional(),
  conflictResolverStrategy: z.enum([...]).optional(),
  // ... other common fields
});

export const bindingSchema = z.union([
  bindingCommon.extend({ capability: z.string().optional() }),
  bindingCommon.extend({ subCase: subCaseSchema.optional() }),
  bindingCommon.extend({ humanTask: humanTaskSchema.optional() }),
]);
```

### Nested Discriminators

When a type has discriminated unions at multiple levels (e.g., `Binding` has target variants AND `Trigger` within `on` has its own variants), the generator handles them independently. `typeToZod()` is recursive — when it processes the `Trigger` type for the `on` field, it checks the config again and produces a nested union.

### Config Loading

The discriminator config is loaded externally and passed to `generateFormatSchema` as a parameter, keeping the function synchronous and testable:

```typescript
// Signature change: added optional discriminatorConfig parameter
export function generateFormatSchema(
  project: Project,
  config: FormatConfig,
  discriminatorConfig?: Record<string, DiscriminatorRule>,
): string {
```

The main block loads configs with top-level `await` (the file uses ES module `import.meta.url`):

```typescript
for (const config of FORMATS) {
  let discriminatorConfig: Record<string, DiscriminatorRule> | undefined;
  if (config.discriminatorManifest) {
    const manifest = await import(resolve(__dirname, config.discriminatorManifest));
    discriminatorConfig = manifest.discriminators;
  }
  const output = generateFormatSchema(project, config, discriminatorConfig);
  const outPath = resolve(__dirname, config.outputFile);
  writeFileSync(outPath, output, 'utf-8');
  console.log(`Generated ${config.formatId} schema to ${outPath}`);
}
```

This design:
- **Keeps `generateFormatSchema` synchronous** — no signature break for callers or tests
- **Separates config loading from generation** — tests pass discriminator configs directly without filesystem access
- **`await import()` stays in the main block** — the entry point, which is the only place that needs async

### FORMATS Array Change

The `caseDefinition` entry gains `discriminatorManifest`:

```typescript
export const FORMATS: FormatConfig[] = [
  {
    formatId: 'caseDefinition',
    rootTypeName: 'CaseHub',
    sourceFile: '../../graph-stencil-case/src/types/generated/case-definition.ts',
    outputFile: '../src/schemas/case-definition.generated.ts',
    exportName: 'caseDefinitionDocumentSchema',
    discriminatorManifest: './discriminators/case-definition.js',
  },
  // org, htn, swf — unchanged (no discriminators)
];
```

Only `caseDefinition` gets a manifest. Org has no discriminators (RelationshipScope is not mutually exclusive). HTN has none identified. SWF uses hand-written schemas.

### Validation at Generation Time

The generator validates each discriminator config entry against the actual TypeScript type:

1. **Type exists:** ts-morph resolves the type name. If not found, error with "Discriminator config references unknown type: X".
2. **Discriminator keys exist:** Each discriminator key must be a property on the type. If not, error with "Discriminator key 'foo' not found on type X. Available properties: [...]".
3. **Discriminator keys are optional:** Each discriminator key should be optional on the source type (since they're mutually exclusive). Warn if a discriminator key is required — it may indicate a config error.

## Narrowing Algorithm Enhancement (pages-lsp)

**Cross-repo dependency:** This change is in `casehub-pages/packages/pages-lsp`, a different repository from this spec's project (`blocks-ui`). Tracked as casehubio/casehub-pages#TBD-narrowing-fix.

**Independent deployability:** The two changes are independently deployable in either order:
- **Narrowing fix alone** (without union schemas): Fixes the latent SWF narrowing bug (`then`/`if` shared keys). CaseDefinition schemas are still flat — no narrowing to do, no regression.
- **Union schemas alone** (without narrowing fix): Falls back to showing all branch completions (deduplicated) — identical to today's flat schema behavior. No regression, but no narrowing benefit until the fix lands.

### Current algorithm (`schemaToCompletions`, `schema-navigation.ts:285-297`)

```typescript
if (siblings && Object.keys(siblings).length > 0) {
  const siblingKeys = new Set(Object.keys(siblings));
  const matching = options.filter(opt => {
    const optShape = getShape(unwrap(opt));
    if (!optShape) return false;
    return [...siblingKeys].some(k => k in optShape);
  });
  if (matching.length === 1) {
    return schemaToCompletions(matching[0]!, siblings);
  }
}
```

This checks if ANY sibling key is in a branch's shape. With `.extend()`, common keys (`name`, `on`, `when`, etc.) appear in every branch. When siblings include `name` — which is essentially always — ALL branches match and narrowing never fires.

The same bug affects the hand-written SWF schema: shared `then`/`if` keys cause all task branches to match when those are siblings.

### Enhanced algorithm

Replace the filter with discriminant-key detection:

```typescript
if (siblings && Object.keys(siblings).length > 0) {
  const siblingKeys = new Set(Object.keys(siblings));

  // Build a map: key → how many branches contain it
  const keyBranchCount = new Map<string, number>();
  const keyBranch = new Map<string, z.ZodType>();
  for (const opt of options) {
    const optShape = getShape(unwrap(opt));
    if (!optShape) continue;
    for (const k of Object.keys(optShape)) {
      const count = (keyBranchCount.get(k) ?? 0) + 1;
      keyBranchCount.set(k, count);
      if (count === 1) keyBranch.set(k, opt);
    }
  }

  // Find if any sibling is a discriminant (unique to one branch)
  for (const sk of siblingKeys) {
    if (keyBranchCount.get(sk) === 1) {
      return schemaToCompletions(keyBranch.get(sk)!, siblings);
    }
  }
}
```

This finds keys that appear in exactly one branch's shape. If a sibling contains such a key, that branch is the match. For the binding union:
- `capability` appears only in branch 1 → discriminant
- `subCase` appears only in branch 2 → discriminant
- `humanTask` appears only in branch 3 → discriminant
- `name`, `on`, `when` appear in all branches → not discriminants

If siblings are `{name: "myBinding", on: {...}, capability: "cap1"}`:
- `name` → in 3 branches, not a discriminant
- `on` → in 3 branches, not a discriminant
- `capability` → in 1 branch → **discriminant match** → narrow to capability branch

The fallback behavior (show all completions, deduplicated) remains unchanged when no discriminant key is found in siblings.

## Format Registration Updates

### CaseDefinition

No import change needed — the import path stays the same, only the generated content changes:

```typescript
// formats/case-definition.ts — unchanged
import { caseDefinitionDocumentSchema } from '../schemas/case-definition.generated.js';
```

### Org

Same — `org.generated.ts` content unchanged (no discriminators for org).

### Dead Code Cleanup

The hand-written schema files are already unused — format registrations already import from `.generated.ts` files:
- `formats/case-definition.ts:3` imports from `../schemas/case-definition.generated.js`
- `formats/org.ts:3` imports from `../schemas/org.generated.js`
- `formats/htn.ts:2` imports from `../schemas/htn.generated.js`

The hand-written files have no remaining references:

- `schemas/case-definition.ts` — **delete** (dead code, not imported by any format registration)
- `schemas/org.ts` — **delete** (dead code, not imported by any format registration)
- `schemas/htn.ts` — **delete** (dead code, `formats/htn.ts` imports from `htn.generated.js`)
- `schemas/swf.ts` — **keep** (hand-written, actively used by `formats/swf.ts`, has union types the generated `swf.generated.ts` cannot produce)
- `schemas/swf.generated.ts` — **keep** (used by staleness tests as the generated reference; its flat `z.record(z.unknown())` for `do` tasks is correct output from the generator given the source type `do: Record<string, unknown>[]`)

This cleanup is independent of the union-aware generation — these files are dead code regardless. It is included here for completeness since the spec touches the schema directory.

### Discriminator JSON Cleanup

- `discriminators/case-definition.json` — **delete** (dead code with incorrect keys; replaced by TypeScript config)

## Testing

### Generator Tests

1. **Union generation:** Given a type with discriminator config, verify the output contains `z.union([...])` with the correct number of branches.
2. **Common field extraction:** Verify non-discriminator properties appear in the common base, not duplicated across branches.
3. **Nested discriminators:** Verify a type with unions at multiple levels produces nested unions.
4. **Validation:** Verify the generator errors when a discriminator key doesn't exist on the source type.

### Narrowing Algorithm Tests (pages-lsp)

Tests for the enhanced `schemaToCompletions` ZodUnion narrowing:

1. **Discriminant key present with realistic siblings:** Siblings `{name: "b1", on: {...}, capability: "cap1"}` → narrows to capability branch. Only capability-branch fields + common fields returned.
2. **No discriminant key present:** Siblings `{name: "b1", on: {...}}` → no narrowing, all branch completions returned (deduplicated).
3. **Multiple discriminant keys present:** Siblings `{capability: "x", subCase: {...}}` → first discriminant found narrows (this is a malformed binding, but should not crash).
4. **SWF task narrowing:** Siblings `{call: "http", then: "next"}` → narrows to call task branch despite shared `then` key.

### Completion Integration Tests

End-to-end narrowing through the full pipeline:

1. **No variant key present:** Completions include all variant keys + common fields.
2. **One variant key present with common siblings:** YAML has `name: myBinding`, `on: {...}`, `capability: myCapability`, cursor on new key — completions show only capability-branch fields, not `subCase` or `humanTask`.
3. **Common field navigation:** Navigating to a common field (e.g., `name`) works regardless of which variant is active.
4. **Nested narrowing:** Within a binding that has `capability` set, navigating to `on` and setting `contextChange` further narrows trigger completions.

### Validation Compatibility Tests

Verify that `safeParse()` succeeds for editing-state documents:

1. **Binding with no variant key:** `{ name: "b1", on: { contextChange: {} } }` — no capability, subCase, or humanTask. `safeParse` must succeed (no false diagnostic errors).
2. **Binding with one variant key:** `{ name: "b1", on: { contextChange: {} }, capability: "cap1" }` — `safeParse` succeeds.
3. **Trigger with no variant key:** `{ }` — no contextChange, cloudEvent, schedule, or scopeActivated. `safeParse` must succeed.
4. **Existing test fixtures:** All existing `generator-core.test.ts` `safeParse` test cases must continue to pass unchanged.

### Staleness Test

The existing staleness test pattern (compare generated output against fresh generation) continues to work. No change needed — the test verifies the committed `.generated.ts` matches what the generator produces.

## Scope — What This Issue Does NOT Cover

- **variantDispatchers wiring** — not needed. The `FormatRegistration.variantDispatchers` field and `SchemaRegistry.getVariantSchema()` are dead infrastructure never invoked by the completion pipeline. The enhanced narrowing algorithm makes them redundant. (Whether to remove the dead infrastructure is a separate cleanup decision.)
- **JSON manifest files** — not created. TypeScript configs replace the dead `case-definition.json`.
- **SWF schema restructuring** — SWF already has unions. The narrowing fix benefits SWF automatically.
- **WorkerFunction/McpTransport/ModelProvider discriminators** — these types aren't part of the CaseDefinition YAML schema type tree. They exist in the diagram editor's worker-function module. Tracked in casehubio/blocks-ui#TBD-worker-function-discriminators.
- **Zod v4 migration** — tracked separately in casehubio/casehub-pages#451. This work uses stable APIs (`z.union()`, `z.object()`, `.extend()`) that are identical across v3 and v4.
- **pages-lsp narrowing algorithm fix** — tracked as casehubio/casehub-pages#TBD-narrowing-fix. Independently deployable from the union schema generation (see §Narrowing Algorithm Enhancement for deployment ordering).

## References

- `packages/lsp-schemas/scripts/generate-domain-schemas.ts` — existing generator (290 lines, `discriminatorManifest` hook on line 15, existing `DiscriminatorRule`/`DiscriminatorManifest` interfaces on lines 18-24)
- `packages/lsp-schemas/scripts/discriminators/case-definition.json` — existing dead discriminator config (to be deleted)
- `packages/lsp-schemas/src/formats/case-definition.ts` — format registration (imports generated schema from `case-definition.generated.js`)
- `packages/lsp-schemas/src/formats/org.ts` — format registration (imports generated schema from `org.generated.js`)
- `packages/lsp-schemas/src/formats/htn.ts` — format registration (imports generated schema from `htn.generated.js`)
- `packages/lsp-schemas/src/formats/swf.ts` — format registration (imports hand-written schema from `swf.js`)
- `packages/lsp-schemas/src/schemas/case-definition.ts` — hand-written schema (dead code, to be deleted)
- `packages/lsp-schemas/src/schemas/org.ts` — hand-written schema (dead code, to be deleted)
- `packages/lsp-schemas/src/schemas/htn.ts` — hand-written schema (dead code, to be deleted)
- `packages/lsp-schemas/src/schemas/swf.ts` — hand-written SWF schema (kept, actively used, already has unions)
- `packages/lsp-schemas/src/schemas/swf.generated.ts` — generated SWF schema (kept for staleness tests)
- `casehub-pages/packages/pages-lsp/src/schema-navigation.ts:219-308` — `schemaToCompletions` ZodUnion handling (to be enhanced)
- `casehub-pages/packages/pages-lsp/src/schema-navigation.ts:185-192` — `navigateSchema` ZodUnion handling (works correctly, no change)
- `casehub-pages/packages/pages-lsp/src/completion.ts` — `handleCompletion` (no change)
- `casehub-pages/packages/pages-lsp/src/diagnostics.ts:40` — `computeDiagnostics` calls `format.documentSchema.safeParse()` (union schemas must remain compatible)
- `casehub-pages/packages/pages-lsp/src/types.ts` — FormatRegistration, VariantDispatch interfaces (no change)
- `packages/graph-stencil-case/src/types/generated/case-definition.ts` — CaseDefinition TypeScript types (source for generator)
- `packages/graph-stencil-case/src/worker-function/types.ts` — WorkerFunctionType, McpTransportType, ModelProviderKey
- `packages/graph-stencil-case/src/adapter/yaml-editor.ts` — switchBindingTarget, switchTriggerType (documents variant keys)
- `packages/graph-stencil-org/src/types.ts` — RelationshipScope (3 independently optional fields, NOT mutually exclusive)
- `docs/specs/issue-408-yaml-schema-completion/2026-09-05-yaml-schema-completion-design.md` — schema completion design
- [Zod v4 release notes](https://zod.dev/v4) — z.discriminatedUnion deprecation, z.union stability
- casehubio/casehub-pages#451 — Zod v4 migration (separate, not blocking)
