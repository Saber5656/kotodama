# Title

ASRService: model resolution, warm-up, lifecycle

## Summary

Implement `ASRService` (actor) orchestrating the ASR engine: resolve the configured model via ModelStore, keep-warm policy, `warm()` for the arming state, transcription entry point with timeout, model-switch handling, memory-pressure unload, and error mapping.

## Context

DESIGN §8 (lifecycle), §13 (keepWarm=asrOnly default, memory-pressure), §6.1 rows 3–11 (warmASR effect, asrTimeout). DictationController (22) talks to this service, never to the engine.

## Scope

`Sources/ASRService/` (service layer) + tests.

## Detailed Requirements

1. API (`ASRServicing` protocol + fake in TestSupport):
   - `func warm() async` — load configured model if not loaded (no-op if warm); called on arming; never throws (failures logged; surfaced on transcribe).
   - `func transcribe(_ audio: CapturedAudio) async throws -> TranscriptResult` — resolves model (SettingsStore `asr.modelId` → ModelStore fileURL), ensures loaded, runs engine with 30 s timeout (`transcriptionTimeout`), maps engine errors to taxonomy; on `modelLoadFailed` triggers ModelStore.reverify(id:) in background and rethrows.
   - `func unload(reason: UnloadReason) async` (`modelSwitch|memoryPressure|shutdown`).
   - `var state: AsyncStream<ASRServiceState>` (`unloaded|loading|ready(modelID)|transcribing`) for HUD/cold-load messaging.
2. Settings observation: when `asr.modelId` changes → if idle, unload+lazy-reload on next warm; if transcribing, complete current then switch (test both).
3. Keep-warm: per `format.keepWarm` (`asrOnly`/`both` keep loaded; `none` → unload after each transcription + 60 s grace timer, injected clock).
4. Memory pressure: subscribe `DispatchSource.makeMemoryPressureSource(.critical)` → unload unless transcribing (defer until done). Log event.
5. Silence gate: apply `CapturedAudio.isLikelySilence` before engine call → return empty TranscriptResult (maps to `emptyTranscript` upstream) without engine invocation (test).
6. Timeout implementation: structured `withThrowingTaskGroup` race; on timeout call `engine.cancelCurrent()` and throw taxonomy error only after engine confirms abort or a 1 s hard cap.
7. All timings recorded to `TimingRecorder`.

## Acceptance Criteria

- [ ] Unit tests with `FakeASREngine` + fake store/settings/clock: warm idempotency; cold transcribe loads then runs; timeout cancels engine; model-switch during idle and during transcribe; keepWarm `none` grace unload; memory-pressure deferral while transcribing; silence gate short-circuit; reverify triggered on load failure.
- [ ] State stream sequences asserted for cold and warm paths.
- [ ] Strict-concurrency clean; no engine symbol leaks outside `ASRService` sources (grep in PR).

## Validation

CI unit tests; local smoke with real engine behind `KOTODAMA_IT=1` (one warm+transcribe log excerpt in PR).

## Dependencies

04, 14 (12 transitively).

## Non-goals

Pipeline decisions (22), formatting (17), UI (20/25), download (11).

## Design References

DESIGN §8, §13 (memory policy), §6.1 rows 3–11, §6.4; ADR-002.
