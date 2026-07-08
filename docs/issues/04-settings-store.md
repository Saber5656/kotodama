# Title

SettingsStore: typed UserDefaults-backed settings

## Summary

Implement `SettingsStore` exposing every setting in DESIGN §7.3 as typed, observable properties over `UserDefaults`, with defaults, validation, and a RAM-based default for the LLM model id.

## Context

All services and UI read configuration through this module; scattering raw UserDefaults keys is forbidden. Keys/defaults are normative in DESIGN §7.3.

## Scope

`Sources/SettingsStore/` + tests. Depends only on `CoreTypes`.

## Detailed Requirements

1. `@MainActor @Observable final class SettingsStore` (or actor + published snapshot — choose `@Observable`, document why) with one property per DESIGN §7.3 row, typed (`StyleID`, enums for `hotkey.mode`, `insertion.mode`, `format.keepWarm` defined here or in CoreTypes — put enums in CoreTypes).
2. Key strings must match DESIGN §7.3 verbatim; expose `enum SettingsKey: String, CaseIterable` and test that all keys are covered.
3. Defaults registered via `UserDefaults.register(defaults:)` at init; `format.llmModelId` default computed: `ProcessInfo.processInfo.physicalMemory >= 16 GiB ? "qwen3-4b-instruct-2507-q4km" : "sarashina2.2-3b-instruct-q4km"` (inject a `MemoryInfoProviding` protocol for tests).
4. Validation/clamping on write: `audio.maxUtteranceSeconds` ∈ [30, 300]; `history.retentionDays` ∈ {0, 7, 30, 90}; unknown persisted enum raw values fall back to defaults on read (corrupt-defaults resilience test).
5. Init takes `UserDefaults` instance (suite injectable for tests; production uses `.standard`).
6. A `migrateIfNeeded()` hook with `settings.schemaVersion` int key (currently 1, no-op) — reserved for future migrations.
7. No side effects beyond UserDefaults (launch-at-login registration is issue 25's concern, reading `app.launchAtLogin` only).

## Acceptance Criteria

- [ ] Every DESIGN §7.3 key implemented; test enumerates `SettingsKey.allCases` against a hardcoded expected list (drift alarm).
- [ ] Defaults test on a fresh suite matches DESIGN §7.3 defaults column, both RAM branches covered via fake `MemoryInfoProviding`.
- [ ] Clamping and corrupt-value fallback tests pass.
- [ ] Concurrency-clean under Swift 6 strict mode (no warnings).

## Validation

`swift test --filter SettingsStoreTests` green in CI; PR lists any deviation from §7.3 (none expected — deviations require DESIGN update in same PR).

## Dependencies

03.

## Non-goals

Settings UI (25), KeyboardShortcuts persistence internals (09 stores its shortcut via the library's own storage; SettingsStore only holds `hotkey.mode`), iCloud/export.

## Design References

DESIGN §7.3 (normative keys/defaults), §13 (keepWarm semantics), §3.1 DoD item 9.
