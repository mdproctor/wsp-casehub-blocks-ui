# Speaker Diarization & Identification — Design Spec

**Issue:** casehubio/blocks#191 (Speaker diarization), casehubio/blocks#211 (Speaker identification/voiceprint) — merged
**Date:** 2026-09-03
**Scale:** XL / High

## Summary

Adds speaker awareness to the CaseHub speech platform: offline diarization (segment a recording by speaker) and real-time speaker identification (recognise which family member is talking to the avatar). Three composable SPI interfaces in `speech-api`, dual inference paths (sherpa-onnx C API for offline, ORT for real-time), campplus as the shared embedding model, hybrid enrollment (auto-detect + explicit), and pluggable voiceprint persistence.

## Architecture

### Dual Inference Paths

Two modes share the campplus embedding model but use different inference paths:

**Offline diarization** — sherpa-onnx's `SherpaOnnxCreateOfflineSpeakerDiarization` C API bundles segmentation (pyannote), embedding extraction (campplus), and spectral clustering internally. We bind the C API via FFM/Panama, pass config + audio, get `(start, end, speaker)` segments back.

**Real-time speaker ID** — extract campplus embedding via `OnnxRuntimeLibrary` (proven path in `CosyVoice3VoiceEncoder.extractSpeakerEmbedding`), compare against registered voiceprints via cosine similarity in the `SpeakerRegistry`.

Both produce campplus 192-dim embeddings. The offline path discovers speakers via internal clustering; the real-time path identifies known speakers via registry matching. They don't need to interoperate.

### Component Diagram

```
speech-api (SPI)                    speech-sherpa (implementations)
┌─────────────────────────┐         ┌──────────────────────────────────────┐
│ SpeakerEmbeddingExtractor│────────▸│ SherpaOnnxSpeakerEmbeddingExtractor  │
│                          │         │   (ORT + campplus.onnx)              │
├─────────────────────────┤         ├──────────────────────────────────────┤
│ SpeakerRegistry          │────────▸│ CosineDistanceSpeakerRegistry        │
│   └─ VoiceprintStore     │────────▸│   └─ FileVoiceprintStore             │
├─────────────────────────┤         ├──────────────────────────────────────┤
│ SpeakerDiarizationService│────────▸│ SherpaOnnxDiarizationService         │
│                          │         │   (sherpa-onnx C API via FFM)        │
└─────────────────────────┘         └──────────────────────────────────────┘
```

### Avatar Pipeline Integration

Current flow:
```
audio → denoiser → VAD → STT → cleanup → LLM → TTS → avatar
```

New flow:
```
audio → denoiser → VAD → ┬─ STT (existing)
                          └─ SpeakerEmbeddingExtractor
                               → SpeakerRegistry.identify()
                                        ↓
                            transcript + speakerLabel
                                        ↓
                            LLM (context: "Talking to [name]")
                                        ↓
                            TTS → avatar
```

Speaker ID runs in parallel with STT on the same audio buffer. Embedding extraction (~50ms) completes before STT (200-500ms+), adding zero latency.

## SPI Layer (`speech-api`)

All types are pure Java records and interfaces with zero foundation dependencies, following the established pattern of `SpeechToTextService`, `TextToSpeechService`, and `TranscriptionOptions`.

### Core Types

```java
record SpeakerEmbedding(float[] vector, int dimensions) {}

record SpeakerMatch(String name, double confidence) {}

record DiarizedSegment(long startMs, long endMs, String speakerLabel,
                       float[] samples, int sampleRate) {}

record DiarizationOptions(int numSpeakersHint,
                          float clusterThreshold) {}
```

- `DiarizedSegment` includes extracted `float[] samples` so consumers can compose with STT without re-reading and slicing the original file
- `DiarizationOptions.numSpeakersHint` — set to `-1` for automatic speaker count detection via threshold-based clustering. When set to a positive integer, forces that exact number of clusters.
- `DiarizationOptions.clusterThreshold` — clustering distance threshold used when `numSpeakersHint` is `-1`. Larger values → fewer speakers. `0.0` for sherpa-onnx default.

### Interfaces

```java
interface SpeakerEmbeddingExtractor {
    SpeakerEmbedding extract(float[] samples, int sampleRate);
}

interface SpeakerRegistry {
    void register(String name, SpeakerEmbedding embedding);
    Optional<SpeakerMatch> identify(SpeakerEmbedding embedding,
                                     double confidenceThreshold);
    List<String> registeredSpeakers();
    void remove(String name);
}

interface VoiceprintStore {
    void save(String name, SpeakerEmbedding embedding);
    Map<String, SpeakerEmbedding> loadAll();
    void delete(String name);
}

interface SpeakerDiarizationService {
    List<DiarizedSegment> diarize(Path audioFile,
                                  DiarizationOptions options);
}
```

