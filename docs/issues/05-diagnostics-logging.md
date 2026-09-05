# Title

Diagnostics: content-free logging, SensitiveString, diagnostics report

## Summary

Implement the `Diagnostics` module: os_log category wrappers, a `SensitiveString` type making transcript content unloggable by construction, pipeline timing records, and a "Copy diagnostics" report generator containing metadata only.

## Context

DESIGN §14 T9 and §20: dictated content must never reach logs or diagnostics. Enforcement is by type design (wrapper), not discipline. Every service logs through this module.

## Scope

`Sources/Diagnostics/` + tests. Depends on `CoreTypes`.

## Detailed Requirements

1. `Logger` factory: `KotodamaLog.pipeline / audio / asr / llm / insert / models / history` — `os.Logger(subsystem: Bundle.main.bundleIdentifier ?? "io.github.saber5656.Kotodama", category: <name>)` (categories per DESIGN §20).
2. `SensitiveString`: struct wrapping `String`; `CustomStringConvertible`/`CustomDebugStringConvertible` return `"<redacted len=\(count)>"`; NOT `Codable`; no public accessor named `value` — the only unwrap is `consumeForInsertion()` / `consumeForHistory()` / `consumeForPrompt()` (three explicitly-named intents, all `@discardableResult`-free plain functions returning `String`). Interpolating into `os.Logger` must produce the redacted form (test via `description`).
3. `TimingRecorder` (actor): append `PipelineTimingRecord { at: Date; timings: PipelineTimings; styleID; asrModelID; llmModelID?; status: SessionStatus }`, ring buffer of last 50, `snapshot()`.
4. `DiagnosticsReport.generate(...)` pure function assembling: app version/build, macOS version, chip + RAM (via injected provider), permission states (injected), installed model ids/sizes/verifiedAt (injected), settings snapshot (non-sensitive keys only — exclude none currently but structure allows), last 20 timing records. Output: human-readable text, JA labels. **Type system must make it impossible to pass a `SensitiveString` into the report** (no API accepts it).
5. Policy doc `Sources/Diagnostics/LOGGING_POLICY.md`: rules (never log content at any level; lengths/durations/ids ok; `SensitiveString` mandatory for transcript-carrying variables across ALL modules) — later issues cite this.
6. Debug-only `assertNoContent(_ line: String)` helper for tests (regex: no JA chars runs > 8 in log lines? keep simple: helper checks a log line contains no substring of a given sensitive sample).

## Acceptance Criteria

- [ ] `SensitiveString` interpolation/description tests prove redaction; compile-time check documented (no `.value`).
- [ ] TimingRecorder ring-buffer behavior tested (51st evicts 1st).
- [ ] DiagnosticsReport golden test: fixed injected inputs → exact expected text snapshot; contains zero content fields.
- [ ] LOGGING_POLICY.md present; SwiftLint custom regex rule `no_rawtext_logging` added (warn on `rawText`/`formattedText` inside `Logger` interpolation) — best-effort regex, documented limitations.

## Validation

`swift test --filter DiagnosticsTests` green; PR shows a sample report output.

## Dependencies

03.

## Non-goals

UI (About pane wiring is 25), file logging (none in v1), crash reporting (never, ADR-004), the network canary (27).

## Design References

DESIGN §20, §14 T9, §14.6; ADR-004.
