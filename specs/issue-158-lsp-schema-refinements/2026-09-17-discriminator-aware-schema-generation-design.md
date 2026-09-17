# Design: Discriminator-Aware Schema Generation for Narrowed LSP Completion

**Issue:** casehubio/blocks-ui#158
**Date:** 2026-09-17
**Branch:** issue-158-lsp-schema-refinements

## Overview

Enhance the `generate-domain-schemas.ts` generator to produce `z.union()` schemas at discriminated type points, enabling the existing pages-lsp completion engine to narrow suggestions based on which variant key is present in the YAML. No new infrastructure (variantDispatchers, JSON manifests) is needed — the existing `schemaToCompletions` ZodUnion sibling-based narrowing handles it automatically.

## Problem

The generated Zod schemas for CaseDefinition, Org, and HTN produce flat `z.object()` types where mutually exclusive optional keys all appear on the same object. The completion engine shows all properties simultaneously — typing `capability:` inside a binding still shows `subCase` and `humanTask` as suggestions.

## Solution

The generator reads a per-format discriminator config that declares which TypeScript type names have key-presence variants. At those types, it splits properties into common and variant groups, producing `z.union([common.extend({variant1}), common.extend({variant2}), ...])` instead of a flat object.

The pages-lsp completion engine already narrows `ZodUnion` completions by matching sibling keys against each option's shape (`schema-navigation.js:273-298`). When only one option's shape contains a sibling key, completions narrow to that option. This is the same mechanism the hand-written SWF schema already uses successfully.

## Discriminator Config

A TypeScript config file per format, referenced by `FormatConfig.discriminatorManifest` (field already exists on line 15 of the generator):

```typescript
// packages/lsp-schemas/scripts/discriminators/case-definition.ts

export interface DiscriminatorRule {
  variantKeys: string[];
}

export const discriminators: Record<string, DiscriminatorRule> = {
  Binding: {
    variantKeys: ['capability', 'subCase', 'humanTask'],
  },
  Trigger: {
    variantKeys: ['contextChange', 'cloudEvent', 'schedule', 'scopeActivated'],
  },
};
```

Each entry maps a **TypeScript type name** (as seen by ts-morph) to its variant keys. The remaining properties on that type become common fields shared across all variants.

### CaseDefinition Discriminators

| TypeScript Type | Variant Keys | Strategy |
|----------------|-------------|----------|
| `Binding` | `capability`, `subCase`, `humanTask` | key-presence |
| `Trigger` | `contextChange`, `cloudEvent`, `schedule`, `scopeActivated` | key-presence |

The issue body lists 5 discriminated unions (adding WorkerFunction, McpTransport, ModelProvider), but the current CaseDefinition TypeScript types (`graph-stencil-case/src/types/generated/case-definition.ts`) don't include worker function types — those are in `graph-stencil-case/src/worker-function/types.ts`, which is a separate type hierarchy not part of the `CaseHub` root type that the generator walks. Worker function types are only reachable through the diagram editor, not the YAML document schema.