### Consumer Composition

**Offline diarization + transcription:**
```java
List<DiarizedSegment> segments = diarizer.diarize(recording, options);
for (DiarizedSegment seg : segments) {
    RecognitionStream stream = stt.startStream(transcriptionOpts);
    stream.acceptSamples(seg.samples(), seg.sampleRate());
    TranscriptionResult result = stream.finalResult();
    // result.text() + seg.speakerLabel() + seg.startMs()
}
```

**Real-time speaker ID (avatar):**
```java
SpeakerEmbedding emb = extractor.extract(turnAudio, 16000);
Optional<SpeakerMatch> match = registry.identify(emb, 0.7);
String speaker = match.map(SpeakerMatch::name).orElse("Unknown");
```

## FFM Bindings (`speech-sherpa`)

### SherpaLibrary Additions

New `MethodHandle` fields for offline diarization C API functions:

| MethodHandle | C Function | Signature |
|-------------|------------|-----------|
| `createDiarization` | `SherpaOnnxCreateOfflineSpeakerDiarization` | `(config*) → handle*` |
| `destroyDiarization` | `SherpaOnnxDestroyOfflineSpeakerDiarization` | `(handle*) → void` |
| `diarizationGetSampleRate` | `SherpaOnnxOfflineSpeakerDiarizationGetSampleRate` | `(handle*) → int32` |
| `diarizationSetConfig` | `SherpaOnnxOfflineSpeakerDiarizationSetConfig` | `(handle*, config*) → void` |
| `diarizationProcess` | `SherpaOnnxOfflineSpeakerDiarizationProcess` | `(handle*, float*, int32) → result*` |
| `diarizationProcessWithCallback` | `SherpaOnnxOfflineSpeakerDiarizationProcessWithCallback` | `(handle*, float*, int32, callback, void*) → result*` |
| `diarizationResultGetNumSegments` | `SherpaOnnxOfflineSpeakerDiarizationResultGetNumSegments` | `(result*) → int32` |
| `diarizationResultSortByStartTime` | `SherpaOnnxOfflineSpeakerDiarizationResultSortByStartTime` | `(result*) → segments*` |
| `diarizationDestroyResult` | `SherpaOnnxOfflineSpeakerDiarizationDestroyResult` | `(result*) → void` |

### SherpaLayouts Additions

Byte offsets for `SherpaOnnxOfflineSpeakerDiarizationConfig` using the 4096-byte zero-filled allocation pattern (per GE-20260826-190329):

```
SherpaOnnxOfflineSpeakerDiarizationConfig:
  // SherpaOnnxOfflineSpeakerSegmentationModelConfig segmentation
  DIARIZATION_SEGMENTATION_PYANNOTE    // const char* — model path
  DIARIZATION_SEGMENTATION_NUM_THREADS // int32
  DIARIZATION_SEGMENTATION_DEBUG       // int32
  DIARIZATION_SEGMENTATION_PROVIDER    // const char*

  // SherpaOnnxSpeakerEmbeddingExtractorConfig embedding
  DIARIZATION_EMBEDDING_MODEL          // const char* — campplus path
  DIARIZATION_EMBEDDING_NUM_THREADS    // int32
  DIARIZATION_EMBEDDING_DEBUG          // int32
  DIARIZATION_EMBEDDING_PROVIDER       // const char*

  // SherpaOnnxFastClusteringConfig clustering
  DIARIZATION_CLUSTERING_NUM_CLUSTERS  // int32
  DIARIZATION_CLUSTERING_THRESHOLD     // float

  // Top-level
  DIARIZATION_MIN_DURATION_ON          // float
  DIARIZATION_MIN_DURATION_OFF         // float
```

Config struct is significantly simpler than the STT config — no 17 nested model sub-configs. Exact byte offsets determined from the C header field ordering and pointer sizes on 64-bit platforms (8 bytes per pointer, 4 bytes per int32/float, with alignment padding).

### Result Struct Reading

`SherpaOnnxOfflineSpeakerDiarizationSegment` is a simple struct:
```
offset 0: float start   (seconds)
offset 4: float end     (seconds)
offset 8: int32 speaker (cluster index)
```

Stride: 12 bytes (possibly 16 with padding — verify from C header). Read via `MemorySegment.get(ValueLayout.JAVA_FLOAT, offset)` and `MemorySegment.get(ValueLayout.JAVA_INT, offset)`.

## Real-Time Embedding Extraction

