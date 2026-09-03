## D1: Scope — full pipeline with merged #211

**Choice:** Full pipeline covering diarization engine + STT integration for speaker-attributed transcripts, with #211 (Speaker identification/voiceprint) merged into this issue.
**Alternatives:**
- Diarization engine only — produces (start, end, speaker) segments without transcription. Defers value.
- Keep #191 and #211 separate — risks rework at the seam since embedding extraction is the shared foundation.
**Rationale:** "Multi-speaker transcription with speaker labels" means end-to-end: diarization segments combined with STT output. The embedding extractor is the shared foundation for both diarization and speaker identification, so splitting them creates an artificial boundary.
**Trade-offs:** Larger scope (L/High) but avoids duplicate work on the embedding layer.
**Sources:** casehubio/blocks#191, casehubio/blocks#211, speech-api SPI interfaces
**Exploration:** quick
**Status:** captured

## D2: Mode — both real-time and offline

**Choice:** Both real-time speaker identification (during live avatar conversations) and offline diarization (for recorded audio).
**Alternatives:**
- Real-time only — matches the family avatar use case but misses meeting transcription.
- Offline only — doesn't serve the live avatar use case the user described.
**Rationale:** Real-time speaker ID is the primary use case (avatar recognises which family member is speaking). Offline diarization for recorded conversations is also needed. The embedding extractor is shared between both modes.
**Trade-offs:** More implementation surface than either mode alone.
**Sources:** User requirement: "think about how a family interacts, it would be good if it automatically recognised which family member it's talking to"
**Exploration:** quick
**Status:** captured

## D3: Enrollment — hybrid (auto-detect + explicit)

**Choice:** Hybrid enrollment: auto-detect unknown speakers and prompt for name, plus an explicit enrollment option for better initial accuracy.
**Alternatives:**
- Auto-enroll only — zero friction but lower initial voiceprint quality.
- Explicit enrollment only — higher accuracy but breaks conversation flow.
**Rationale:** Auto-detect provides zero-friction onboarding for families. Explicit enrollment option gives better initial accuracy when desired. Both paths feed the same voiceprint registry.
**Trade-offs:** Two enrollment paths to implement and test.
**Sources:** Avatar demo interaction flow (SpeechSession.java, SpeechWebSocket.java)
**Exploration:** quick
**Status:** captured

## D4: Voiceprint persistence — dual (local + server-side)

**Choice:** SPI-abstracted voiceprint store with both file-system and REST API implementations.
**Alternatives:**
- Local file system only — simple but no multi-device access.
- Server-side only — adds server dependency, privacy concerns for biometric data.
- In-memory only — re-enroll every session, poor UX.
**Rationale:** Local storage works offline and fits the local-first sherpa-onnx model. Server-side enables multi-device access in production. SPI abstraction keeps the core engine agnostic to storage backend.
**Trade-offs:** Two storage implementations to maintain.
**Sources:** Existing pattern: Provisioner caches to ~/.casehub/, platform APIs for production
**Exploration:** quick
**Status:** captured

## D5: Embedding architecture — sherpa-onnx C API for everything

**Choice:** Use sherpa-onnx's `SherpaOnnxCreateSpeakerEmbeddingExtractor` C API for real-time speaker embedding extraction, and `SherpaOnnxCreateOfflineSpeakerDiarization` for offline diarization.
**Alternatives:**
- Reuse campplus.onnx via OnnxRuntimeLibrary for real-time, sherpa-onnx C API for offline only — avoids binding more C functions but uses different embedding models/spaces between modes.
- Pure Java/ORT for everything — maximum control but reimplements sherpa-onnx's diarization pipeline (segmentation, overlap handling, spectral clustering).
**Rationale:** Consistent FFM binding layer matching existing STT/TTS/VAD/denoiser patterns. The C API handles model loading, threading, and ORT session management. Offline and online paths use the same embedding space — critical for voiceprint portability. The 4096-byte zero-filled config pattern applies directly.
**Trade-offs:** More C API functions to bind (embedding extractor + diarization), but follows the established SherpaLibrary pattern exactly.
**Sources:** [GE-20260826-51c700] FFM struct layout, [GE-20260826-190329] oversized zero-filled allocation, sherpa-onnx c-api.h, SherpaLibrary.java, SherpaLayouts.java
**Exploration:** quick
**Status:** captured

## D6: SPI shape — layered, 3 composable interfaces

**Choice:** Three independent interfaces: `SpeakerEmbeddingExtractor` (extract embedding from audio), `SpeakerRegistry` (register/match voiceprints with pluggable `VoiceprintStore` SPI), `SpeakerDiarizationService` (offline diarization + combined diarize-and-transcribe).
**Alternatives:**
- Unified `SpeakerService` — simpler API but less composable; can't use embedding extraction without the full service.
- Two interfaces (online + offline) — clean mode separation but hides the embedding extractor inside each.
**Rationale:** Each interface serves a distinct consumer: embedding extractor for voice cloning reuse (CosyVoice3), registry for identification, diarization service for batch processing. Real-time speaker ID composes Extractor + Registry. Offline diarization is self-contained.
**Trade-offs:** Three interfaces to implement and document vs one, but each is small and focused.
**Depends on:** D5 (embedding architecture determines implementation)
**Sources:** speech-api existing SPI pattern (SpeechToTextService, TextToSpeechService, VoiceActivityFilterFactory)
**Exploration:** quick
**Status:** captured

## D7: Avatar integration — post-STT, per turn

**Choice:** Speaker identification runs in parallel with STT on the same audio buffer, per conversation turn. The speaker label is passed to the LLM as context.
**Alternatives:**
- Pre-STT gate — identify speaker before STT starts. Adds latency to the critical path.
- Parallel with STT (explicit fork-join) — slightly more complex threading for marginal benefit since embedding extraction is faster than STT.
**Rationale:** Embedding extraction takes ~50ms on a few seconds of speech; STT takes 200-500ms+. Running them in parallel on the same buffer adds zero latency. The speaker label enriches the LLM context ("You are talking to Mark") for personalised responses.
**Trade-offs:** Speaker ID result must be available before the LLM call — if extraction fails or times out, the turn proceeds without a speaker label.
**Depends on:** D6 (SPI shape determines what the avatar session calls)
**Sources:** SpeechSession.java (current avatar pipeline), SpeechWebSocket.java
**Exploration:** quick
**Status:** captured
