# Decisions — #242 Agent Project Drill-Down

## D1: Third tab content

**Choice:** Steps tab — runtime execution steps with step-cards and control flow
**Alternatives:**
- Manifest tab — LLM config details only, no runtime steps
- Both as separate tabs — four tabs total, more UI complexity
**Rationale:** Matches the mockup (section 15). LLM config (model, provider) already visible in eidos identity header badges. Runtime steps show the agent's actual execution flow, which is the drill-down's purpose.
**Trade-offs:** LLM-specific config (temperature, tools, output schema) not shown; could be added later if needed.
**Sources:** step-review-viewer.html section 15, agent-viewer.ts, issue #242
**Exploration:** quick
**Status:** captured

## D2: Step data source

**Choice:** Lane steps from pipeline — pass the agent's lane steps from CoordinationPipeline.phases[parallel].lanes[agentName]
**Alternatives:**
- Separate step data on AgentViewerData — more flexible but requires consumer to extract and pass
**Rationale:** Data already exists in the pipeline. The drill-down event carries the steps. No new data types needed.
**Trade-offs:** Coupled to CoordinationPipeline structure — won't work for agents shown outside a pipeline context (but those don't need a Steps tab).
**Sources:** coordination-types.ts (SwimLanes, CoordinationStep), agent-viewer.ts (AgentViewerData)
**Exploration:** quick
**Status:** captured

## D3: Drill-down event model

**Choice:** CustomEvent bubble — emit 'coordination:drill-down' with detail containing agent name, steps, and descriptor reference
**Alternatives:**
- pages-event (emitPagesEvent) — more structured but adds unnecessary pages dependency
**Rationale:** Standard DOM pattern. Consumer catches with addEventListener. Consistent with existing blocks-ui event patterns (e.g., catalog:template:selected).
**Trade-offs:** No topic-based routing like pages-event provides, but drill-down is a direct parent-child interaction.
**Sources:** coordination-pipeline-view.ts (existing drillDown flag), blocks-ui event patterns
**Exploration:** quick
**Status:** captured