### SherpaOnnxSpeakerEmbeddingExtractor

Implements `SpeakerEmbeddingExtractor` using `OnnxRuntimeLibrary` with `campplus.onnx`. Follows the exact preprocessing path proven in `CosyVoice3VoiceEncoder.extractSpeakerEmbedding`:

1. Resample input to 16kHz if needed (`AudioResampler`)
2. Compute mel spectrogram (`MelSpectrogram` with `CAMPPLUS_MEL` config: nFft=512, hopLength=160, nMels=80, fMin=0, fMax=8000)
3. Log mel transformation + mean normalisation
4. Run `campplus.onnx` via `OnnxRuntimeLibrary.Session.runFloat()`
5. Return `SpeakerEmbedding(vector, 192)`

**Thread safety:** Each extraction creates a confined `Arena` for tensor allocation and releases ORT values explicitly (per GE-20260829-c497e0 — ORT tensor handles leak despite Arena cleanup).

**Model sharing:** campplus.onnx is already loaded by `Provisioner` for CosyVoice3. The `OnnxRuntimeLibrary.Session` is thread-safe for concurrent `runFloat()` calls — create one session at startup, share across extractions.

### CosineDistanceSpeakerRegistry

Pure Java implementation of `SpeakerRegistry`:

- `ConcurrentHashMap<String, SpeakerEmbedding>` for in-memory cache (same pattern as `VoiceRegistry`)
- `VoiceprintStore` delegate for persistence — loaded at construction time via `store.loadAll()`
- `identify()`: compute cosine similarity against all registered embeddings, return best match above `confidenceThreshold`
- `register()`: add to in-memory cache + persist via `store.save()`
- `remove()`: remove from cache + persist via `store.delete()`

Cosine similarity: `dot(a, b) / (norm(a) * norm(b))`. Simple loop — 192-dim vectors, no need for SIMD.

## Voiceprint Persistence

### FileVoiceprintStore

Local file storage under `~/.casehub/voiceprints/`:

```
~/.casehub/voiceprints/
  mark.json        # { "name": "Mark", "vector": [...], "dimensions": 192 }
  sarah.json
```

- JSON serialization of `SpeakerEmbedding` + name metadata
- Atomic write: write to `{name}.json.tmp`, rename to `{name}.json`
- `loadAll()`: read all `.json` files in directory at startup
- Thread-safe via atomic file operations (no locking needed — one writer per name)

### RestVoiceprintStore

SPI defined but not implemented in this issue. REST implementation is a follow-up when the platform endpoint exists:

```
GET    /api/speakers              → List<VoiceprintEntry>
PUT    /api/speakers/{name}       → register/update
DELETE /api/speakers/{name}       → remove
```

### Privacy Considerations

Speaker embeddings are biometric data under GDPR Article 9 and BIPA:
- File store uses filesystem permissions only (adequate for local demo)
- REST implementation must address encryption at rest and consent
- `SpeakerRegistry.remove()` enables right-to-erasure compliance
- Embeddings are one-way — the original audio cannot be reconstructed from a campplus vector

## Avatar Pipeline Changes (`speech-ws`)

### Structural Changes

1. **`ConversationTurn`** — add speaker field:
   ```java
   record ConversationTurn(String role, String text, String speaker) {}
   ```

2. **`PromptAssembler`** — include speaker context:
   ```java
   // System prompt addition:
   "The current speaker is {speaker}. Previous speakers in this conversation: {history}."
   ```

