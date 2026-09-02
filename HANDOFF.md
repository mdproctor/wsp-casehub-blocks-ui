# HANDOFF — casehub-blocks-ui

## Last Session

Completed #190 (speech denoising) and #184 (VAD pre-filtering). Built a composable audio pre-processing pipeline: `audio → denoiser → VAD → STT`, each stage independently toggleable at runtime via config properties. Offline DPDFNet denoiser for file-based STT, online GTCRN streaming denoiser for the avatar WebSocket path, Silero VAD for speech/silence gating. Both repos (blocks + blocks-ui) rebased onto main and merged before starting work. #215 (shallow fusion n-gram) skipped — L/High complexity, requires C-level whisper.cpp beam search modifications.

## Immediate Next Step

Advance to #168 (Cache offline recognizer) — next in the .plan queue at position 6/10.

## References

- `specs/issue-190-speech-denoising/` — denoising spec + decisions
- `specs/issue-184-vad/` — VAD spec + decisions
- `plans/2026-09-02-speech-denoising.md` — denoising implementation plan
- `plans/2026-09-02-voice-activity-detection.md` — VAD implementation plan
- `blog/2026-09-02-mdp01-the-audio-pipeline-nobody-documents.md` — technical reference diary
