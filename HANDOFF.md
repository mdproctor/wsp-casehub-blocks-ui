# HANDOFF — casehub-blocks-ui

## Last Session

**Issue:** casehubio/blocks#191 — Speaker diarization & identification (merged with #211)
**Queue position:** 8/10 (issue #191 active — all 5 tasks complete)
**Branch:** `issue-190-speech-denoising` (both blocks and blocks-ui repos)

### What Was Done

**All 3 batches complete — 5 commits on blocks repo:**

**Batch 1: Embedding + Registry (Tasks 1-2)**

Task 1 — SPI types + campplus embedding extractor (`94cf7ae`):
- 8 SPI files in `speech-api`: `SpeakerEmbedding`, `SpeakerMatch`, `DiarizedSegment`, `DiarizationOptions`, `SpeakerEmbeddingExtractor`, `SpeakerRegistry`, `VoiceprintStore`, `SpeakerDiarizationService`
- `CampplusSpeakerEmbeddingExtractor` in `speech-sherpa` — ORT with campplus.onnx, 192-dim embeddings, replicates CosyVoice3VoiceEncoder preprocessing path
- `Provisioner.ensureCampplusModel()` — standalone campplus provisioning from HuggingFace
- 3 integration tests passing

Task 2 — voiceprint registry + file persistence (`a6a3362`):
- `CosineDistanceSpeakerRegistry` — ConcurrentHashMap cache, pluggable VoiceprintStore
- `FileVoiceprintStore` — JSON to `~/.casehub/voiceprints/`, atomic rename
- 12 unit tests passing

**Batch 2: Offline Diarization (Task 3)**

Task 3 — FFM diarization bindings (`a436428`):
- 9 diarization MethodHandles added to `SherpaLibrary`
- Config byte offsets in `SherpaLayouts` — 72 bytes total (not 64 as spec assumed)
- Key finding: `SherpaOnnxOfflineSpeakerSegmentationPyannoteModelConfig` has `window_shift_ratio` float field making pyannote sub-struct 16 bytes, not 8 — shifts all subsequent offsets
- `SherpaOnnxDiarizationService` with per-instance handle, per-call clustering config
- `Provisioner.ensureDiarizationModels()` for pyannote segmentation model
- Explicit "cpu" provider strings required (NULL causes SIGSEGV in strlen)
- 3 integration tests passing

**Batch 3: Avatar Integration (Tasks 4-5)**

Task 4 — protocol messages + ConversationTurn (`a1ddda2`):
- `SpeakerPrompt`, `SpeakerIdentify`, `SpeakerIdentified` added to `AvatarMessage` sealed interface
- `MessageCodec` encode/decode for all 3 new message types
- `ConversationTurn` gains nullable `speaker` field with backward-compatible 2-arg constructor
- `DefaultPromptAssembler` formats speaker labels (`Mark (User): Hi`) and adds `Speaking with:` to system prompt
- 8 new tests (23 total MessageCodec + PromptAssembler)

Task 5 — SpeechSession speaker ID + enrollment + CDI (`03ea47b`):
- Ring buffer (5s, 320KB at 16kHz mono float) in `SpeechSession` — caps at RING_BUFFER_SIZE, doesn't wrap
- Speaker identification in `handleStop()` — concurrent with STT, minimum 1.5s audio required
- Auto-enrollment state machine: unknown speaker triggers `SpeakerPrompt`, `SpeakerIdentify` response enrolls
- Explicit enrollment: `SpeakerIdentify` during recording queues name, enrolled on stop
- `withSpeakerServices()` fluent setter (matches existing patterns)
- `SpeechWebSocket` injects `Instance<SpeakerEmbeddingExtractor>` and `Instance<SpeakerRegistry>` with graceful degradation
- 4 CDI producers in `SpeechProducers`: embeddingExtractor, speakerRegistry, voiceprintStore, diarizer
- 7 new tests (108 total speech-ws)

### What's Next

**Issue #211 (Speaker identification/voiceprint) should be closed** — its work was delivered as part of the #191 merge. Use `work next` to advance, then close #211.

**After closing #191/#211, queue position moves to 10/10 — queue drained.** Run `work end` to close the branch.

**Deferred work (from spec):**
- RestVoiceprintStore — REST-backed persistence against platform endpoint
- ECAPA-TDNN upgrade — if campplus insufficient for family use case
- Privacy/GDPR compliance for cloud voiceprint storage
- Confidence threshold calibration — 0.7 cosine threshold is a tunable default
- SpeechSession refactoring — constructor parameter explosion

### Key Design Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Merge #191 + #211 | Shared campplus foundation, single developer, primary use case requires both |
| D2 | Both real-time + offline | Real-time speaker ID is primary (family avatar), offline diarization is secondary |
| D5 | Dual inference paths | sherpa-onnx C API for offline, ORT for real-time — each mode uses its best tool |
| D6 | 3 composable SPIs | Extractor, Registry, DiarizationService — independently usable |
| D8 | campplus model | Already provisioned for CosyVoice3, 192-dim, adequate for 2-8 family speakers |
| NEW | 72-byte config | Pyannote sub-struct has window_shift_ratio field — spec assumed 64 bytes, SIGSEGV caught this |
| NEW | Explicit provider strings | NULL provider causes strlen crash — must set "cpu" explicitly |

### Repo State

| Repo | Branch | Status |
|------|--------|--------|
| blocks | `issue-190-speech-denoising` | 5 new commits (Tasks 1-5), 108 speech-ws tests green |
| blocks-ui | `issue-190-speech-denoising` | Design artifacts only (specs, plans) |
