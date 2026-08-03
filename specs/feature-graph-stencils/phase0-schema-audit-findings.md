# Phase 0 — Schema Audit Findings

**Date:** 2026-08-03
**Schema:** `engine/schema/src/main/resources/schema/CaseDefinition.yaml`
**Verdict:** Schema is current. No patches needed.

## Key Finding

The Java model classes in `engine/schema/target/generated-sources/jsonschema2pojo/` are **generated from the same schema** via jsonschema2pojo. The schema is the single source of truth — there is no separate hand-written Java model to drift from.

The only hand-written class is `engine/schema/src/main/java/io/casehub/model/Worker.java`, which overrides the generated Worker with custom Jackson serialization (WorkerMarshaller) to handle plugin-supplied worker functions.

## Worker.java vs Schema

| Java field | In schema explicitly? | How handled |
|-----------|----------------------|-------------|
| `name` | Yes | required |
| `description` | Yes | |
| `capabilities` | Yes | required |
| `executionPolicy` | Yes | `$ref ExecutionPolicy` |
| `sequence` | Yes | |
| `contextType` | Yes | |
| `outputType` | Yes | |
| `inputSchema` | No | `additionalProperties: true` — plugin-supplied |
| `outputSchema` | No | `additionalProperties: true` — plugin-supplied |
| `workflow` | No | `additionalProperties: true` — SWF `do:` syntax |
| `agent` | No | `additionalProperties: true` — AI agent config |

Worker's `additionalProperties: true` is intentional — worker functions are plugin-supplied via the WorkerFunctionProviderRegistry. The TypeScript type will have an `[k: string]: unknown` index signature, which is correct for the editor.

## Stages Check

No reference to "stage" or "stages" in the schema. The stages concept was fully removed.

## Example YAML Verification

`document-processing.yaml` (design spec's reference example) parses cleanly. Workers use `do:` syntax for SWF-style workflow definitions — these fall under `additionalProperties: true` and are correctly untyped in the schema.

## Conclusion

No schema patches needed. Proceed directly to TypeScript type generation.
