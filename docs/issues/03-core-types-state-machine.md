# Title

CoreTypes: domain models, error taxonomy, dictation state machine

## Summary

Implement the pure, dependency-free `CoreTypes` module: domain value types, `KotodamaError`, the dictation state machine as a pure transition function, and injection protocols (clock/UUID). Exhaustive unit tests over the transition table.

## Context

DESIGN §6.1 defines an 18-row transition table that `DictationController` (issue 22) will execute. Encoding it as a pure function in `CoreTypes` lets us test every row without services, and keeps all modules sharing one vocabulary (DESIGN §6.4 error taxonomy, §7 data types).

## Scope

`Packages/KotodamaKit/Sources/CoreTypes/` + tests. No I/O, no Foundation-beyond-basics (Date/UUID/URL ok), no imports of other modules.

## Detailed Requirements

1. Value types (all `Sendable`, `Equatable`, `Codable` where noted):
   - `StyleID: String, CaseIterable` = `raw | seibun | keigo`.
   - `ModelKind` = `asr | llm`.
   - `InsertionMethod` = `paste | ax | clipboard | none`.
   - `SessionStatus` = `inserted | copied | cancelled | failed`.
   - `DictationSessionRecord` (Codable): fields exactly per DESIGN §7.2 columns (id: UUID, startedAt: Date, audioSeconds: Double, rawText: String, formattedText: String?, styleID, targetBundleID: String?, insertionMethod, asrModelID: String, llmModelID: String?, asrMs: Int?, llmMs: Int?, status).
   - `TranscriptResult { text: String; durationMs: Int; avgLogprob: Double? }`.
   - `PipelineTimings { armMs, recordMs, asrMs, llmMs, insertMs: Int? }`.
2. `KotodamaError: Error, Equatable, Sendable` — exactly the cases in DESIGN §6.4, with associated values as specified there (`modelLoadFailed(id: String)`, `insertionFailed(method: InsertionMethod)`, `downloadFailed(reason: DownloadFailureReason)` where `DownloadFailureReason = network|checksum|diskFull|cancelled`). Each case exposes `var messageKey: String` (stable localization key, format `error.<caseName>`) and `var recovery: RecoveryAction` (`openSettings(pane:) | retry | downloadModel | copyInstead | none`).
3. State machine:
   - `enum DictationState: Equatable, Sendable` = `idle, arming, recording(start: Date), transcribing, formatting, inserting`.
   - `enum DictationEvent` = every event named in DESIGN §6.1 (hotkeyDown, hotkeyUp, audioReady, maxDurationReached, esc, asrResult(TranscriptResult), asrFailed(KotodamaError), asrTimeout, formatted(String), formatFailed(KotodamaError), formatTimeout, insertFinished(InsertionOutcome), insertFailed(KotodamaError)).
   - `struct DictationContext` (inputs the guards need): `enabled, micPermitted, axTrusted, asrModelInstalled, llmModelInstalled: Bool`, `styleID: StyleID`, `hotkeyMode: HotkeyMode (hold|toggle)`, `now: Date`, `minHoldMs: Int = 300`.
   - Pure function `func reduce(state: DictationState, event: DictationEvent, context: DictationContext) -> Transition` where `Transition { next: DictationState; effects: [Effect] }` and `enum Effect` = `startAudio, stopAudioAndTranscribe, warmASR, discardAudio, format(text: String, style: StyleID), insert(text: String), fallbackInsertRaw(text: String), writeHistory(SessionStatus), hudShow(HudState), hudError(KotodamaError), cancelASR, cancelFormat, none` (exact list may add cases only with a DESIGN §6.1 table update in the same PR).
   - The function must implement **all 18 rows** of DESIGN §6.1 including guards (tap-guard row 5, busy row 18, raw-style skip row 8, formatting-fallback row 13).
4. Injection protocols: `protocol Clock { var now: Date { get } }`, `protocol UUIDGenerator { func next() -> UUID }` + system defaults. (Services in later issues must take these via init.)
5. Public API is documented with `///` including the DESIGN row numbers each branch implements.

## Acceptance Criteria

- [ ] One unit test per transition-table row (named `row01_…` … `row18_…`), plus negative tests: events invalid for a state produce `next == state` and `effects == [.none]` (define and document this convention).
- [ ] Property test: from any state, `esc` or any error/timeout event leads to `idle` within one transition (invariant DESIGN §6.1).
- [ ] `KotodamaError` cases each have unique `messageKey`; test asserts uniqueness and stability (snapshot of all keys).
- [ ] Module has zero dependencies besides the standard library/Foundation; `swift test` green; 100 % of `reduce` branches covered (coverage report excerpt in PR).

## Validation

`swift test --package-path Packages/KotodamaKit --filter CoreTypesTests` output + coverage excerpt in PR.

## Dependencies

01.

## Non-goals

Executing effects (22), timers/timeout scheduling (22 owns wall-clock; CoreTypes only defines timeout events), persistence (23), localization content for messageKeys (30).

## Design References

DESIGN §6.1 (table — normative), §6.4 (errors), §7.2 (record fields), §4.1 (tap guard, modes).
