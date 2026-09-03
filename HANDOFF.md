# HANDOFF — casehub-blocks-ui

## Last Session

Completed #168 (cache offline recognizer — XS/Low, blocks repo) and #176 (platform-specific Maven JARs for native lib bundling — M/High, blocks repo). 

**#168:** `SherpaOnnxSpeechToText` now caches the offline recognizer keyed by `(modelSize, languageHint)`, implementing `AutoCloseable` for cleanup. Eliminates ~500ms model-loading overhead per `transcribe()` call.

**#176:** Built the full native JAR packaging pipeline:
- `NativeJarExtractor` — scans classpath for native libs at `META-INF/native/sherpa-onnx/<version>/<platform>/`, extracts to Provisioner cache dir with atomic move and file locking
- `SherpaLibrary.load()` gains Tier 1.5 (classpath extraction) between system path and local cache
- `NativePackager` — build-time entry point that copies provisioned native libs into JAR resource structure
- 5 Maven modules (`speech-sherpa-native-{osx-arm64,osx-x64,linux-x64,linux-arm64,win-x64}`) using `exec-maven-plugin` to invoke NativePackager during `generate-resources`
- Verified: osx-arm64 JAR builds and contains both `libsherpa-onnx-c-api.dylib` + `libonnxruntime.dylib` at the correct resource path

## Immediate Next Step

Run `work next` to advance to #191 (Speaker diarization) — position 8/10 in the queue.

## Queue State

Position 7/10 (7 done, 3 remaining):
- [x] #139, #148, #187, #190, #215, #184, #168, #176
- [ ] #191 — Speaker diarization
- [ ] #211 — Speaker identification/voiceprint

## References

- `specs/issue-176-native-jars/` — design spec + decisions
- `plans/2026-09-03-native-jar-bundling.md` — implementation plan
- `specs/issue-190-speech-denoising/` — denoising spec (prior session)
- `specs/issue-184-vad/` — VAD spec (prior session)
