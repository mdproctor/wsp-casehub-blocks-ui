# Design Journal — issue-166-agent-setup-wizard

## 2026-09-21 — First implementation session: Batches 1-3 + Task 8

Executed the implementation plan through 8 of 13 tasks. All foundation
types, avatar package, manifest editor, and catalog are built and tested.

**Completed:**
- Batch 1 (Foundation): Manifest types, FullAgentDescriptor/AgentDisposition,
  event topics, relationship registry — all in blocks-ui-core
- Batch 2 (Avatar-2D): canonical term registry mapping disposition axes
  to DiceBear Avataaars features, generator with deterministic/candidate/
  customise modes, <agent-avatar> web component
- Batch 3 (Manifest-Editor): provider card grid with detection badges,
  preset system (4 built-in), expanded section with credential type
  switcher, tier-grouped model list, alias editor, test connection button
- Batch 4 partial (Catalog): 10 template descriptors, filterTemplates()
  pure function, <agent-catalog> with popular section, pill filters,
  search, card grid

**Setup notes:**
- Slot clone needed .casehub-packages symlinked from main repo
- blocks-ui-core must be built before downstream packages can test
- IntelliJ MCP sessions for the slot are transient — may need re-opening

**Remaining (5 tasks):**
- Task 9: From-scratch wizard (6-step flow)
- Task 10: Agent profile character sheet
- Task 11: Relationship table editing
- Task 12: Arc diagram + ego subgraph
- Task 13: Integration and workspace registration

