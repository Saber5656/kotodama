# Title

WhisperASREngine: whisper.cpp wrapper actor

## Summary

Implement `WhisperASREngine` (actor) wrapping the vendored whisper.xcframework C API: model load/unload, Japanese transcription of 16 kHz Float32 samples with pinned decode parameters, cancellation via abort callback, timing capture — plus an A/B fixture report (kotoba vs large-v3-turbo) that pins final decode params.

## Context

DESIGN §8. The audio contract comes from issue 08; model paths from issue 12. Known risks (DESIGN §21 U3): distil-decoder repetition on silence/short audio; this issue must resolve them empirically and encode the result.

## Scope

`Sources/ASRService/Whisper/` + tests + `docs/qa/asr-ab-report.md`. C interop with whisper.xcframework only here.

## Detailed Requirements

1. Conform to `ASREngine` protocol (define in `Sources/ASRService/ASREngine.swift`):
   - `func load(modelPath: URL) async throws` (→ `modelLoadFailed(id)` on failure; capture load ms)
   - `func unload() async`
   - `func transcribe(_ audio: CapturedAudio, config: ASRConfig) async throws -> TranscriptResult`
   - `func cancelCurrent()`
   - `var isLoaded: Bool { get async }`
   `ASRConfig { language: String = "ja"; translate = false; timestamps = false; beamSize: Int?; temperatureFallback: Bool }` — defaults pinned by the A/B below.
2. whisper.cpp usage (verify exact symbol names against the pinned tag; typical): `whisper_init_from_file_with_params` (Metal on), `whisper_full` with `whisper_full_params`: `language="ja"`, `no_context=true`, `single_segment=false`, `print_*=false`, `suppress_blank=true`, threads = `ProcessInfo.activeProcessorCount` capped 8, greedy or beam per A/B result; abort via `encoder_begin_callback`/`abort_callback` checking a cancellation flag.
3. Result text: concatenate segments, trim; strip known whisper artifacts: leading/trailing whitespace, `[音楽]`-style bracketed non-speech tags (list from A/B observations, kept in one regex constant with tests).
4. Timing: `durationMs` = wall time of `whisper_full`; log via `KotodamaLog.asr` (durations, model id, sample count — no text).
5. Memory: `unload()` frees context; loading a second model requires explicit unload first (actor state machine: unloaded/loading/loaded/transcribing — invalid calls throw programmer-error assertions in debug, mapped errors in release).
6. Tests:
   - Unit: state machine misuse, artifact-stripping regex, cancellation flag path (with a `FakeWhisperBackend` seam if feasible — else document why C API is untestable without model and rely on integration tier).
   - Integration (gated `KOTODAMA_IT=1`, not in PR CI): fixtures `Tests/Fixtures/asr/` — ja-10s.wav (scripted read), ja-3s-casual.wav, silence-3s.wav, ja-numbers.wav (dates/amounts). Assert: non-empty for speech; **golden keyword containment** (each fixture has expected-keywords list, ≥ 90 % present) rather than exact match; silence → empty/near-empty after silence gate; each ≤ budget (10 s audio ≤ 2.5 s p95 warm on dev machine, recorded not asserted in CI).
7. A/B report `docs/qa/asr-ab-report.md`: kotoba-q5_0 vs turbo-q5_0 on all fixtures × {greedy, beam5} × {temp-fallback on/off}: text output, duration, repetition/hallucination notes → conclude pinned `ASRConfig` defaults + any param deltas; update DESIGN §8 in the same PR if conclusions differ from its assumptions.

## Acceptance Criteria

- [ ] Unit tests green in CI (no model download in CI).
- [ ] Integration suite passes locally with both models (transcript + timing table in PR).
- [ ] Cancellation: aborting a 10 s transcription returns within 300 ms (measured, in report).
- [ ] Repetition/silence-hallucination behavior documented with chosen mitigations encoded (params or gate) — DESIGN §21 U3 closed or spawned as follow-up issue.
- [ ] No content logging (grep evidence per LOGGING_POLICY).

## Validation

CI units + local `KOTODAMA_IT=1 swift test --filter WhisperIntegration` transcript + `docs/qa/asr-ab-report.md` in PR.

## Dependencies

06, 08 (contract types), 12 (paths for integration run).

## Non-goals

Model lifecycle policy/warm-up (15), streaming partials (v2), language auto-detect (ja forced; turbo entry still ja-forced in v1), SpeechAnalyzer (v2).

## Design References

DESIGN §8, §13 (budgets), §21 U3; ADR-002; research/2026-07-asr-engines.md.