3. **`SpeechSession`** — after `finalResult()`:
   - Extract embedding from the buffered audio (already accumulated in `WhisperSpeechToText`'s 30s buffer)
   - Call `registry.identify(embedding, 0.7)`
   - If unknown: send `SpeakerPrompt` to client
   - If known: attach `speakerLabel` to the conversation turn

4. **`SpeechWebSocket`** — inject `SpeakerEmbeddingExtractor` and `SpeakerRegistry` via CDI

### Auto-Enrollment Flow

1. User speaks → STT transcribes, embedding extracted in parallel
2. `registry.identify(embedding, 0.7)` returns `Optional.empty()` → unknown speaker
3. Server sends `SpeakerPrompt("I don't recognise your voice — what's your name?")` to client
4. Client responds with `SpeakerIdentify("Mark")`
5. Server calls `registry.register("Mark", embedding)`
6. Subsequent turns from this voice match against "Mark"

### New Protocol Messages

```java
// server → client: ask for speaker name
record SpeakerPrompt(String message) implements AvatarMessage {}

// client → server: provide name for enrollment
record SpeakerIdentify(String name) implements AvatarMessage {}

// server → client: confirm speaker identification
record SpeakerIdentified(String name, double confidence) implements AvatarMessage {}
```

### Explicit Enrollment

Alternative to auto-enrollment — a deliberate "register your voice" flow:

1. Client sends `SpeakerIdentify("Mark")` while recording
2. Server accumulates 3-5 seconds of speech
3. Server extracts embedding and calls `registry.register("Mark", embedding)`
4. Server sends `SpeakerIdentified("Mark", 1.0)` to confirm

Both enrollment paths feed the same registry.

## Model Provisioning

| Model | Purpose | Source | Size | Status |
|-------|---------|--------|------|--------|
| `campplus.onnx` | Speaker embedding (192-dim) | Already provisioned for CosyVoice3 | ~7MB | Exists |
| `sherpa-onnx-pyannote-segmentation-3-0/model.onnx` | Speaker segmentation | sherpa-onnx releases (`speaker-segmentation-models`) | ~5MB | New |

**Provisioner additions:**
- `ensureDiarizationModels()` — download pyannote segmentation model to `~/.casehub/models/sherpa-onnx-pyannote-segmentation-3-0/`
- SHA-256 checksum verification (existing pattern)
- File-lock concurrent download protection (existing pattern)

No new native libraries. `libsherpa-onnx-c-api.dylib` (already loaded by `SherpaLibrary`) includes the diarization functions.

## CDI Wiring (`speech-demo`)

New producer methods in `SpeechProducers`:

```java
@Produces @ApplicationScoped
SpeakerEmbeddingExtractor embeddingExtractor() {
    // Reuse campplus session from CosyVoice3 if available, else create new
    return new SherpaOnnxSpeakerEmbeddingExtractor(campplusSession);
}

@Produces @Singleton
SpeakerRegistry speakerRegistry(VoiceprintStore store) {
    return new CosineDistanceSpeakerRegistry(store);
}

@Produces @Singleton
VoiceprintStore voiceprintStore() {
    return new FileVoiceprintStore(
        Path.of(System.getProperty("user.home"), ".casehub", "voiceprints"));
}

@Produces @ApplicationScoped
SpeakerDiarizationService diarizer() {
    Provisioner.ensureDiarizationModels();
    return new SherpaOnnxDiarizationService(SherpaLibrary.load());
}
```

## Testing Strategy

| Layer | Test Type | What It Covers |
|-------|-----------|----------------|
| SPI records | Unit | `SpeakerEmbedding`, `DiarizedSegment`, `DiarizationOptions` construction |
| `CosineDistanceSpeakerRegistry` | Unit | Register, identify (match/no-match/threshold), remove, thread safety, cosine similarity edge cases (zero vector, identical vectors) |
| `FileVoiceprintStore` | Unit | Save/load/delete round-trip, atomic write, corrupt file handling |
| `SherpaOnnxSpeakerEmbeddingExtractor` | Integration | Extract embedding from known WAV → verify 192-dim output, same speaker → high similarity, different speakers → low similarity |
| `SherpaOnnxDiarizationService` | Integration | Diarize `0-four-speakers-zh.wav` (sherpa-onnx test file) → verify segment count, speaker labels, DiarizedSegment.samples non-empty |
| Diarization + STT composition | Integration | Diarize → feed segments to STT → verify speaker-attributed transcript |
| Avatar pipeline | Integration | Speak → identify → transcript includes speaker label, auto-enrollment flow |

## References

- [sherpa-onnx c-api.h](https://github.com/k2-fsa/sherpa-onnx/blob/master/sherpa-onnx/c-api/c-api.h) — diarization C API function signatures and config structs
- [sherpa-onnx speaker diarization docs](https://k2-fsa.github.io/sherpa/onnx/speaker-diarization/index.html) — model downloads and usage
- [GE-20260826-51c700] — FFM struct layout requires exact match of ALL nested model sub-configs
- [GE-20260826-190329] — oversized zero-filled allocation for FFM config structs
- [GE-20260826-3608ec] — native lib JARs contain JNI libs, not C API libs
- [GE-20260901-defe71] — SIGSEGV from ORT API version mismatch after native library swap
- [GE-20260829-c497e0] — ORT tensor handles leak despite Arena cleanup
- `SherpaLibrary.java` — existing FFM binding patterns
- `SherpaLayouts.java` — existing byte offset patterns
- `CosyVoice3VoiceEncoder.extractSpeakerEmbedding` — proven ORT campplus extraction path
- `VoiceRegistry.java` — thread-safe voice storage pattern
- `SpeechSession.java`, `ConversationTurn.java`, `PromptAssembler.java` — avatar pipeline
