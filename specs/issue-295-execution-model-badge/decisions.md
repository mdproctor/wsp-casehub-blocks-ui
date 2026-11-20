# Decisions — #295 Execution Model Badge

## D1: Type set is open, not a closed enum

**Choice:** The badge shows whatever document type is extracted from the `name.type.yaml` filename convention. Known types get specific colours; unknown types get a neutral default.
**Alternatives:**
- Closed enum of execution models — simpler but requires code changes for every new type
**Rationale:** The platform has an expanding set of YAML document types (choreography, htn, goap, llm, swf, playbook, agent-conf, avatar-conf, agency, etc.). A closed enum would require constant maintenance.
**Trade-offs:** No compile-time exhaustiveness check on the type — mitigated by the default colour fallback.
**Sources:** User clarification during brainstorming — avatar-conf, agency, agent-conf are all valid types.
**Exploration:** quick
**Status:** captured

## D2: Detection strategy — filename primary, content fallback

**Choice:** Extract type from `name.type.yaml` filename pattern (primary). Fall back to content-based `detectDiagramType()` for files without the convention.
**Alternatives:**
- Content-only detection — requires heuristic code for every type, fragile
- Filename-only — breaks for inline YAML and legacy files
**Rationale:** The `name.type.yaml` convention is the intended standard. Content detection already exists as a partial fallback.
**Trade-offs:** Filename detection fails silently for files not following the convention — the fallback covers the gap.
**Sources:** User clarification — `name.type.yaml` is the decided convention, may not be consistent everywhere yet.
**Exploration:** quick
**Status:** captured

## D3: Badge placement — toolbar left side

**Choice:** Colour-coded pill in the toolbar, left side after save/export buttons.
**Alternatives:**
- Toolbar right side next to case status — groups status info but gets crowded
- Dedicated identity strip below toolbar — more prominent but adds UI weight
**Rationale:** Always visible, low visual weight, doesn't interfere with the runtime toggle area.
**Trade-offs:** Badge is small — might be missed on first glance. Acceptable for a metadata indicator.
**Sources:** casehub-diagram-toolbar.ts existing layout
**Exploration:** quick
**Status:** captured

## D4: Drop 'sequential' and 'portfolio' from execution model types

**Choice:** These are topology/composition properties of choreography, not distinct execution models.
**Alternatives:**
- Keep them as separate types — more granular but conceptually wrong
**Rationale:** A sequential case is a choreography with linear topology. A portfolio case is a choreography that dispatches to sub-cases. Neither has different runtime semantics.
**Trade-offs:** Less granular badging — but the badge shows document type, not topology.
**Sources:** User challenge during brainstorming
**Exploration:** quick
**Status:** captured