The generator walks from the `CaseHub` root type. Only `Binding` and `Trigger` are discriminated unions within that type tree. If the worker function types are later added to the CaseDefinition YAML schema (via the engine's `CaseDefinition.yaml`), they'll automatically appear in the generated types and can be added to the discriminator config at that point.

### Org Discriminators

| TypeScript Type | Variant Keys | Strategy |
|----------------|-------------|----------|
| `RelationshipScope` | `capabilityName`, `domain`, `custom` | key-presence |

### HTN Discriminators

None identified in the current type tree. HTN types (`HtnTask`, `HtnMethod`) don't have key-presence unions.

### SWF

No generator change. SWF continues using its hand-written schema (`lsp-schemas/src/schemas/swf.ts`) which already has a proper `z.union([callTaskSchema, setTaskSchema, switchTaskSchema, ...])`. The source TypeScript type is `do: Record<string, unknown>[]` — too flat for the generator to produce union types from.

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

When processing an object type (the `type.isObject()` branch, line 131), the generator checks if the type's symbol name exists in the loaded discriminator config. If so:

1. **Partition properties** into common (not in `variantKeys`) and variant (in `variantKeys`).
2. **Generate a common base schema** from the common properties: `z.object({ name: z.string(), on: ..., when: ... })`.
3. **Generate per-variant schemas** by extending the common base with each variant key and its type: `commonSchema.extend({ capability: z.string() })`.
4. **Emit `z.union([...])`** wrapping all variant schemas.

Variant keys that are optional on the source type (they always are, since only one is present at runtime) become **required** on their variant schema — if you're in the `capability` variant, `capability` is not optional. The other variant keys are absent (not optional — absent).

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
  bindingCommon.extend({ capability: z.string() }),
  bindingCommon.extend({ subCase: subCaseSchema }),
  bindingCommon.extend({ humanTask: humanTaskSchema }),
]);
```

### Nested Discriminators

When a type has discriminated unions at multiple levels (e.g., `Binding` has target variants AND `Trigger` within `on` has its own variants), the generator handles them independently. `typeToZod()` is recursive — when it processes the `Trigger` type for the `on` field, it checks the config again and produces a nested union.

### Config Loading

The generator loads the config dynamically based on `FormatConfig.discriminatorManifest`:

```typescript
// In generateFormatSchema():
let discriminatorConfig: Record<string, DiscriminatorRule> = {};
if (config.discriminatorManifest) {
  const manifest = await import(resolve(__dirname, config.discriminatorManifest));
  discriminatorConfig = manifest.discriminators;
}
```

The config path is added to `FormatConfig`:

```typescript
{
  formatId: 'caseDefinition',
  rootTypeName: 'CaseHub',
  sourceFile: '../../graph-stencil-case/src/types/generated/case-definition.ts',
  outputFile: '../src/schemas/case-definition.generated.ts',
  exportName: 'caseDefinitionDocumentSchema',
  discriminatorManifest: './discriminators/case-definition.js',
},
```

### Validation at Generation Time

The generator validates each discriminator config entry against the actual TypeScript type:

1. **Type exists:** ts-morph resolves the type name. If not found, error with "Discriminator config references unknown type: X".
2. **Variant keys exist:** Each variant key must be a property on the type. If not, error with "Variant key 'foo' not found on type X. Available properties: [...]".
3. **Variant keys are optional:** Each variant key should be optional on the source type (since they're mutually exclusive). Warn if a variant key is required — it may indicate a config error.

## Format Registration Updates

### CaseDefinition

Switch from the generated schema import to the new union-aware generated schema. The import path doesn't change — only the generated content does:

```typescript
// formats/case-definition.ts — no import change needed
import { caseDefinitionDocumentSchema } from '../schemas/case-definition.generated.js';
```

### Org

Same — `org.generated.ts` now contains `RelationshipScope` as a union.

### Hand-Written Schema Cleanup

After the generated schemas produce unions, the hand-written schema files become redundant for case and org:

- `schemas/case-definition.ts` — **delete** (replaced by union-aware `case-definition.generated.ts`)
- `schemas/org.ts` — **delete** (replaced by union-aware `org.generated.ts`)
- `schemas/swf.ts` — **keep** (hand-written, already has unions, no generated equivalent)
- `schemas/htn.ts` — check if it exists and whether it adds value over the generated schema

The format registration imports switch from `.ts` to `.generated.ts` where applicable. For SWF, the import stays on the hand-written schema.

## Testing

### Generator Tests

1. **Union generation:** Given a type with discriminator config, verify the output contains `z.union([...])` with the correct number of branches.
2. **Common field extraction:** Verify non-variant properties appear in the common base, not duplicated across branches.
3. **Nested discriminators:** Verify a type with unions at multiple levels produces nested unions.
4. **Validation:** Verify the generator errors when a variant key doesn't exist on the source type.

### Completion Narrowing Tests

Integration tests verifying the end-to-end narrowing works:

1. **No variant key present:** Completions include all variant keys + common fields.
2. **One variant key present:** Completions narrow to that variant's fields + common fields. Other variant keys excluded.
3. **Common field navigation:** Navigating to a common field (e.g., `name`) works regardless of which variant is active.
4. **Nested narrowing:** Within a binding that has `capability` set, navigating to `on` and setting `contextChange` further narrows trigger completions.

### Staleness Test

The existing staleness test pattern (compare generated output against fresh generation) continues to work. No change needed — the test verifies the committed `.generated.ts` matches what the generator produces.

## Scope — What This Issue Does NOT Cover

- **variantDispatchers wiring** — not needed (D2). The `FormatRegistration.variantDispatchers` field stays unused.
- **JSON manifest files** — not created (D1). The schema structure IS the discrimination data.
- **SWF schema restructuring** — SWF already has unions. No change.
- **WorkerFunction/McpTransport/ModelProvider discriminators** — these types aren't part of the CaseDefinition YAML schema type tree. They exist in the diagram editor's worker-function module. If they're added to the YAML schema via the engine, the discriminator config can be extended.
- **Zod v4 migration** — tracked separately in casehubio/casehub-pages#451. This work uses stable APIs (`z.union()`, `z.object()`, `.extend()`) that are identical across v3 and v4.
- **pages-lsp changes** — none needed. The existing ZodUnion handling works.

## References

- `packages/lsp-schemas/scripts/generate-domain-schemas.ts` — existing generator (290 lines, `discriminatorManifest` hook on line 15)
- `packages/lsp-schemas/src/formats/case-definition.ts` — format registration (imports generated schema)
- `packages/lsp-schemas/src/schemas/case-definition.ts` — hand-written schema (to be deleted)
- `packages/lsp-schemas/src/schemas/swf.ts` — hand-written SWF schema (kept, already has unions)
- `node_modules/@casehubio/pages-lsp/dist/schema-navigation.js:273-298` — ZodUnion sibling narrowing
- `node_modules/@casehubio/pages-lsp/dist/types.d.ts` — FormatRegistration, VariantDispatch interfaces
- `packages/graph-stencil-case/src/types/generated/case-definition.ts` — CaseDefinition TypeScript types (source for generator)
- `packages/graph-stencil-case/src/worker-function/types.ts` — WorkerFunctionType, McpTransportType, ModelProviderKey
- `packages/graph-stencil-case/src/adapter/yaml-editor.ts` — switchBindingTarget, switchTriggerType (documents variant keys)
- `packages/graph-stencil-org/src/types.ts` — RelationshipScope (3 variant keys)
- `docs/specs/issue-407-lsp-ide-plugins/2026-09-10-lsp-ide-plugins-design.md` — LSP architecture spec
- `docs/specs/issue-408-yaml-schema-completion/2026-09-05-yaml-schema-completion-design.md` — schema completion design
- [ts-to-zod](https://github.com/fabien0102/ts-to-zod) — prior art for `@discriminator` JSDoc pattern
- [Zod v4 release notes](https://zod.dev/v4) — z.discriminatedUnion deprecation, z.union stability
- casehubio/casehub-pages#451 — Zod v4 migration (separate, not blocking)
