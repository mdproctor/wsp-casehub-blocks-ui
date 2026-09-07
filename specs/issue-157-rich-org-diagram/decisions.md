## D1: Agent descriptor data flow

**Choice:** Merge into GraphNode.properties during adaptation — extend `toOrgGraph` to accept both YAML and the agents map: `toOrgGraph(yaml, agents?)`. The adapter looks up each member's descriptor and merges slot, disposition, capabilities into the node's properties.
**Alternatives:**
- Pass via NodeDecoration — violates node-decoration-runtime-boundary protocol; descriptors are static identity, not runtime overlays.
- Renderer context/lookup — requires changing the stencil render signature across graph-renderer infrastructure for one consumer's need.
**Rationale:** The adapter already transforms domain data into graph model properties. Descriptor data is static definition data (same as agentId, role). The protocol explicitly says properties carry "definition data." No infrastructure changes needed.
**Trade-offs:** GraphNode.properties grows larger per agent node. Acceptable — the data is needed for rendering and is bounded per agent.
**Sources:** node-decoration-runtime-boundary protocol, org-adapter.ts, issue #157 comment (OrgDiagramData shape)
**Exploration:** quick
**Status:** captured
