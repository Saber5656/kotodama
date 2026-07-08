# Title

AudioCapture: 16 kHz mono capture with level metering

## Summary

Implement `AudioCaptureService`: AVAudioEngine microphone capture converted to 16 kHz mono Float32, input device selection, RMS level stream for the HUD, max-duration cutoff, and clean cancellation. Audio lives in RAM only.

## Context

DESIGN §8 fixes the ASR audio contract (16 kHz mono Float32 PCM). DESIGN §14 A1: audio is never written to disk. The HUD (20) needs a level stream; the controller (22) needs start/stop/cancel with precise semantics.

## Scope

`Sources/AudioCapture/` + tests. Depends on `CoreTypes`, `Diagnostics`.

## Detailed Requirements

1. Protocol `AudioCapturing: Sendable`:
   - `func start(deviceUID: String?, maxDuration: Duration) async throws`
   - `func stop() async -> CapturedAudio` (`CapturedAudio { samples: [Float]; sampleRate: 16_000; duration: TimeInterval }`)
   - `func cancel() async` (discard, free buffers)
   - `var levels: AsyncStream<Float> { get }` (RMS 0…1, ~30 Hz)
   - `var events: AsyncStream<AudioCaptureEvent>` (`started`, `maxDurationReached`, `deviceDisconnected`)
   + `FakeAudioCapture` in TestSupport (feeds fixture samples).
2. Implementation: `AVAudioEngine.inputNode` tap (native format) → `AVAudioConverter` to 16 kHz/mono/Float32; accumulate in a preallocated ring/array sized from maxDuration (120 s ⇒ ~7.7 MB — allocate lazily on start, free on stop/cancel).
3. Device selection: `deviceUID == nil` → system default input (and follow default-device changes mid-recording is NOT required — emit `deviceDisconnected` and stop instead); explicit UID resolved via CoreAudio (`kAudioHardwarePropertyDevices`); unknown UID → fall back to default + log warning.
4. `maxDurationReached`: stop capture exactly at limit, emit event, `stop()` returns what was captured (state machine row 6).
5. Silence gate helper (used by controller): `CapturedAudio.isLikelySilence(threshold: Float = 0.004, minActiveRatio: 0.02) -> Bool` — pure, tested with fixtures (DESIGN §8 silence-hallucination mitigation).
6. Engine start latency measured and logged (budget ≤ 150 ms hotkey→capture, DESIGN §13); pre-warm API `prepare()` that builds the engine without starting the tap.
7. Failure taxonomy: map engine/converter errors to `KotodamaError.audioEngineFailure` with os_log detail (no content — levels/durations only). Tap buffer overflow or converter error mid-capture → stop with error event.
8. Tests: converter pipeline unit-tested by injecting fixture buffers (48 kHz stereo sine + recorded JA WAV under `Tests/Fixtures/`, ≤ 1 MB total) through the same conversion path (`AudioConverterCore` extracted as pure-ish component); silence gate cases; max-duration; cancel frees memory (weak-ref assertion).

## Acceptance Criteria

- [ ] Contract tests green with fixtures: output is 16 kHz mono Float32, length within ±1 frame of expected; WAV fixture round-trip matches golden sample count.
- [ ] Silence gate: silent fixture → true; speech fixture → false.
- [ ] No file writes anywhere in the module (code-review checklist + grep in PR: no FileManager/URL(fileURLWithPath) writes).
- [ ] Manual test app path (`swift run` snippet or debug menu action) records 3 s and prints sample count + peak RMS — transcript in PR.
- [ ] Strict-concurrency clean; levels stream delivers ~30 Hz on a real 3 s capture (log evidence).

## Validation

CI unit tests + manual capture transcript + memory check (Instruments allocation screenshot or `footprint` output showing buffers freed after cancel).

## Dependencies

03, 05.

## Non-goals

VAD/auto-stop-on-silence (v2), ASR submission (15/22), WAV export (never in prod; test-only helpers allowed under Tests/), input gain control.

## Design References

DESIGN §8 (audio contract), §13 (latency/memory), §14 A1/T9, §6.1 rows 3–7.
