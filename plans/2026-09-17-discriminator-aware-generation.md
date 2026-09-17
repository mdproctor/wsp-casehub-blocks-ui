# Discriminator-Aware Schema Generation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #158 — discriminator manifests for narrowed completion
**Issue group:** #158, #159, #160, #161

**Goal:** Enhance the ts-morph generator to produce `z.union()` schemas at discriminated type points, enabling the pages-lsp completion engine to narrow suggestions when a variant key is present.

**Architecture:** The existing generator walks TypeScript interfaces via ts-morph and produces flat `z.object()` Zod schemas. A per-format discriminator config declares which type names have key-presence variants. When the generator encounters a configured type, it partitions properties into common and variant groups, producing `z.union([common.extend({variant1}), ...])`. The pages-lsp ZodUnion handler provides the narrowing — a separate pages-lsp algorithm fix (pages#TBD) is needed for full narrowing, but the union schemas are independently deployable.

**Tech Stack:** TypeScript, ts-morph, Zod v3 (stable `z.union()`/`.extend()` API), Vitest

## Global Constraints

- Generated code uses only stable Zod public API: `z.object()`, `.extend()`, `z.union()`, `.optional()`. No `._def` access.
- Discriminator keys stay **optional** on their variant schema to avoid false diagnostics during mid-edit states.
- SWF keeps its hand-written schema — source types are too flat for generation.
- The `typeToZod` function signature gains an optional `discriminatorConfig` parameter; existing callers are unaffected.

---

## Batch 1: Generator Union Support

### Task 1: Discriminator config + union generation helper (TDD)

**Files:**
- Create: `packages/lsp-schemas/scripts/discriminators/case-definition.ts`
- Modify: `packages/lsp-schemas/scripts/generate-domain-schemas.ts` (add helper function, update `typeToZod` signature)
- Test: `packages/lsp-schemas/test/generator-core.test.ts` (add discriminator tests)

**Interfaces:**
- Produces: `DiscriminatorRule` type (exported from config), `tryBuildDiscriminatedUnion()` helper (internal to generator), updated `typeToZod(type, depth, visited, discriminatorConfig?)` signature

- [ ] **Step 1: Write the discriminator config file**

Create `packages/lsp-schemas/scripts/discriminators/case-definition.ts`:

```typescript
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

- [ ] **Step 2: Write the failing test for union generation**

Add to `packages/lsp-schemas/test/generator-core.test.ts`:

```typescript
import type { DiscriminatorRule } from '../scripts/discriminators/case-definition.js';

describe('discriminator-aware generation', () => {
  const project = new Project({ useInMemoryFileSystem: true });

  function zodForTypeWithDiscriminators(
    typeCode: string,
    typeName: string,
    discriminatorConfig: Record<string, DiscriminatorRule>,
  ): string {
    const file = project.createSourceFile(
      'disc.ts',
      typeCode,
      { overwrite: true },
    );
    const typeAlias = file.getTypeAliasOrThrow(typeName);
    return typeToZod(typeAlias.getType(), 0, new Set(), discriminatorConfig);
  }

  it('produces z.union for key-presence discriminated type', () => {
    const code = `
      export type Target = {
        name?: string;
        capability?: string;
        subCase?: { ns: string };
        humanTask?: { title: string };
      };
    `;
    const config: Record<string, DiscriminatorRule> = {
      Target: { strategy: 'key-presence', discriminatorKeys: ['capability', 'subCase', 'humanTask'] },
    };
    const result = zodForTypeWithDiscriminators(code, 'Target', config);
    expect(result).toContain('z.union(');
    expect(result).toContain('.extend(');
    expect(result).toContain('capability');
    expect(result).toContain('subCase');
    expect(result).toContain('humanTask');
    // Common field should NOT be in extend calls
    expect(result.match(/name/g)?.length).toBe(1);
  });

  it('keeps discriminator keys optional on variant schemas', () => {
    const code = `
      export type Target = {
        name?: string;
        capability?: string;
        subCase?: { ns: string };
      };
    `;
    const config: Record<string, DiscriminatorRule> = {
      Target: { strategy: 'key-presence', discriminatorKeys: ['capability', 'subCase'] },
    };
    const result = zodForTypeWithDiscriminators(code, 'Target', config);
    // capability should be optional in its branch
    expect(result).toContain('capability: z.string().optional()');
  });

  it('produces flat z.object when type not in config', () => {
    const code = `export type Plain = { a: string; b?: number };`;
    const config: Record<string, DiscriminatorRule> = {};
    const result = zodForTypeWithDiscriminators(code, 'Plain', config);
    expect(result).toContain('z.object(');
    expect(result).not.toContain('z.union(');
  });

  it('handles intersection types with index signatures', () => {
    const code = `
      type Base = {
        name?: string;
        capability?: string;
        subCase?: string;
        [k: string]: unknown;
      };
      type Extra = { [k: string]: unknown };
      export type Binding = Base & Extra;
    `;
    const config: Record<string, DiscriminatorRule> = {
      Binding: { strategy: 'key-presence', discriminatorKeys: ['capability', 'subCase'] },
    };
    const result = zodForTypeWithDiscriminators(code, 'Binding', config);
    expect(result).toContain('z.union(');
  });

  it('produces correct number of union branches', () => {
    const code = `
      export type Trigger = {
        contextChange?: {};
        cloudEvent?: {};
        schedule?: {};
        scopeActivated?: {};
      };
    `;
    const config: Record<string, DiscriminatorRule> = {
      Trigger: { strategy: 'key-presence', discriminatorKeys: ['contextChange', 'cloudEvent', 'schedule', 'scopeActivated'] },
    };
    const result = zodForTypeWithDiscriminators(code, 'Trigger', config);
    // 4 extend calls = 4 branches
    const extendCount = (result.match(/\.extend\(/g) || []).length;
    expect(extendCount).toBe(4);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/lsp-schemas run test -- --grep "discriminator-aware"`
Expected: FAIL — `typeToZod` doesn't accept 4th parameter yet

- [ ] **Step 4: Implement the helper and update typeToZod**

In `packages/lsp-schemas/scripts/generate-domain-schemas.ts`:

1. Replace the existing dead `DiscriminatorRule` interface (lines 18-24) with an import:

```typescript
import type { DiscriminatorRule } from './discriminators/case-definition.js';
```

2. Add the helper function after `getSchemaVarName`:

```typescript
function tryBuildDiscriminatedUnion(
  type: Type,
  nonIndexProps: MorphSymbol[],
  depth: number,
  visited: Set<string>,
  discriminatorConfig?: Record<string, DiscriminatorRule>,
): string | null {
  if (!discriminatorConfig) return null;

  const symbol = type.getSymbol() || type.getAliasSymbol();
  const symbolName = symbol?.getName() ?? '';
  if (!symbolName || symbolName.startsWith('__')) return null;

  const rule = discriminatorConfig[symbolName];
  if (!rule) return null;

  const discKeys = new Set(rule.discriminatorKeys);
  const commonProps = nonIndexProps.filter(p => !discKeys.has(p.getName()));
  const variantProps = nonIndexProps.filter(p => discKeys.has(p.getName()));

  if (variantProps.length === 0) return null;

  const indent = '  '.repeat(depth + 2);
  const closingIndent = '  '.repeat(depth + 1);

  // Build common base
  const commonFields = commonProps
    .map(p => propToZodField(p, depth + 1, visited))
    .filter(Boolean);

  const commonVar = `${closingIndent}const _common = z.object({\n${indent}${commonFields.join(`,\n${indent}`)},\n${closingIndent}})`;

  // Build per-variant branches
  const branches = variantProps.map(p => {
    const field = propToZodField(p, depth + 1, visited);
    return `_common.extend({ ${field} })`;
  });

  return `(() => {\n${commonVar};\n${closingIndent}return z.union([\n${indent}${branches.join(`,\n${indent}`)},\n${closingIndent}]);\n${closingIndent}})()`;
}
```

3. Update `typeToZod` signature to accept optional config:

```typescript
export function typeToZod(
  type: Type,
  depth: number,
  visited: Set<string>,
  discriminatorConfig?: Record<string, DiscriminatorRule>,
): string {
```

4. In the `type.isIntersection()` branch (line 109), before the final `z.object(...)` return, add the discriminator check:

```typescript
// After computing nonIndexProps and fields, before the return:
const unionResult = tryBuildDiscriminatedUnion(type, nonIndexProps, depth, visited, discriminatorConfig);
if (unionResult) return unionResult;
```

5. In the `type.isObject()` branch (line 131), before the final `z.object(...)` return, add the same check:

```typescript
// After computing nonIndexProps, before generating fields:
const nonIndexPropsForDisc = props.filter(p => !isIndexSignature(p) && !p.getName().startsWith('__@'));
const unionResult = tryBuildDiscriminatedUnion(type, nonIndexPropsForDisc, depth, visited, discriminatorConfig);
if (unionResult) return unionResult;
```

6. Thread `discriminatorConfig` through recursive `typeToZod` calls in both branches — replace `typeToZod(baseType, depth, visited)` with `typeToZod(baseType, depth, visited, discriminatorConfig)` in `propToZodField`:

```typescript
export function propToZodField(
  prop: MorphSymbol,
  depth: number,
  visited: Set<string>,
  discriminatorConfig?: Record<string, DiscriminatorRule>,
): string {
  // ... existing code ...
  let zodType = typeToZod(baseType, depth, visited, discriminatorConfig);
  // ...
}
```

And update all `propToZodField` call sites to pass `discriminatorConfig`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/lsp-schemas run test -- --grep "discriminator-aware"`
Expected: PASS — all 5 discriminator tests green

- [ ] **Step 6: Run full test suite to verify no regressions**

Run: `yarn workspace @casehubio/lsp-schemas run test`
Expected: All existing tests still pass

- [ ] **Step 7: Commit**

```bash
git add packages/lsp-schemas/scripts/discriminators/case-definition.ts
git add packages/lsp-schemas/scripts/generate-domain-schemas.ts
git add packages/lsp-schemas/test/generator-core.test.ts
git commit -m "feat(lsp-schemas): discriminator-aware union generation in typeToZod

Add discriminator config for CaseDefinition (Binding, Trigger).
typeToZod produces z.union([common.extend({variant}), ...]) when a
type is in the config. Discriminator keys stay optional to avoid
false diagnostics during mid-edit states.

Refs #158"
```

---

### Task 2: Wire config into generator pipeline + regenerate schemas

**Files:**
- Modify: `packages/lsp-schemas/scripts/generate-domain-schemas.ts` (update `generateFormatSchema` + FORMATS + main block)
- Modify: `packages/lsp-schemas/src/schemas/case-definition.generated.ts` (regenerated — will contain unions)
- Test: `packages/lsp-schemas/test/case-definition.test.ts` (add safeParse compatibility tests)
- Test: `packages/lsp-schemas/test/staleness.test.ts` (verify staleness test still passes)

**Interfaces:**
- Consumes: `discriminators` export from `scripts/discriminators/case-definition.ts`, `tryBuildDiscriminatedUnion` helper from Task 1
- Produces: Regenerated `case-definition.generated.ts` with `z.union()` for Binding and Trigger types

- [ ] **Step 1: Write failing safeParse compatibility tests**

Add to `packages/lsp-schemas/test/case-definition.test.ts`:

```typescript
import { caseDefinitionDocumentSchema } from '../src/schemas/case-definition.generated.js';

describe('CaseDefinition schema safeParse compatibility', () => {
  it('accepts binding with no variant key (mid-edit)', () => {
    const doc = {
      dsl: 'casehub/case',
      namespace: 'test',
      name: 'case1',
      version: '1.0',
      spec: {
        bindings: [{ name: 'b1', on: { contextChange: {} } }],
      },
    };
    const result = caseDefinitionDocumentSchema.safeParse(doc);
    expect(result.success).toBe(true);
  });

  it('accepts binding with capability variant', () => {
    const doc = {
      dsl: 'casehub/case',
      namespace: 'test',
      name: 'case1',
      version: '1.0',
      spec: {
        bindings: [{
          name: 'b1',
          on: { contextChange: {} },
          capability: 'review',
        }],
      },
    };
    const result = caseDefinitionDocumentSchema.safeParse(doc);
    expect(result.success).toBe(true);
  });

  it('accepts binding with subCase variant', () => {
    const doc = {
      dsl: 'casehub/case',
      namespace: 'test',
      name: 'case1',
      version: '1.0',
      spec: {
        bindings: [{
          name: 'b1',
          on: { contextChange: {} },
          subCase: { namespace: 'ns', name: 'sub', version: '1.0' },
        }],
      },
    };
    const result = caseDefinitionDocumentSchema.safeParse(doc);
    expect(result.success).toBe(true);
  });

  it('accepts trigger with no variant key (mid-edit)', () => {
    const doc = {
      dsl: 'casehub/case',
      namespace: 'test',
      name: 'case1',
      version: '1.0',
      spec: {
        bindings: [{ name: 'b1', on: {} }],
      },
    };
    const result = caseDefinitionDocumentSchema.safeParse(doc);
    expect(result.success).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify current state (they should pass with flat schema)**

Run: `yarn workspace @casehubio/lsp-schemas run test -- --grep "safeParse compatibility"`
Expected: PASS — flat schema accepts all these inputs

- [ ] **Step 3: Update generateFormatSchema to accept discriminator config**

In `packages/lsp-schemas/scripts/generate-domain-schemas.ts`, update `generateFormatSchema`:

```typescript
export function generateFormatSchema(
  project: Project,
  config: FormatConfig,
  discriminatorConfig?: Record<string, DiscriminatorRule>,
): string {
  // ... existing sourceFile + rootType resolution ...

  lazyRefs.clear();
  const visited = new Set<string>();
  const zodCode = typeToZod(rootType, 0, visited, discriminatorConfig);

  // ... existing prelude + return ...
}
```

- [ ] **Step 4: Update FORMATS entry for caseDefinition**

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

- [ ] **Step 5: Update main block to load discriminator configs**

Replace the main block (lines 273-289) with:

```typescript
if (typeof process !== 'undefined' && process.argv[1] &&
    import.meta.url === `file://${process.argv[1]}`) {
  const project = new Project({
    tsConfigFilePath: resolve(__dirname, '../tsconfig.generator.json'),
  });

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

  if (FORMATS.length === 0) {
    console.log('No formats configured yet.');
  }
}
```

- [ ] **Step 6: Regenerate the schemas**

Run: `yarn workspace @casehubio/lsp-schemas run generate`

Inspect `packages/lsp-schemas/src/schemas/case-definition.generated.ts` — verify it contains `z.union(` for Binding and Trigger types.

- [ ] **Step 7: Run safeParse compatibility tests**

Run: `yarn workspace @casehubio/lsp-schemas run test -- --grep "safeParse compatibility"`
Expected: PASS — union schema still accepts all editing states

- [ ] **Step 8: Run full test suite including staleness**

Run: `yarn workspace @casehubio/lsp-schemas run test`
Expected: All tests pass, including staleness test (generated output matches committed file)

- [ ] **Step 9: Run full project typecheck and build**

Run: `yarn typecheck && yarn build`
Expected: No type errors, clean build

- [ ] **Step 10: Commit**

```bash
git add packages/lsp-schemas/scripts/generate-domain-schemas.ts
git add packages/lsp-schemas/src/schemas/case-definition.generated.ts
git add packages/lsp-schemas/test/case-definition.test.ts
git commit -m "feat(lsp-schemas): wire discriminator config into generator pipeline

generateFormatSchema accepts optional discriminatorConfig.
CaseDefinition FORMATS entry references case-definition discriminator
manifest. Regenerated schema has z.union() for Binding and Trigger.
safeParse compatibility verified for mid-edit states.

Refs #158"
```

---

## Batch 2: Dead Code Cleanup

### Task 3: Delete dead schemas, dead JSON discriminator, dead interfaces

**Files:**
- Delete: `packages/lsp-schemas/src/schemas/case-definition.ts` (dead — `formats/case-definition.ts` imports from `.generated.js`)
- Delete: `packages/lsp-schemas/src/schemas/org.ts` (dead — `formats/org.ts` imports from `.generated.js`)
- Delete: `packages/lsp-schemas/src/schemas/htn.ts` (dead — `formats/htn.ts` imports from `.generated.js`)
- Delete: `packages/lsp-schemas/discriminators/case-definition.json` (dead — replaced by TypeScript config)

**Interfaces:**
- Consumes: nothing (these files have no live importers)
- Produces: nothing (pure cleanup)

- [ ] **Step 1: Verify no live imports reference the dead files**

Run:
```bash
grep -r "from.*schemas/case-definition'" packages/lsp-schemas/src/ packages/lsp-schemas/test/
grep -r "from.*schemas/org'" packages/lsp-schemas/src/ packages/lsp-schemas/test/
grep -r "from.*schemas/htn'" packages/lsp-schemas/src/ packages/lsp-schemas/test/
grep -r "case-definition.json" packages/lsp-schemas/
```

Expected: No matches (all imports use `.generated.js` variants). If any matches found, update them before deleting.

- [ ] **Step 2: Delete the dead files**

```bash
rm packages/lsp-schemas/src/schemas/case-definition.ts
rm packages/lsp-schemas/src/schemas/org.ts
rm packages/lsp-schemas/src/schemas/htn.ts
rm packages/lsp-schemas/discriminators/case-definition.json
```

- [ ] **Step 3: Run full test suite to verify no breakage**

Run: `yarn workspace @casehubio/lsp-schemas run test`
Expected: All tests pass — no test imported these files

- [ ] **Step 4: Run full project typecheck**

Run: `yarn typecheck`
Expected: Clean — no file references the deleted schemas

- [ ] **Step 5: Commit**

```bash
git add -u packages/lsp-schemas/src/schemas/ packages/lsp-schemas/discriminators/
git commit -m "chore(lsp-schemas): delete dead hand-written schemas and JSON discriminator

case-definition.ts, org.ts, htn.ts were unused — format registrations
import from .generated.ts files. discriminators/case-definition.json
was dead code with incorrect keys, replaced by TypeScript config.

Refs #158"
```

---

## References

- [2026-09-17-discriminator-aware-schema-generation-design.md] — design spec this plan implements
- `packages/lsp-schemas/scripts/generate-domain-schemas.ts` — generator (290 lines, `typeToZod` at line 45, intersection handler at line 109, object handler at line 131)
- `packages/lsp-schemas/test/generator-core.test.ts` — existing generator tests (`zodForType` pattern)
- `packages/lsp-schemas/test/case-definition.test.ts` — existing format + completion tests
- `packages/lsp-schemas/src/schemas/case-definition.generated.ts` — generated schema (target of regeneration)
- `packages/lsp-schemas/discriminators/case-definition.json` — dead JSON config (to be deleted)
- `packages/graph-stencil-case/src/types/generated/case-definition.ts` — CaseDefinition TypeScript types (generator source)
- casehubio/casehub-pages#451 — Zod v4 migration (independent)
- casehubio/blocks-ui#158 — focal issue
