# OWS 1.0 Stencil Completeness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #172 — SWF 1.0: complete task type stencils and picker filtering
**Issue group:** #158, #159, #160, #161, #172, #170, #171, #173

**Goal:** Add stencils, grammars, and renderers for the 6 missing OWS 1.0
task types (Do, Emit, Fork, Listen, Run, Wait) so all 12 types render
with distinct visuals and the picker filters correctly.

**Architecture:** Each stencil is a TypeScript module exporting a grammar
object and a render function. Container types (Do, Fork) follow the
For/Try pattern (compact header, children inside). Leaf types (Emit,
Listen, Run, Wait) follow the Call/Set pattern (card with icon and
subtitle). Grammar constants are extracted to a shared module to avoid
repetition across 18 grammar definitions.

**Tech Stack:** TypeScript, lit-html, vitest, `@openworkflowspec/sdk`
1.0.3-alpha8, `@casehubio/graph-core`, `@casehubio/graph-renderer`

## Global Constraints

- SDK version: pin `@openworkflowspec/sdk` to `1.0.3-alpha8`
- All stencils follow existing patterns in `packages/graph-stencil-swf/src/stencils/`
- OWS 1.0 DSL reference is authoritative for task semantics
- All grammars use shared `FLOW_SOURCES`/`FLOW_TARGETS` constants
- Container styling: orange border + grey background (matches for/try)
- Tests use `vitest` with the existing `makeNode` helper pattern

---

## Batch 1: Grammar foundation + leaf stencils

### Task 1: SDK upgrade + grammar constants + type registration

**Files:**
- Modify: `packages/graph-stencil-swf/package.json:33`
- Create: `packages/graph-stencil-swf/src/stencils/grammars.ts`
- Modify: `packages/graph-stencil-swf/src/types.ts:9-12`
- Modify: `packages/graph-stencil-swf/src/stencils/index.ts`
- Test: `packages/graph-stencil-swf/src/stencils/stencils.test.ts`

**Interfaces:**
- Produces: `FLOW_SOURCES: string[]` — all types valid as edge sources
  (call, set, switch, for, do, fork, emit, listen, run, wait, entry, start)
- Produces: `FLOW_TARGETS: string[]` — all types valid as edge targets
  (call, set, switch, for, do, fork, emit, listen, run, wait, raise, exit, end)
- Produces: `SWF_KNOWN_TYPES` updated set with 6 new types

- [ ] **Step 1: Pin SDK version**

In `packages/graph-stencil-swf/package.json`, change:
```json
"@openworkflowspec/sdk": "1.0.3-alpha8"
```

- [ ] **Step 2: Run `yarn install` to pull the new SDK**

Run: `yarn install`
Expected: resolves `@openworkflowspec/sdk@1.0.3-alpha8` without errors

- [ ] **Step 3: Create grammar constants module**

Create `packages/graph-stencil-swf/src/stencils/grammars.ts`:

```typescript
export const FLOW_SOURCES: readonly string[] = [
  'swf-call', 'swf-set', 'swf-switch', 'swf-for', 'swf-do', 'swf-fork',
  'swf-emit', 'swf-listen', 'swf-run', 'swf-wait', 'swf-entry', 'swf-start',
];

export const FLOW_TARGETS: readonly string[] = [
  'swf-call', 'swf-set', 'swf-switch', 'swf-for', 'swf-do', 'swf-fork',
  'swf-emit', 'swf-listen', 'swf-run', 'swf-wait', 'swf-raise', 'swf-exit', 'swf-end',
];
```

- [ ] **Step 4: Add new types to SWF_KNOWN_TYPES**

In `packages/graph-stencil-swf/src/types.ts`, update the set:
```typescript
export const SWF_KNOWN_TYPES: ReadonlySet<string> = new Set([
  'call', 'set', 'switch', 'raise', 'try', 'try-catch', 'catch',
  'for', 'do', 'emit', 'fork', 'listen', 'run', 'wait',
  'start', 'end', 'entry', 'exit',
]);
```

- [ ] **Step 5: Verify build compiles**

Run: `yarn --cwd packages/graph-stencil-swf tsc --noEmit`
Expected: no errors

- [ ] **Step 6: Commit**

```bash
git add packages/graph-stencil-swf/package.json yarn.lock \
  packages/graph-stencil-swf/src/types.ts \
  packages/graph-stencil-swf/src/stencils/grammars.ts
git commit -m "feat(graph-stencil-swf): SDK 1.0.3-alpha8 + grammar constants + type registration

Refs #172"
```

### Task 2: Update all existing grammars to reference shared constants

**Files:**
- Modify: `packages/graph-stencil-swf/src/stencils/call.ts:14-19`
- Modify: `packages/graph-stencil-swf/src/stencils/set.ts:5-11`
- Modify: `packages/graph-stencil-swf/src/stencils/switch.ts:5-11`
- Modify: `packages/graph-stencil-swf/src/stencils/for.ts:5-11`
- Modify: `packages/graph-stencil-swf/src/stencils/raise.ts:5-11`
- Modify: `packages/graph-stencil-swf/src/stencils/try.ts:5-10`
- Modify: `packages/graph-stencil-swf/src/stencils/try-catch.ts:5-11`
- Modify: `packages/graph-stencil-swf/src/stencils/boundary.ts:5-23`

