# HANDOFF — casehub-blocks-ui

## Last Session

**Issue:** casehubio/blocks#191 — Speaker diarization & identification (merged with #211)
**Queue position:** 8/10 (issue #191 active)
**Branch:** `issue-190-speech-denoising` (both blocks and blocks-ui repos)

### What Was Done

**Design phase completed:**
- Brainstormed with 9 decisions, standard decision review (3 rounds, 13 issues), user override on scope (reverted reviewer's un-merge of #191/#211)
- Wrote full design spec at `specs/issue-191-speaker-diarization/2026-09-03-speaker-diarization-design.md`
- Standard spec review (3 rounds, 27 issues, 22 verified)
- Implementation plan at `plans/2026-09-03-speaker-diarization.md` — 3 batches, 5 tasks

**Batch 1 complete (Embedding + Registry):**

Task 1 — SPI types + campplus embedding extractor:
- 8 SPI files in `speech-api`: `SpeakerEmbedding`, `SpeakerMatch`, `DiarizedSegment`, `DiarizationOptions`, `SpeakerEmbeddingExtractor`, `SpeakerRegistry`, `VoiceprintStore`, `SpeakerDiarizationService`
- `CampplusSpeakerEmbeddingExtractor` in `speech-sherpa` — ORT with campplus.onnx, 192-dim embeddings, replicates CosyVoice3VoiceEncoder preprocessing path
- `Provisioner.ensureCampplusModel()` — standalone campplus provisioning from HuggingFace
- 3 integration tests passing

Task 2 — voiceprint registry + file persistence:
- `CosineDistanceSpeakerRegistry` — ConcurrentHashMap cache, pluggable VoiceprintStore
- `FileVoiceprintStore` — JSON to `~/.casehub/voiceprints/`, atomic rename
- 12 unit tests passing

Both commits landed in the **blocks** repo on branch `issue-231-summarisation-api-extraction`.

### What's Next

**Batch 2: Offline Diarization** (Task 3)
- Add 9 diarization MethodHandles to `SherpaLibrary`
- Add config byte offsets to `SherpaLayouts` (64 bytes, much simpler than STT config)
- Implement `SherpaOnnxDiarizationService` with per-instance handle, per-call clustering config
- Add `Provisioner.ensureDiarizationModels()` for pyannote segmentation model
- Critical: verify byte offsets against C header — wrong offsets cause SIGSEGV

**Batch 3: Avatar Integration** (Tasks 4-5)
- Task 4: Protocol messages (`SpeakerPrompt`, `SpeakerIdentify`, `SpeakerIdentified`), `ConversationTurn` speaker field, `PromptAssembler` speaker formatting, `MessageCodec` encode/decode
- Task 5: `SpeechSession` ring buffer (5s, ~320KB), speaker ID parallel with STT, auto-enrollment state machine, explicit enrollment, CDI wiring in `SpeechProducers`

### Key Design Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Merge #191 + #211 | Shared campplus foundation, single developer, primary use case requires both |
| D2 | Both real-time + offline | Real-time speaker ID is primary (family avatar), offline diarization is secondary |
| D5 | Dual inference paths | sherpa-onnx C API for offline, ORT for real-time — each mode uses its best tool |
| D6 | 3 composable SPIs | Extractor, Registry, DiarizationService — independently usable |
| D8 | campplus model | Already provisioned for CosyVoice3, 192-dim, adequate for 2-8 family speakers |

### Repo State

| Repo | Branch | Status |
|------|--------|--------|
| blocks | `issue-231-summarisation-api-extraction` | 2 commits (Task 1 + Task 2), tests green |
| blocks-ui | `issue-190-speech-denoising` | Design artifacts only (specs, plans, decisions) |
