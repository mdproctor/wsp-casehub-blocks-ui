# Phase 0 — Schema Verification + TypeScript Type Generation

**Date:** 2026-08-03
**Issue:** #103 (Epic: Visual Diagram Editor — Domain Layer)
**Status:** Approved
**Parent spec:** `specs/2026-08-01-visual-diagram-editor-design.md` (parent workspace)

---

## 1. Problem

The visual diagram editor (Phase 2+) needs TypeScript types that accurately represent the CaseDefinition YAML structure. The JSON Schema at `engine/schema/src/main/resources/schema/CaseDefinition.yaml` is the canonical type definition, but it may be stale after the stages removal and recent model changes. The current placeholder types in `graph-stencil-case/src/types/case-definition.ts` cover ~20% of the schema's fields.

Phase 0 must verify the schema against the Java model and generate complete TypeScript types before any editor work begins.

## 2. Scope

Two deliverables:

1. **Structural audit** — verify every `$defs` entry in `CaseDefinition.yaml` against the corresponding Java model class in engine. Produce a findings list. Patch the schema in engine if discrepancies are found.
2. **TypeScript type generation** — generate complete types from the verified schema using `json-schema-to-typescript`. Checked-in output with a re-runnable script.

**Out of scope:** Cross-parser YAML compatibility test and React Flow + Lit bridge spike (both are pages-side, covered by casehub-pages#258).

## 3. Structural Audit

### 3.1 Method

Open the engine project in IntelliJ. For each `$defs` entry in the schema, navigate to the corresponding Java class and compare:

| Check | What to compare |
|-------|----------------|
| Field names | Schema properties vs Java fields/getters |
| Types | `string` → `String`, `integer` → `int`/`Integer`, `array` → `List<>`, `object` → nested class |
| Required | Schema `required` arrays vs Java `@NonNull` / constructor requirements |
| Enums | Schema `enum` values vs Java enum constants |
| Composition | Schema `$ref` vs Java class composition / `@JsonTypeInfo` |
| Extension points | `additionalProperties`/`unevaluatedProperties` vs Java's open-ended maps |
| `oneOf` constraints | Schema discriminated unions vs Java's polymorphic dispatch |

### 3.2 Schema $defs to audit

Root: `CaseDefinition` (top-level object)

Core domain types:
- `CaseDefinitionSpec`, `Binding`, `Worker`, `Milestone`, `Goal`, `SubCase`
- `Capability`, `HumanTask`, `CaseCompletion`, `GoalExpression`

Trigger types:
- `Trigger`, `ContextChangeTrigger`, `CloudEventTrigger`, `ScheduleTrigger`, `ScopeActivatedTrigger`

Policy/config types:
- `OutcomePolicy`, `ExecutionPolicy`, `RetryPolicy`, `Authorization`, `Cbr`, `Use`
- `LabelRule`, `InboundSignalMapping`

Agent types (code-generation directives — verify but exclude from TypeScript):
- `Agent`, `AgentModel`, `OpenAiModel`, `OllamaModel`, `AnthropicModel`, `MistralAiModel`, `GoogleAiGeminiModel`

### 3.3 Key risk

The design spec notes the schema may be stale after "stages removal". If the Java model no longer has a `stages` concept but the schema still references it (or vice versa), that is the primary thing to catch.

### 3.4 Output

A findings list committed to this spec directory. For each discrepancy:
- What the schema says
- What the Java model says
- Recommended schema patch

Schema patches are committed to the engine repo (separate session or flagged for engine work).

## 4. TypeScript Type Generation

### 4.1 Tool

`json-schema-to-typescript` (npm). Handles JSON Schema 2020-12, produces TypeScript interfaces from `$defs`.

### 4.2 Generator script

Location: `packages/graph-stencil-case/scripts/generate-types.ts`

Inputs:
- Path to `CaseDefinition.yaml` — defaults to `../../../engine/schema/src/main/resources/schema/CaseDefinition.yaml` (peer repo convention). Accepts a CLI argument override.

Processing:
1. Read YAML, parse to JSON Schema object (using `yaml` npm package)
2. Strip `_codegen*` properties from `CaseDefinitionSpec` before generation (these are engine-only code generation directives — `_codegenAgent`, `_codegenAgentModel`, `_codegenOpenAi`, `_codegenOllama`, `_codegenAnthropic`, `_codegenMistral`, `_codegenGoogleAi`)
3. Run `json-schema-to-typescript` with strict settings (`strictIndexSignatures: true`, `enableConstEnums: false`)
4. Write output to `src/types/generated/case-definition.ts`

### 4.3 Output structure

```
packages/graph-stencil-case/
  scripts/
    generate-types.ts          ← generator script
  src/
    types/
      generated/
        case-definition.ts     ← generated output (checked in)
      case-definition.ts       ← re-export barrel + convenience aliases
      index.ts                 ← public type exports
```

The re-export barrel (`src/types/case-definition.ts`) replaces the current placeholder file. It re-exports the generated types and may add narrowed convenience types for the editor (e.g., `EditorBinding` that omits fields not relevant to the visual editor).

### 4.4 Special cases

| Schema feature | Generated output | Rationale |
|----------------|-----------------|-----------|
| Worker `additionalProperties: true` | `[k: string]: unknown` index signature | Worker functions are plugin-supplied — editor shouldn't constrain |
| CaseDefinitionSpec `unevaluatedProperties: true` | `[k: string]: unknown` index signature | Extension point for plugin modules |
| `_codegen*` properties | Stripped before generation | Engine-only code generation directives |
| `oneOf` on Binding (capability/subCase/humanTask) | Union type | Binding has exactly one target type |
| `oneOf` on Trigger (contextChange/cloudEvent/schedule/scopeActivated) | Union type | Trigger has exactly one type |
| `oneOf` on HumanTask (title/titleExpression/templateRef) | Union type | HumanTask mode is exclusive |
| Agent model types | Excluded (stripped with `_codegen*`) | Not part of the definition-view editor |

### 4.5 Package.json script

```json
{
  "generate:types": "tsx scripts/generate-types.ts"
}
```

### 4.6 CI verification (optional)

Add to the build: `yarn generate:types && git diff --exit-code src/types/generated/`. Fails if the checked-in types don't match the script output. Catches forgotten re-generation after schema changes.

## 5. Integration

After generation:

1. Replace the placeholder `case-definition.ts` with the re-export barrel
2. Update `src/index.ts` to export the full generated type set
3. Stencil property schemas in `stencils/index.ts` remain hand-written JSON Schema objects (they are subsets for the property panel, not full types) — but field names and constraints should align with the generated types
4. The `CaseAdapter` in `adapter/case-adapter.ts` will consume the generated types in Phase 2

## 6. Dependencies

- `json-schema-to-typescript` — devDependency in graph-stencil-case
- `yaml` — already a project dependency (used for YAML parsing throughout blocks-ui)
- `tsx` — devDependency for running the generator script
- Engine repo checkout at `../../engine/` — required only for running the generator, not for building
