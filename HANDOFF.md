# HANDOFF — casehub-blocks-ui

## Last Session

Completed #240 (coordination-pipeline-view) — first component from epic #239 gallery showcase. Built `coordination-pipeline-view` Lit component rendering coordination playbooks as vertical pipelines. Iterated on YAML ergonomics: flattened `SwimLane` to `Record<name, steps>` and removed `step:`/`barrier:` wrapper nesting from `PhaseItem` union after user feedback. Populated epic #239 queue (#241-#251) in `.plan`.

## Immediate Next Step

Start #241 — Agent Pipeline Graph View. Same KYC data rendered as node-edge topology. Reuses `CoordinationPipeline` types from #240. Will need a layout approach (ELK or manual SVG positioning) different from the CSS grid used for swim lanes.

## References

- `packages/agent-viewer/src/coordination-pipeline-view.ts` — the component
- `packages/agent-viewer/src/coordination-types.ts` — shared types (#241 reuses these)
- `packages/agent-viewer/gallery.html` — section 5 (rendered) + YAML tab
- `casehub-pages/docs/mockups/step-review-viewer.html` — section 14 is #241's mockup