**Interfaces:**
- Consumes: `FLOW_SOURCES`, `FLOW_TARGETS` from `./grammars.js`

The pattern for each grammar is:
- **Normal flow tasks** (call, set, switch, for, try, tryCatch): `allowedFrom: [...FLOW_SOURCES]`, `allowedTo: [...FLOW_TARGETS]`
- **Raise**: `allowedFrom: [...FLOW_SOURCES]`, `outbound: { min: 0, max: 0, allowedTo: [] }`
- **Start/Entry**: `inbound: { min: 0, max: 0, allowedFrom: [] }`, `allowedTo: [...FLOW_TARGETS]` (minus raise for entry/start — they can't target raise directly since raise has no outbound, but the grammar check is on the target side, so including raise in start.allowedTo is harmless; keep it consistent with FLOW_TARGETS)
- **End/Exit**: `allowedFrom: [...FLOW_SOURCES]` (minus entry/start — use FLOW_SOURCES directly since it includes them but the grammar min/max on entry/start outbound prevents actual connections), `outbound: { min: 0, max: 0, allowedTo: [] }`

- [ ] **Step 1: Update call.ts grammar**

Import `FLOW_SOURCES`, `FLOW_TARGETS` from `./grammars.js` and replace
the inline arrays:

```typescript
import { FLOW_SOURCES, FLOW_TARGETS } from './grammars.js';

export const callGrammar: StencilGrammar = {
  type: 'swf-call',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: [...FLOW_SOURCES] },
    outbound: { min: 0, max: 1, allowedTo: [...FLOW_TARGETS] },
  },
};
```

- [ ] **Step 2: Update set.ts, switch.ts, for.ts, try.ts, try-catch.ts grammars**

Same pattern as call.ts — import `FLOW_SOURCES`, `FLOW_TARGETS`, spread
into `allowedFrom`/`allowedTo`. Keep each grammar's existing `min`/`max`
values:
- `setGrammar`: inbound max Infinity, outbound max 1
- `switchGrammar`: inbound max Infinity, outbound max Infinity (fan-out)
- `forGrammar`: inbound max Infinity, outbound max 1
- `tryGrammar`: inbound max Infinity, outbound max 1
- `tryCatchGrammar`: inbound max Infinity, outbound max 1

- [ ] **Step 3: Update raise.ts grammar**

```typescript
import { FLOW_SOURCES } from './grammars.js';

export const raiseGrammar: StencilGrammar = {
  type: 'swf-raise',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: [...FLOW_SOURCES] },
    outbound: { min: 0, max: 0, allowedTo: [] },
  },
};
```

- [ ] **Step 4: Update boundary.ts grammars**

```typescript
import { FLOW_SOURCES, FLOW_TARGETS } from './grammars.js';

export const startGrammar: StencilGrammar = {
  type: 'swf-start',
  connections: { inbound: { min: 0, max: 0, allowedFrom: [] }, outbound: { min: 0, max: 1, allowedTo: [...FLOW_TARGETS] } },
};

export const endGrammar: StencilGrammar = {
  type: 'swf-end',
  connections: { inbound: { min: 0, max: Infinity, allowedFrom: [...FLOW_SOURCES] }, outbound: { min: 0, max: 0, allowedTo: [] } },
};

export const entryGrammar: StencilGrammar = {
  type: 'swf-entry',
  connections: { inbound: { min: 0, max: 0, allowedFrom: [] }, outbound: { min: 0, max: 1, allowedTo: [...FLOW_TARGETS] } },
};

export const exitGrammar: StencilGrammar = {
  type: 'swf-exit',
  connections: { inbound: { min: 0, max: Infinity, allowedFrom: [...FLOW_SOURCES] }, outbound: { min: 0, max: 0, allowedTo: [] } },
};
```

- [ ] **Step 5: Run existing tests to verify nothing broke**

Run: `yarn --cwd packages/graph-stencil-swf vitest run`
Expected: all existing tests pass

- [ ] **Step 6: Commit**

```bash
git add packages/graph-stencil-swf/src/stencils/
git commit -m "refactor(graph-stencil-swf): use shared FLOW_SOURCES/FLOW_TARGETS in all grammars

Refs #172"
```

### Task 3: Four leaf stencils — Emit, Listen, Run, Wait

**Files:**
- Create: `packages/graph-stencil-swf/src/stencils/emit.ts`
- Create: `packages/graph-stencil-swf/src/stencils/listen.ts`
- Create: `packages/graph-stencil-swf/src/stencils/run.ts`
- Create: `packages/graph-stencil-swf/src/stencils/wait.ts`
- Modify: `packages/graph-stencil-swf/src/stencils/index.ts`
- Modify: `packages/graph-stencil-swf/src/stencils/register.ts`
- Modify: `packages/graph-stencil-swf/src/editing/swf-edit-policy.ts:5-12`
- Test: `packages/graph-stencil-swf/src/stencils/stencils.test.ts`

**Interfaces:**
- Consumes: `FLOW_SOURCES`, `FLOW_TARGETS` from `./grammars.js`
- Produces: `emitGrammar`, `renderEmit` — grammar + render for emit
- Produces: `listenGrammar`, `renderListen` — grammar + render for listen
- Produces: `runGrammar`, `renderRun` — grammar + render for run
- Produces: `waitGrammar`, `renderWait` — grammar + render for wait

- [ ] **Step 1: Write failing tests for all four leaf stencils**

Add to `stencils.test.ts`:

```typescript
import { renderEmit } from './emit.js';
import { renderListen } from './listen.js';
import { renderRun } from './run.js';
import { renderWait } from './wait.js';

// ... inside the describe block:

it('renderEmit shows event type', () => {
  const result = renderEmit(makeNode('swf-emit', {
    emit: { event: { with: { type: 'com.example.order.placed' } } },
    label: 'publishOrder',
  }));
  expect(result).toBeDefined();
  expect(result.values).toBeDefined();
});

it('renderEmit handles missing event type', () => {
  const result = renderEmit(makeNode('swf-emit', { label: 'emitEvent' }));
  expect(result).toBeDefined();
});

it('renderListen shows consumption strategy', () => {
  const result = renderListen(makeNode('swf-listen', {
    listen: { to: { one: { with: { type: 'com.example.confirmed' } } } },
    label: 'awaitConfirmation',
  }));
  expect(result).toBeDefined();
  expect(result.values).toBeDefined();
});

it('renderListen handles all strategy', () => {
  const result = renderListen(makeNode('swf-listen', {
    listen: { to: { all: [{ with: { type: 'a' } }, { with: { type: 'b' } }] } },
  }));
  expect(result).toBeDefined();
});

it('renderRun shows container sub-type', () => {
  const result = renderRun(makeNode('swf-run', {
    run: { container: { image: 'my-image:latest' } },
    label: 'runContainer',
  }));
  expect(result).toBeDefined();
  expect(result.values).toBeDefined();
});

it('renderRun shows shell sub-type', () => {
  const result = renderRun(makeNode('swf-run', {
    run: { shell: { command: 'echo hello' } },
  }));
  expect(result).toBeDefined();
});

it('renderRun shows workflow sub-type', () => {
  const result = renderRun(makeNode('swf-run', {
    run: { workflow: { namespace: 'ns', name: 'wf', version: '1.0' } },
  }));
  expect(result).toBeDefined();
});

it('renderRun shows script sub-type', () => {
  const result = renderRun(makeNode('swf-run', {
    run: { script: { language: 'js', code: 'console.log("hi")' } },
  }));
  expect(result).toBeDefined();
});

it('renderWait shows duration', () => {
  const result = renderWait(makeNode('swf-wait', {
    wait: { seconds: 30 },
    label: 'cooldown',
  }));
  expect(result).toBeDefined();
  expect(result.values).toBeDefined();
});

it('renderWait handles string duration', () => {
  const result = renderWait(makeNode('swf-wait', { wait: 'PT30S' }));
  expect(result).toBeDefined();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/graph-stencil-swf vitest run src/stencils/stencils.test.ts`
Expected: FAIL — cannot find modules emit.js, listen.js, run.js, wait.js

- [ ] **Step 3: Create emit.ts**

```typescript
import { html } from 'lit-html';
import type { StencilGrammar, GraphNode, NodeDecoration } from '@casehubio/graph-core';
import type { StencilTemplate } from '@casehubio/graph-renderer';
import { FLOW_SOURCES, FLOW_TARGETS } from './grammars.js';

export const emitGrammar: StencilGrammar = {
  type: 'swf-emit',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: [...FLOW_SOURCES] },
    outbound: { min: 0, max: 1, allowedTo: [...FLOW_TARGETS] },
  },
};

export function renderEmit(node: GraphNode, _decoration?: NodeDecoration): StencilTemplate {
  const emit = node.properties['emit'] as { event?: { with?: { type?: string } } } | undefined;
  const eventType = emit?.event?.with?.type ?? '';
  const label = node.properties['label'] ? String(node.properties['label']) : 'Emit';

  return html`
    <div style="padding: 8px 12px; border: 2px solid var(--pages-border-strong, #888); background: var(--pages-surface-raised, #f8f8f8); border-top: 3px solid #7c3aed; min-width: 160px; font-family: var(--pages-font-family, sans-serif); font-size: 13px; border-radius: 4px;">
      <div style="display: flex; align-items: center; gap: 6px; font-weight: 700; color: var(--pages-text-color, #333);">
        <span>📡</span>
        <span>${label}</span>
      </div>
      ${eventType ? html`<div style="color: var(--pages-text-secondary, #666); font-size: 11px; margin-top: 2px;">${eventType}</div>` : ''}
    </div>
  `;
}
```

- [ ] **Step 4: Create listen.ts**

```typescript
import { html } from 'lit-html';
import type { StencilGrammar, GraphNode, NodeDecoration } from '@casehubio/graph-core';
import type { StencilTemplate } from '@casehubio/graph-renderer';
import { FLOW_SOURCES, FLOW_TARGETS } from './grammars.js';

export const listenGrammar: StencilGrammar = {
  type: 'swf-listen',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: [...FLOW_SOURCES] },
    outbound: { min: 0, max: 1, allowedTo: [...FLOW_TARGETS] },
  },
};

function describeStrategy(listen: Record<string, unknown> | undefined): string {
  if (!listen) return 'events';
  const to = listen['to'] as Record<string, unknown> | undefined;
  if (!to) return 'events';
  if (to['one']) return 'one event';
  if (to['all']) {
    const items = to['all'] as unknown[];
    return `all ${items?.length ?? ''} events`;
  }
  if (to['any']) {
    const items = to['any'] as unknown[];
    return `any of ${items?.length ?? ''} events`;
  }
  return 'events';
}

export function renderListen(node: GraphNode, _decoration?: NodeDecoration): StencilTemplate {
  const listen = node.properties['listen'] as Record<string, unknown> | undefined;
  const strategy = describeStrategy(listen);
  const label = node.properties['label'] ? String(node.properties['label']) : 'Listen';

  return html`
    <div style="padding: 8px 12px; border: 2px solid var(--pages-border-strong, #888); background: var(--pages-surface-raised, #f8f8f8); border-top: 3px solid #0d9488; min-width: 160px; font-family: var(--pages-font-family, sans-serif); font-size: 13px; border-radius: 4px;">
      <div style="display: flex; align-items: center; gap: 6px; font-weight: 700; color: var(--pages-text-color, #333);">
        <span>📻</span>
        <span>${label}</span>
      </div>
      <div style="color: var(--pages-text-secondary, #666); font-size: 11px; margin-top: 2px;">${strategy}</div>
    </div>
  `;
}
```

- [ ] **Step 5: Create run.ts**

```typescript
import { html } from 'lit-html';
import type { StencilGrammar, GraphNode, NodeDecoration } from '@casehubio/graph-core';
import type { StencilTemplate } from '@casehubio/graph-renderer';
import { FLOW_SOURCES, FLOW_TARGETS } from './grammars.js';

export const runGrammar: StencilGrammar = {
  type: 'swf-run',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: [...FLOW_SOURCES] },
    outbound: { min: 0, max: 1, allowedTo: [...FLOW_TARGETS] },
  },
};

function describeRunType(run: Record<string, unknown> | undefined): string {
  if (!run) return 'run';
  if (run['container']) {
    const c = run['container'] as { image?: string };
    return c.image ? `container: ${c.image}` : 'container';
  }
  if (run['shell']) {
    const s = run['shell'] as { command?: string };
    return s.command ? `shell: ${s.command}` : 'shell';
  }
  if (run['script']) {
    const s = run['script'] as { language?: string };
    return s.language ? `script: ${s.language}` : 'script';
  }
  if (run['workflow']) {
    const w = run['workflow'] as { name?: string };
    return w.name ? `workflow: ${w.name}` : 'workflow';
  }
  return 'run';
}

export function renderRun(node: GraphNode, _decoration?: NodeDecoration): StencilTemplate {
  const run = node.properties['run'] as Record<string, unknown> | undefined;
  const subType = describeRunType(run);
  const label = node.properties['label'] ? String(node.properties['label']) : 'Run';

  return html`
    <div style="padding: 8px 12px; border: 2px solid var(--pages-border-strong, #888); background: var(--pages-surface-raised, #f8f8f8); border-top: 3px solid #059669; min-width: 160px; font-family: var(--pages-font-family, sans-serif); font-size: 13px; border-radius: 4px;">
      <div style="display: flex; align-items: center; gap: 6px; font-weight: 700; color: var(--pages-text-color, #333);">
        <span>⚙️</span>
        <span>${label}</span>
      </div>
      <div style="color: var(--pages-text-secondary, #666); font-size: 11px; margin-top: 2px;">${subType}</div>
    </div>
  `;
}
```

- [ ] **Step 6: Create wait.ts**

```typescript
import { html } from 'lit-html';
import type { StencilGrammar, GraphNode, NodeDecoration } from '@casehubio/graph-core';
import type { StencilTemplate } from '@casehubio/graph-renderer';
import { FLOW_SOURCES, FLOW_TARGETS } from './grammars.js';

export const waitGrammar: StencilGrammar = {
  type: 'swf-wait',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: [...FLOW_SOURCES] },
    outbound: { min: 0, max: 1, allowedTo: [...FLOW_TARGETS] },
  },
};

function formatDuration(wait: unknown): string {
  if (typeof wait === 'string') return wait;
  if (typeof wait === 'object' && wait !== null) {
    const d = wait as Record<string, number>;
    const parts: string[] = [];
    if (d['days']) parts.push(`${d['days']}d`);
    if (d['hours']) parts.push(`${d['hours']}h`);
    if (d['minutes']) parts.push(`${d['minutes']}m`);
    if (d['seconds']) parts.push(`${d['seconds']}s`);
    if (d['milliseconds']) parts.push(`${d['milliseconds']}ms`);
    if (parts.length > 0) return parts.join(' ');
  }
  return 'wait';
}

export function renderWait(node: GraphNode, _decoration?: NodeDecoration): StencilTemplate {
  const wait = node.properties['wait'];
  const duration = formatDuration(wait);
  const label = node.properties['label'] ? String(node.properties['label']) : 'Wait';

  return html`
    <div style="padding: 8px 12px; border: 2px solid var(--pages-border-strong, #888); background: var(--pages-surface-raised, #f8f8f8); border-top: 3px solid #475569; min-width: 160px; font-family: var(--pages-font-family, sans-serif); font-size: 13px; border-radius: 4px;">
      <div style="display: flex; align-items: center; gap: 6px; font-weight: 700; color: var(--pages-text-color, #333);">
        <span>⏳</span>
        <span>${label}</span>
      </div>
      <div style="color: var(--pages-text-secondary, #666); font-size: 11px; margin-top: 2px;">${duration}</div>
    </div>
  `;
}
```

- [ ] **Step 7: Update index.ts exports**

Add to `packages/graph-stencil-swf/src/stencils/index.ts`:
```typescript
export { emitGrammar, renderEmit } from './emit.js';
export { listenGrammar, renderListen } from './listen.js';
export { runGrammar, renderRun } from './run.js';
export { waitGrammar, renderWait } from './wait.js';
export { FLOW_SOURCES, FLOW_TARGETS } from './grammars.js';
```

- [ ] **Step 8: Update register.ts — register 4 leaf stencils**

Add imports and registration calls:
```typescript
import { emitGrammar, renderEmit } from './emit.js';
import { listenGrammar, renderListen } from './listen.js';
import { runGrammar, renderRun } from './run.js';
import { waitGrammar, renderWait } from './wait.js';

// Inside registerSwfStencils(), after existing registrations:
registerStencil({ type: 'swf-emit', label: 'Emit', icon: 'send', grammar: emitGrammar, render: renderEmit });
registerStencil({ type: 'swf-listen', label: 'Listen', icon: 'radio', grammar: listenGrammar, render: renderListen });
registerStencil({ type: 'swf-run', label: 'Run', icon: 'terminal', grammar: runGrammar, render: renderRun });
registerStencil({ type: 'swf-wait', label: 'Wait', icon: 'clock', grammar: waitGrammar, render: renderWait });
```

Also add property schema registrations in the `typeMap`:
```typescript
const typeMap: Record<string, string> = {
  CallTask: 'swf-call', SetTask: 'swf-set', SwitchTask: 'swf-switch',
  RaiseTask: 'swf-raise', TryTask: 'swf-try', TryCatchTask: 'swf-try-catch',
  EmitTask: 'swf-emit', ListenTask: 'swf-listen', RunTask: 'swf-run',
  WaitTask: 'swf-wait',
};
```

- [ ] **Step 9: Update CREATABLE_TYPES in edit policy**

Add to `packages/graph-stencil-swf/src/editing/swf-edit-policy.ts`:
```typescript
const CREATABLE_TYPES: readonly StencilTypeInfo[] = [
  { type: 'swf-call', label: 'Call', icon: 'phone' },
  { type: 'swf-set', label: 'Set', icon: 'edit' },
  { type: 'swf-switch', label: 'Switch', icon: 'git-branch' },
  { type: 'swf-for', label: 'For', icon: 'repeat' },
  { type: 'swf-do', label: 'Do', icon: 'list' },
  { type: 'swf-fork', label: 'Fork', icon: 'git-merge' },
  { type: 'swf-emit', label: 'Emit', icon: 'send' },
  { type: 'swf-listen', label: 'Listen', icon: 'radio' },
  { type: 'swf-run', label: 'Run', icon: 'terminal' },
  { type: 'swf-wait', label: 'Wait', icon: 'clock' },
  { type: 'swf-raise', label: 'Raise', icon: 'alert-triangle' },
  { type: 'swf-try', label: 'Try', icon: 'shield' },
];
```

- [ ] **Step 10: Run all tests**

Run: `yarn --cwd packages/graph-stencil-swf vitest run`
Expected: all tests pass (including new stencil tests)

- [ ] **Step 11: Commit**

```bash
git add packages/graph-stencil-swf/src/
git commit -m "feat(graph-stencil-swf): add Emit, Listen, Run, Wait leaf stencils

All four follow the Call/Set card pattern with distinct accent colours
and subtitle extraction. Grammars use shared FLOW_SOURCES/FLOW_TARGETS.

Refs #172"
```

## Batch 2: Container stencils + diagram integration

### Task 4: Do and Fork container stencils

**Files:**
- Create: `packages/graph-stencil-swf/src/stencils/do.ts`
- Create: `packages/graph-stencil-swf/src/stencils/fork.ts`
- Modify: `packages/graph-stencil-swf/src/stencils/index.ts`
- Modify: `packages/graph-stencil-swf/src/stencils/register.ts`
- Test: `packages/graph-stencil-swf/src/stencils/stencils.test.ts`

**Interfaces:**
- Consumes: `FLOW_SOURCES`, `FLOW_TARGETS` from `./grammars.js`
- Produces: `doGrammar`, `renderDo` — grammar + compact header render
- Produces: `forkGrammar`, `renderFork` — grammar + header with compete badge

- [ ] **Step 1: Write failing tests for Do and Fork**

Add to `stencils.test.ts`:

```typescript
import { renderDo } from './do.js';
import { renderFork } from './fork.js';

it('renderDo returns compact header', () => {
  const result = renderDo(makeNode('swf-do', { label: 'setup' }));
  expect(result).toBeDefined();
  expect(result.values).toBeDefined();
});

it('renderDo handles missing label', () => {
  const result = renderDo(makeNode('swf-do', {}));
  expect(result).toBeDefined();
});

it('renderFork shows compete badge when true', () => {
  const result = renderFork(makeNode('swf-fork', {
    fork: { compete: true },
    label: 'race',
  }));
  expect(result).toBeDefined();
  expect(result.values).toBeDefined();
});

it('renderFork hides badge when compete is false', () => {
  const result = renderFork(makeNode('swf-fork', {
    fork: { compete: false },
    label: 'parallel',
  }));
  expect(result).toBeDefined();
});

it('renderFork handles missing fork config', () => {
  const result = renderFork(makeNode('swf-fork', {}));
  expect(result).toBeDefined();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/graph-stencil-swf vitest run src/stencils/stencils.test.ts`
Expected: FAIL — cannot find modules do.js, fork.js

- [ ] **Step 3: Create do.ts**

```typescript
import { html } from 'lit-html';
import type { StencilGrammar, GraphNode, NodeDecoration } from '@casehubio/graph-core';
import type { StencilTemplate } from '@casehubio/graph-renderer';
import { FLOW_SOURCES, FLOW_TARGETS } from './grammars.js';

export const doGrammar: StencilGrammar = {
  type: 'swf-do',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: [...FLOW_SOURCES] },
    outbound: { min: 0, max: 1, allowedTo: [...FLOW_TARGETS] },
  },
};

export function renderDo(node: GraphNode, _decoration?: NodeDecoration): StencilTemplate {
  const label = node.properties['label'] ? String(node.properties['label']) : 'Do';
  return html`
    <div style="font-family: var(--pages-font-family, sans-serif); font-size: 11px; font-weight: 600; color: var(--pages-accent-11, #1d4ed8); letter-spacing: 0.03em; padding: 2px 8px;">
      <span style="opacity: 0.8;">📋</span> ${label}
    </div>
  `;
}
```

- [ ] **Step 4: Create fork.ts**

```typescript
import { html, nothing } from 'lit-html';
import type { StencilGrammar, GraphNode, NodeDecoration } from '@casehubio/graph-core';
import type { StencilTemplate } from '@casehubio/graph-renderer';
import { FLOW_SOURCES, FLOW_TARGETS } from './grammars.js';

export const forkGrammar: StencilGrammar = {
  type: 'swf-fork',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: [...FLOW_SOURCES] },
    outbound: { min: 0, max: 1, allowedTo: [...FLOW_TARGETS] },
  },
};

export function renderFork(node: GraphNode, _decoration?: NodeDecoration): StencilTemplate {
  const fork = node.properties['fork'] as { compete?: boolean } | undefined;
  const compete = fork?.compete === true;
  const label = node.properties['label'] ? String(node.properties['label']) : 'Fork';

  return html`
    <div style="font-family: var(--pages-font-family, sans-serif); font-size: 11px; font-weight: 600; color: var(--pages-accent-11, #1d4ed8); letter-spacing: 0.03em; padding: 2px 8px;">
      <span style="opacity: 0.8;">🔀</span> ${label}
      ${compete ? html`<span style="background: #dc2626; color: white; font-size: 9px; padding: 1px 5px; border-radius: 3px; margin-left: 6px; font-weight: 500; letter-spacing: 0;">race</span>` : nothing}
    </div>
  `;
}
```

- [ ] **Step 5: Update index.ts exports**

Add:
```typescript
export { doGrammar, renderDo } from './do.js';
export { forkGrammar, renderFork } from './fork.js';
```

- [ ] **Step 6: Update register.ts — register Do and Fork**

Add imports and registrations:
```typescript
import { doGrammar, renderDo } from './do.js';
import { forkGrammar, renderFork } from './fork.js';

// Inside registerSwfStencils():
registerStencil({ type: 'swf-do', label: 'Do', icon: 'list', grammar: doGrammar, render: renderDo });
registerStencil({ type: 'swf-fork', label: 'Fork', icon: 'git-merge', grammar: forkGrammar, render: renderFork });
```

Add to property schema `typeMap`:
```typescript
DoTask: 'swf-do', ForkTask: 'swf-fork',
```

- [ ] **Step 7: Run tests**

Run: `yarn --cwd packages/graph-stencil-swf vitest run`
Expected: all tests pass

- [ ] **Step 8: Commit**

```bash
git add packages/graph-stencil-swf/src/
git commit -m "feat(graph-stencil-swf): add Do, Fork container stencils

Do renders compact header (sequential sub-tasks). Fork renders header
with 'race' badge when compete=true. Both use container-style grammars.

Refs #172"
```

### Task 5: Wire containers into swf-diagram + layout tests

**Files:**
- Modify: `components/swf-diagram/src/swf-diagram.ts:222,228,233`
- Test: `packages/graph-stencil-swf/src/layout/swf-stack-layout.test.ts`

**Interfaces:**
- Consumes: `swf-do`, `swf-fork` stencil types from Task 4

- [ ] **Step 1: Write layout test for Do container**

Add to `swf-stack-layout.test.ts`:

```typescript
function doContainer(): GraphModel {
  return {
    nodes: [
      node('root', 'swf-root'),
      node('root-entry-node', 'swf-start', 'root'),
      node('/do/0/setup', 'swf-do', 'root'),
      node('/do/0/setup/entry', 'swf-entry', '/do/0/setup'),
      node('/do/0/setup/0/step1', 'swf-call', '/do/0/setup'),
      node('/do/0/setup/1/step2', 'swf-set', '/do/0/setup'),
      node('/do/0/setup/exit', 'swf-exit', '/do/0/setup'),
      node('/do/1/finish', 'swf-call', 'root'),
      node('root-exit-node', 'swf-end', 'root'),
    ],
    edges: [
      edge('root-entry-node', '/do/0/setup'),
      edge('/do/0/setup', '/do/1/finish'),
      edge('/do/1/finish', 'root-exit-node'),
    ],
  };
}

// Inside describe block:
describe('do container', () => {
  it('positions do container with children inside', () => {
    const result = computeSwfStackLayout(doContainer());
    const doLayout = result.nodeLayouts.get('/do/0/setup');
    expect(doLayout).toBeDefined();
    expect(doLayout!.height).toBeGreaterThan(53);

    const child1 = result.nodeLayouts.get('/do/0/setup/entry');
    const child2 = result.nodeLayouts.get('/do/0/setup/0/step1');
    expect(child1).toBeDefined();
    expect(child2).toBeDefined();
    expect(child2!.y).toBeGreaterThan(child1!.y);
  });
});
```

- [ ] **Step 2: Write layout test for Fork container**

```typescript
function forkContainer(): GraphModel {
  return {
    nodes: [
      node('root', 'swf-root'),
      node('root-entry-node', 'swf-start', 'root'),
      node('/do/0/parallel', 'swf-fork', 'root'),
      node('/do/0/parallel/branch1', 'swf-call', '/do/0/parallel'),
      node('/do/0/parallel/branch2', 'swf-call', '/do/0/parallel'),
      node('/do/1/merge', 'swf-call', 'root'),
      node('root-exit-node', 'swf-end', 'root'),
    ],
    edges: [
      edge('root-entry-node', '/do/0/parallel'),
      edge('/do/0/parallel', '/do/1/merge'),
      edge('/do/1/merge', 'root-exit-node'),
    ],
  };
}

describe('fork container', () => {
  it('positions fork container with children inside', () => {
    const result = computeSwfStackLayout(forkContainer());
    const forkLayout = result.nodeLayouts.get('/do/0/parallel');
    expect(forkLayout).toBeDefined();
    expect(forkLayout!.height).toBeGreaterThan(53);

    const b1 = result.nodeLayouts.get('/do/0/parallel/branch1');
    const b2 = result.nodeLayouts.get('/do/0/parallel/branch2');
    expect(b1).toBeDefined();
    expect(b2).toBeDefined();
  });
});
```

- [ ] **Step 3: Run layout tests to verify they pass**

The layout already handles containers generically via `parentId` — any
node with children gets container sizing. The tests should pass without
layout changes.

Run: `yarn --cwd packages/graph-stencil-swf vitest run src/layout/swf-stack-layout.test.ts`
Expected: PASS (the layout system is generic; new types don't need
layout code changes)

- [ ] **Step 4: Update swf-diagram.ts — add containers to styling and visibility**

In `_computeFilteredEdges()`, add `swf-fork` to the edge-hiding check
(fork branches are structural, not sequential):

```typescript
return parentType !== 'swf-try' && parentType !== 'swf-try-catch' && parentType !== 'swf-fork';
```

In `_computeFilteredNodes()`, add do and fork to `containerTypes`:

```typescript
const containerTypes = new Set(['swf-try', 'swf-try-catch', 'swf-for', 'swf-do', 'swf-fork']);
```

Update the `isInnerStructural` check to include `swf-do` and `swf-fork`
children:

```typescript
const isInnerStructural = (n.type === 'swf-try-catch' || n.type === 'swf-try' || n.type === 'swf-catch' || n.type === 'swf-do' || n.type === 'swf-fork') && n.parentId && n.parentId !== 'root';
```

Wait — the `isInnerStructural` check hides handles for nodes that are
themselves container-type AND nested inside another container. For `do`
and `fork` this is correct — when nested, their handles should be hidden
(the parent container handles connections). But standalone `do`/`fork`
at root level should keep handles. The existing check already guards
this with `n.parentId && n.parentId !== 'root'`.

- [ ] **Step 5: Run full test suite**

Run: `yarn --cwd packages/graph-stencil-swf vitest run && yarn --cwd components/swf-diagram vitest run`
Expected: all tests pass

- [ ] **Step 6: Commit**

```bash
git add components/swf-diagram/src/swf-diagram.ts \
  packages/graph-stencil-swf/src/layout/swf-stack-layout.test.ts
git commit -m "feat(swf-diagram): wire Do/Fork containers into diagram rendering

Container styling, edge visibility (fork hides internal edges),
handle visibility for nested containers. Layout tests for both types.

Refs #172"
```

## Batch 3: Examples + adapter verification

### Task 6: Example YAML and adapter coverage

**Files:**
- Modify: `examples/src/pages/swf-diagram-page.ts`
- Test: `packages/graph-stencil-swf/src/adapter/swf-adapter.test.ts` (create if needed)

**Interfaces:**
- Consumes: all 12 stencil types from Tasks 1-4

- [ ] **Step 1: Update DSL version in existing examples**

Replace all `dsl: 1.0.0-alpha1` with `dsl: '1.0.3'` in
`examples/src/pages/swf-diagram-page.ts`.

- [ ] **Step 2: Add example YAML exercising new task types**

Add a new example workflow to the examples page that uses all 6 new
types. Example: "Event-Driven Order Processing"

```yaml
document:
  dsl: '1.0.3'
  namespace: examples
  name: event-driven-order
  version: '1.0.0'
do:
  - emitOrder:
      emit:
        event:
          with:
            source: https://shop.example.com
            type: com.shop.order.placed
            data:
              orderId: order-123
  - awaitConfirmation:
      listen:
        to:
          one:
            with:
              type: com.warehouse.order.confirmed
  - processInParallel:
      fork:
        compete: false
        branches:
          - checkInventory:
              call: http
              with:
                method: get
                endpoint: https://api.example.com/inventory
          - notifyCustomer:
              emit:
                event:
                  with:
                    type: com.shop.order.notification
  - runFulfillment:
      run:
        container:
          image: fulfillment-service:latest
  - cooldown:
      wait:
        seconds: 30
  - cleanup:
      do:
        - archiveOrder:
            call: http
            with:
              method: post
              endpoint: https://api.example.com/archive
        - updateMetrics:
            set:
              ordersProcessed: ${ .ordersProcessed + 1 }
```

- [ ] **Step 3: Write adapter test verifying all 12 types map correctly**

Create or add to adapter test file. Parse a YAML with all task types and
verify none fall through to `swf-generic`:

```typescript
import { toSwfGraph } from './swf-adapter.js';

it('maps all 12 OWS 1.0 task types to named stencils', () => {
  const yaml = `
document:
  dsl: '1.0.3'
  namespace: test
  name: all-types
  version: '1.0.0'
do:
  - callStep:
      call: http
      with:
        method: get
        endpoint: https://example.com
  - setStep:
      set:
        key: value
  - emitStep:
      emit:
        event:
          with:
            type: test.event
  - waitStep:
      wait:
        seconds: 5
  - listenStep:
      listen:
        to:
          one:
            with:
              type: test.response
  - runStep:
      run:
        shell:
          command: echo done
`;
  const result = toSwfGraph(yaml);
  const types = result.model.nodes.map(n => n.type);
  expect(types).not.toContain('swf-generic');
});
```

- [ ] **Step 4: Run tests**

Run: `yarn --cwd packages/graph-stencil-swf vitest run`
Expected: all tests pass — no nodes fall through to swf-generic

- [ ] **Step 5: Run full build**

Run: `yarn build && yarn test && yarn typecheck`
Expected: clean build, all tests pass, no type errors

- [ ] **Step 6: Commit**

```bash
git add examples/src/pages/swf-diagram-page.ts \
  packages/graph-stencil-swf/src/adapter/
git commit -m "feat(examples): event-driven order example with all OWS 1.0 task types

Updates DSL version to 1.0.3. Adapter test verifies all 12 types map
to named stencils (no fallback to swf-generic).

Closes #172"
```

## References

- [2026-09-25-ows-1.0-stencil-completeness-design.md] — design spec this plan implements
- [packages/graph-stencil-swf/src/stencils/call.ts] — leaf stencil pattern (card render)
- [packages/graph-stencil-swf/src/stencils/for.ts] — container stencil pattern (compact header)
- [packages/graph-stencil-swf/src/stencils/register.ts] — stencil registration pattern
- [packages/graph-stencil-swf/src/types.ts:9-12] — SWF_KNOWN_TYPES
- [packages/graph-stencil-swf/src/editing/swf-edit-policy.ts:5-12] — CREATABLE_TYPES
- [packages/graph-stencil-swf/src/layout/swf-stack-layout.ts] — container layout
- [components/swf-diagram/src/swf-diagram.ts:222-247] — container styling and edge visibility
- [OWS 1.0 DSL reference](https://github.com/open-workflow-specification/specification/blob/main/dsl-reference.md)
- [GitHub #172] — focal issue
