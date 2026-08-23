## D1: Refactoring strategy — REVISED after decision review

**Choice:** Shared renderers via composition — both components import pure render functions from pages-viz
**Previous choice:** Extend PagesEventTimeline — invalidated by R1-02 (PagesElement render gate incompatible with standalone self-fetch)
**Alternatives:**
- Extend PagesEventTimeline — PagesElement.render() gates on `props && !loading && dataSet`, requiring render() override that negates inheritance value (~125 lines of bridge code)
- Type alignment only — trivial but zero code reduction
**Rationale:** PagesElement's lifecycle (host-pushed data via `pages-data-request` + `DataSourceController`) and DataSourceMixin's lifecycle (self-fetch via `endpoint` + `fetchSource`) are fundamentally different data acquisition models. They are correctly independent — not an "evolution risk" but the right expression of two distinct usage patterns. Shared renderers (pure functions: nodes[] + callbacks → HTML) eliminate the actual duplication (585 lines of rendering code) while keeping each component's data lifecycle correct. BlocksTimeline shrinks from ~955 to ~280 lines (71% reduction).
**Trade-offs:** Two component classes persist — but they serve different contexts (dashboard-embedded vs standalone/panel-hosted) and the remaining ~100 lines in each IS the value each provides.
**Sources:** PagesElement.ts (render gate at line 93-98), PagesEventTimeline.ts, blocks-timeline.ts, decision review R1-02/R1-03/R1-04
**Exploration:** quick → revised after light decision review
**Status:** revised

## D2: Renderer integration pattern — UNCHANGED

**Choice:** Standalone pure render functions exported from pages-viz
**Alternatives:**
- Inline private methods on PagesEventTimeline — not independently testable
**Rationale:** Renderers are pure functions (nodes + option bags → HTML). Most composable unit after types. Independently testable. Both PagesEventTimeline and BlocksTimeline import and call them with their own callbacks. Blocks-timeline code transfers almost verbatim.
**Trade-offs:** Minor file proliferation in pages-viz.
**Sources:** blocks-timeline/src/renderers/vertical.ts, horizontal.ts, compact.ts
**Exploration:** quick
**Status:** captured

## D3: Data pipeline — REVISED (no bridge needed)

**Choice:** BlocksTimeline keeps DataSourceMixin — no bridge needed
**Previous choice:** Thin fetch bridge on BlocksTimeline — invalidated because no inheritance means no lifecycle mismatch to bridge
**Rationale:** Without inheritance, there is no PagesElement lifecycle to bridge. BlocksTimeline keeps DataSourceMixin(LiveRegionMixin(LitElement)) as its base. The data lifecycle code (endpoint, fetchSource, configure, headers) remains unchanged. Consumers keep the same API.
**Sources:** Decision review R1-05
**Exploration:** quick → revised after light decision review
**Depends on:** D1 (composition, not inheritance)
**Status:** revised

## D4: Pagination location — REVISED (stays in blocks)

**Choice:** Pagination stays in BlocksTimeline
**Previous choice:** Push to PagesEventTimeline — invalidated by R1-06 (self-fetch pagination conflicts with PagesElement's host-driven data model where DataSourceController.activePage handles pagination)
**Rationale:** Pagination is a self-fetch concern. PagesEventTimeline's host context handles pagination via DataSourceController.activePage — the host re-pushes data for the requested page. BlocksTimeline's load-more pattern (fetch next page, append to nodes) is specific to standalone self-fetch. Pushing it to pages would create two incompatible pagination paths in one component.
**Trade-offs:** Pagination code (~50 lines) stays in blocks-timeline. This is correct — it's part of the self-fetch lifecycle.
**Sources:** Decision review R1-06, PagesElement.ts (activePage), blocks-timeline.ts (_fetchPage, _loadMore)
**Exploration:** quick → revised after light decision review
**Depends on:** D1 (composition, not inheritance)
**Status:** revised

## D5: Type alignment — REVISED (extend, don't rename)

**Choice:** Adopt pages types (EventTimelineNode, EventTimelineStrategy) as base; extend with BlocksTimelineStrategy for pagination fields. Re-export old names temporarily, file issue to track removal.
**Previous choice:** Simple rename — missed structural differences (readonly, unknown vs TemplateResult, pagination fields)
**Alternatives:**
- Permanent aliases — two names for same type, drift risk
- Keep blocks types independent — no alignment, drift inevitable
**Rationale:** EventTimelineNode and TimelineNode are structurally identical except `readonly` modifiers (pages uses readonly, blocks doesn't). Adopting `readonly` is correct for immutable data objects. Render callback return type `unknown` (pages) vs `TemplateResult` (blocks) is resolved by the type system — TemplateResult is assignable to unknown, so blocks strategies work without change. Pagination fields belong on an extended interface:
```typescript
interface BlocksTimelineStrategy<T = unknown> extends EventTimelineStrategy<T> {
  supportsPagination?: boolean;
  extractPaginationMeta?: (raw: unknown) => PaginationMeta | undefined;
}
```
Temporary re-exports tracked by a follow-up issue to avoid calcification.
**Sources:** Decision review R1-09, event-timeline-types.ts, blocks-timeline types.ts, protocol PP-20260713-8ea1af
**Exploration:** quick → revised after light decision review
**Depends on:** D1 (composition approach)
**Status:** revised

## D6: Strategy CSS class coupling

**Choice:** Strategies use inline styles for render callback output per protocol PP-20260713-8ea1af
**Rationale:** When renderers move to pages-viz, strategy renderNode output (in blocks-ui) would reference CSS classes owned by pages-viz — cross-layer coupling with no typed contract. Protocol PP-20260713-8ea1af already requires "all render callback output must use inline styles to be self-contained." Current eventChronologyStrategy.renderNode produces `<span class="event-type-badge lifecycle">` which violates this protocol. Fix as part of this refactoring.
**Trade-offs:** Slightly more verbose strategy render callbacks. Correct — inline styles are the protocol.
**Sources:** Decision review R1-10, protocol PP-20260713-8ea1af, eventChronologyStrategy renderNode
**Exploration:** quick
**Status:** captured

## D7: Event topic naming

**Choice:** Align BlocksTimeline to colon-delimited convention during this refactoring: `timeline:node-selected`, `timeline:expand-requested`
**Rationale:** ARC42STORIES.MD §4 documents colon-delimited topic hierarchies. BlocksTimeline uses dots (`timeline.node-selected`), PagesEventTimeline uses colons (`event-timeline:node-selected`). Since renderers call callbacks (not emit events directly), each component emits its own topics — no coupling. But blocks should align to the documented convention. This refactoring is a natural breakpoint.
**Trade-offs:** Consumer event listeners need topic string updates. Mechanical.
**Sources:** Decision review R1-11, ARC42STORIES.MD §4
**Exploration:** quick
**Status:** captured

## D8: Strategy binding — no change needed

**Choice:** Both binding patterns coexist — PagesEventTimeline uses registry (strategyKey), BlocksTimeline uses property injection
**Rationale:** Registry serves YAML-driven dashboards (strategy instances can't be passed as HTML attributes). Property injection serves programmatic composition (type-safe). These are orthogonal to the shared rendering layer. Neither needs to change.
**Sources:** Decision review R1-12
**Exploration:** quick
**Status:** captured
