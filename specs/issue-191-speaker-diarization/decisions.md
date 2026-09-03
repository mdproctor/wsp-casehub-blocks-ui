## D1: Scope — offline diarization only (keep #211 separate)

**Choice:** Offline speaker diarization (#191) only. Speaker identification (#211) remains a separate, coordinated issue.
**Alternatives:**
- Merge #191 and #211 — risks conflating different technical approaches (sherpa-onnx C API vs ORT), inflating scope, and creating artificial architectural coupling.
- Full pipeline covering both diarization and speaker ID — larger scope without proportional benefit since the shared foundation (embedding extraction) operates via different inference paths per mode.
**Rationale:** #191 (L/High, offline diarization via sherpa-onnx C API) and #211 (M/Med, real-time speaker ID via OnnxRuntimeLibrary) have different models, inference paths, scales, and integration points. The "shared embedding extractor" rationale is weaker than initially assumed — offline diarization bundles its own embedding extraction internally via the sherpa-onnx C API. Model choice (D8) is coordinated across both issues but not code-coupled.
**Trade-offs:** Two specs instead of one, but each is correctly scoped and independently deliverable.
**Sources:** casehubio/blocks#191, casehubio/blocks#211, R1-01, R1-02
**Exploration:** quick
**Status:** revised (R1-02 — unmerged to match natural scope boundaries and differing technical approaches)

## D2: Mode — offline diarization only

**Choice:** Offline speaker diarization for recorded audio. Real-time speaker identification is #211's scope.
**Alternatives:**
- Both real-time and offline — doubles implementation surface and conflates architecturally different problems (per-turn bounded comparison vs batch spectral clustering).
- Real-time only — doesn't match #191's stated purpose (multi-speaker transcription with speaker labels).
**Rationale:** #191 is specifically about multi-speaker transcription with speaker labels — a batch/offline problem. Real-time speaker identification ("which family member is speaking") is #211's scope. The only shared component between modes is embedding extraction, and even that operates via different inference paths (sherpa-onnx C API internally for offline, ORT directly for online).
**Trade-offs:** Defers real-time speaker ID to #211, which must be designed and implemented separately.
**Sources:** casehubio/blocks#191 ("Multi-speaker transcription with speaker labels using sherpa-onnx's diarization API"), R1-05
**Exploration:** quick
**Status:** revised (R1-05 — scoped to match issue's stated purpose)

## D3: Enrollment — deferred to #211

**Choice:** Deferred. Enrollment (auto-detect + explicit) is a speaker identification concern, not a diarization concern.
**Rationale:** Offline diarization discovers speakers via spectral clustering — no enrollment step. Speaker enrollment is #211's scope, where the reviewer's concerns about minimum audio duration, detection thresholds, and auto-detect UX should be addressed with proper deep-analysis exploration.
**Sources:** R1-08
**Exploration:** quick
**Status:** revised (deferred — out of scope for offline diarization)

## D4: Voiceprint persistence — deferred to #211

**Choice:** Deferred. Voiceprint storage is a speaker identification concern, not a diarization concern.
**Rationale:** Offline diarization does not persist voiceprints — it discovers and labels speakers within a single recording. The reviewer's privacy analysis concerns (GDPR Article 9, BIPA, encryption at rest, consent requirements) are valid and must be addressed when #211 designs voiceprint persistence. The in-memory-only suggestion for Tier 1 is worth considering there.
**Sources:** R1-07
**Exploration:** quick
**Status:** revised (deferred — out of scope for offline diarization)

## D5: Embedding architecture — sherpa-onnx C API for offline diarization

**Choice:** Use sherpa-onnx's `SherpaOnnxCreateOfflineSpeakerDiarization` C API for offline diarization. Note for #211: real-time embedding extraction should use OnnxRuntimeLibrary with campplus.onnx directly (proven path in CosyVoice3VoiceEncoder).
**Alternatives:**
- sherpa-onnx C API for everything (original choice) — binds additional C functions for speaker embedding extraction when ORT is already proven for campplus. Creates a second parallel extraction path alongside CosyVoice3's existing ORT-based campplus extraction. The claimed "consistent FFM binding layer matching existing STT/TTS/VAD/denoiser patterns" is factually incorrect — no VAD or denoiser bindings exist in the codebase.
- Pure Java/ORT for everything — maximum control but reimplements sherpa-onnx's diarization pipeline (segmentation, overlap handling, spectral clustering) which would be massive scope.
**Rationale:** Offline diarization is sherpa-onnx's strength — the C API bundles segmentation, overlap handling, and spectral clustering that would be prohibitive to reimplement in pure Java. For real-time embedding extraction (recommended for #211), ORT is already proven: `CosyVoice3VoiceEncoder.extractSpeakerEmbedding` extracts campplus embeddings via `OnnxRuntimeLibrary.Session.runFloat()` with established preprocessing (CAMPPLUS_MEL config → log mel → mean normalize → model). Issue #211 explicitly specifies: "Inference via existing OnnxRuntimeLibrary in speech-sherpa — no new native dependencies."
**Trade-offs:** Offline diarization's internal embeddings may be in a different space than ORT-extracted campplus embeddings, but this is acceptable — offline discovers speakers via internal clustering, not by comparing against externally enrolled voiceprints.
**Depends on:** none
**Sources:** SherpaLibrary.java (STT/TTS bindings only — no VAD/denoiser), CosyVoice3VoiceEncoder.extractSpeakerEmbedding (ORT-based campplus path), casehubio/blocks#211, R1-01
**Exploration:** quick
**Status:** revised (R1-01 — split approach, corrected false VAD/denoiser rationale)

## D6: SPI shape — SpeakerDiarizationService only (pure segmentation)

**Choice:** Single `SpeakerDiarizationService` interface returning pure diarization segments `List<DiarizedSegment>` where each segment contains (startMs, endMs, speakerLabel). No combined diarize-and-transcribe method. `SpeakerEmbeddingExtractor` and `SpeakerRegistry` are #211's scope.
**Alternatives:**
- Three composable interfaces (embedding extractor + registry + diarization) — appropriate if both modes are in scope, but overreaches for offline-only. The "voice cloning reuse (CosyVoice3)" rationale was incorrect — CosyVoice3VoiceEncoder's internal SpeakerExtractor takes preprocessed `float[][] logMel`, not raw audio, and cannot use a raw-audio SPI without restructuring its pipeline.
- Combined diarize-and-transcribe method — couples diarization and transcription SPIs, violating the platform pattern of independent composable SPIs (SpeechToTextService doesn't know about TextToSpeechService).
- Unified SpeakerService — conflates concerns.
**Rationale:** Diarization returns pure segments; transcription is composed by the consumer: `segments.forEach(s -> sttService.transcribe(s.audio(), options))`. This follows the platform's existing pattern of independent, composable SPIs. Issue #211 proposed `SpeakerIdentifier` as its SPI name — that naming belongs to #211.
**Depends on:** D5 (sherpa-onnx C API for implementation)
**Sources:** speech-api existing SPI pattern (SpeechToTextService, TextToSpeechService — independent, composable), CosyVoice3VoiceEncoder.SpeakerExtractor (`float[][] logMel` signature — can't use raw-audio SPI), R1-03, R1-09
**Exploration:** quick
**Status:** revised (R1-03, R1-09 — dropped CosyVoice3 reuse claim, removed transcription coupling, scoped to diarization only)

## D7: Avatar integration — deferred to #211

**Choice:** Deferred. Avatar pipeline integration (SpeechSession, ConversationTurn, PromptAssembler) is a real-time speaker identification concern.
**Rationale:** The current avatar pipeline requires significant structural changes for speaker ID: SpeechSession.handleAudio passes chunks to RecognitionStream with no accumulation buffer; ConversationTurn(String role, String text) has no speaker field; PromptAssembler.assemble(String, List<ConversationTurn>) has no speaker parameter; AssembledPrompt(String, String, @Nullable String) has no speaker context. These changes should be designed as part of #211's spec where the integration naturally belongs.
**Sources:** SpeechSession.handleAudio (stream.acceptSamples — no accumulation), ConversationTurn.java, PromptAssembler.java, AssembledPrompt.java, R1-04
**Exploration:** quick
**Status:** revised (R1-04 — deferred to #211 where structural changes naturally belong)

## D8: Model architecture — campplus (explicit choice)

**Choice:** campplus as the default speaker embedding model for offline diarization (and recommended for #211's real-time speaker ID).
**Alternatives:**
- ECAPA-TDNN (SpeechBrain) — issue #211's explicit suggestion, strong VoxCeleb benchmark performance, widely used in speaker verification research, ~192-dim output, ~20MB model. Would require a new model download.
- WeSpeaker / 3D-Speaker — potentially better multilingual support, supported by sherpa-onnx.
- TitaNet (NVIDIA) — state-of-the-art speaker verification accuracy, available as ONNX export.
**Rationale:** campplus is already provisioned and loaded for CosyVoice3 voice cloning (campplus.onnx, 192-dim embeddings). It is sherpa-onnx's default embedding model for its diarization API. Using the same model across CosyVoice3 TTS and speaker diarization avoids provisioning and managing additional model artifacts. For #211's real-time speaker ID via ORT, campplus shares the same model file and preprocessing (CAMPPLUS_MEL config, proven in CosyVoice3VoiceEncoder). ECAPA-TDNN may offer marginally better accuracy on VoxCeleb benchmarks, but campplus is adequate for the family interaction use case, and #211 can revisit if accuracy requirements demand it.
**Trade-offs:** campplus is not the highest-accuracy option available. If #211's evaluation shows insufficient discrimination for the target use case, ECAPA-TDNN is the recommended upgrade path.
**Sources:** campplus.onnx (Provisioner.java — already provisioned), CosyVoice3VoiceEncoder (192-dim embeddings via CAMPPLUS_MEL), sherpa-onnx default diarization model, R1-06
**Exploration:** quick (surfaced by reviewer as implicit decision)
**Status:** captured

## D9: Module placement — speech-api SPI, speech-sherpa implementation

**Choice:** `SpeakerDiarizationService` SPI interface in `speech-api` module (zero foundation dependencies). sherpa-onnx implementation in `speech-sherpa` module.
**Alternatives:**
- All in speech-sherpa — loses the provider-agnostic SPI abstraction that makes implementations swappable.
- New dedicated module — unnecessary; the established two-module pattern handles this cleanly.
**Rationale:** Following the established pattern: pure SPI interfaces → `speech-api` (where SpeechToTextService, StreamingSpeechToTextService, TextToSpeechService already live, all zero foundation deps), implementations → `speech-sherpa` (where SherpaLibrary, OnnxRuntimeLibrary, and all sherpa-onnx bindings live). Since `SpeakerDiarizationService` returns pure segments without transcription coupling, it adds no internal dependency on `SpeechToTextService` at the API level — both remain independently pluggable.
**Sources:** speech-api module (SpeechToTextService.java, TextToSpeechService.java — zero foundation deps), speech-sherpa module (SherpaLibrary.java, OnnxRuntimeLibrary.java), R1-10
**Exploration:** quick (surfaced by reviewer as implicit decision)
**Status:** captured
