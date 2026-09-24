# HANDOFF — casehub-blocks-ui

## Last Session

Diagram stabilisation and pages alignment. Fixed hold-to-drag, splice
indicators, palette placement. Started custom SWF stack layout to
replace ELK — needs TDD completion.

### Pages Alignment
- Rebuilt pages SNAPSHOT on `issue-433-structural-editing` branch
- org-diagram: removed 49 lines of duplicate picker code, delegates to DiagramBaseMixin
- SWF edit policy: `getAddPlacement` returns `splitEdge` for tail insertion
- `_handlePaletteSelect` override checks `getAddPlacement` before mutation

### Hold-to-Drag (pages fixes on `issue-464-orchestration-showcase`)
- Gesture coordinator blocks `mousedown` alongside `pointerdown` (d3-zoom saw unblocked mousedown)
- `elementsFromPoint` uses `containerEl.getRootNode()` for shadow DOM compatibility
- Cursor: `grabbing` on all elements during `node-move-active`
- Splice indicator: edge identity check prevents clear+reapply flicker

### SWF Stack Layout (IN PROGRESS — needs TDD)
- `computeSwfStackLayout` in `graph-stencil-swf/src/layout/`
- Replaces ELK for SWF — instant, no async WASM
- 12 tests passing: linear pipeline, switch branching, bottom-aligned unequal branches, try/catch containers
- **BROKEN: column model not correct.** The layout assigns rows and detects forks but does not properly track columns through the full graph. Pre-fork nodes should center across the child columns. Each branch maintains its own column until convergence. The user described the model as "nested stacks" — everything is containers within containers, bottom-aligned at each level.
- Filed as #169

### Key Design Insight (from user)
The SWF layout is nested stacks:
- `do` array = vertical stack
- `switch` = horizontal fan-out into columns
- Each column is its own vertical stack
- `try/catch` = container with inner stacks
- Single-column nodes center across the child columns
- Bottom-aligned: shorter branches align to the bottom of the longest
- After splice (add/remove), recalculate from the mutation point downward

### Issues Addressed
| # | Title | Status |
|---|-------|--------|
| #165 | Showcase gallery SWF diagram not rendering | Fixed — stale `.casehub-packages` |
| #163 | SWF palette adds connected instead of standalone | Fixed — `getAddPlacement` |
| #164 | Canvas pans during hold-to-drag | Fixed — mousedown blocking in pages |
| #169 | ELK compound node spacing / custom layout | In progress — stack layout started |

## Immediate Next Step

TDD the SWF stack layout properly. The user's model is clear:
1. Write test cases for the nested-stacks column model
2. The layout is recursive: each `do` block is a VStack, each `switch` fans into HStack of VStacks
3. Container sizing flows bottom-up (children determine parent size)
4. Position assignment flows top-down (parent determines child origin)
5. Single-column segments center across the width of their child columns

Do NOT use ELK. Do NOT post-process. Get the recursive stack model right.

## Cross-Module

Pages changes (on `issue-464-orchestration-showcase`):
- `node-gesture-coordinator.ts` — mousedown blocking
- `node-move-coordinator.ts` — shadow DOM `elementsFromPoint`, flicker fix
- `css-isolation.ts` — cursor override during move mode

## References

- Stack layout: `packages/graph-stencil-swf/src/layout/swf-stack-layout.ts`
- Stack layout tests: `packages/graph-stencil-swf/src/layout/swf-stack-layout.test.ts`
- Spec: `specs/issue-158-lsp-schema-refinements/2026-09-17-domain-schema-assembly-design.md`
- Constraint: `packages/blocks-ui-core/src/layout-constraints.ts`
