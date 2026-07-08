# Title

HotkeyService: global hold/toggle hotkey via Carbon

## Summary

Implement `HotkeyService` on the `KeyboardShortcuts` library (Carbon `RegisterEventHotKey`): press/release event stream, hold vs toggle semantics handled upstream of the state machine, default ⌥Space, conflict warnings, and the SwiftUI recorder component for Settings.

## Context

DESIGN §11 and §4.1. Carbon hotkeys deliver both pressed and released events with **no additional TCC permission** — this is why v1 excludes Fn/Globe. The state machine consumes plain `hotkeyDown`/`hotkeyUp` events; this service owns registration and event translation only.

## Scope

`Sources/HotkeyService/` + tests; adds the `KeyboardShortcuts` SPM dependency (pinned exact version) to Package.swift and Package.resolved.

## Detailed Requirements

1. Add dependency `sindresorhus/KeyboardShortcuts` pinned `.exact(<latest stable at implementation>)`; record license (MIT) in the deps table row it already has (DESIGN §18).
2. Define `KeyboardShortcuts.Name.dictation` with default `⌥Space` (`.space, modifiers: [.option]`).
3. Protocol `HotkeyProviding: Sendable` → `var events: AsyncStream<HotkeyEvent>` (`down(at: Date)`, `up(at: Date)`) + `func setEnabled(Bool)` (disable during recorder capture and when `dictation.enabled == false`). Fake in TestSupport.
4. Implementation subscribes `KeyboardShortcuts.onKeyDown/onKeyUp for: .dictation`, timestamps with injected `Clock`, forwards raw down/up — **hold/toggle interpretation stays in CoreTypes reduce()** (context.hotkeyMode); service stays mode-agnostic.
5. SwiftUI component `HotkeyRecorderView` wrapping `KeyboardShortcuts.Recorder`, plus caption text and a conflict note list shown when the chosen combo ∈ known-conflicts table: ⌘Space (Spotlight), ⌃Space (input source), ⌘⇧3/4/5 (screenshots), ⌥Space default's own note (some apps use it). Table lives in this module, localizable keys.
6. Behavior guarantees documented + manually verified: events fire when app is background/menu-bar-only; no events while recorder is capturing; re-registration after shortcut change is atomic (no dead period > 100 ms).
7. Manual checklist file `docs/qa/hotkey-checklist.md`: matrix over macOS 14/15/current-latest — press/hold 5 s/release ordering correct; toggle taps; behavior during fast repeat; after display sleep; with an external keyboard; conflict cases register-fail path (registration failure surfaces `KotodamaError`-mapped message, e.g. combo taken by system).

## Acceptance Criteria

- [ ] Unit tests with fake: down/up ordering preserved, timestamps monotonic (injected clock), setEnabled gates emission.
- [ ] Recorder view renders in a preview + stores/restores the shortcut across relaunch (manual, in PR evidence).
- [ ] Hold 5 s in a background app delivers exactly one down and one up (manual log excerpt).
- [ ] `docs/qa/hotkey-checklist.md` committed with at least macOS 14 or 15 column executed; remaining columns marked pending for issue 27/release.
- [ ] No Input Monitoring/Accessibility prompt is triggered by hotkey registration alone (fresh-VM or TCC-reset evidence).

## Validation

CI unit tests + manual checklist excerpt + short screen recording (or event log) of hold-to-talk down/up in PR.

## Dependencies

03, 04.

## Non-goals

Fn/Globe key (v2, Input Monitoring), event swallowing (CGEventTap; not needed — Carbon consumes registered combos by design), multiple shortcuts/profiles, the state machine's minHold tap-guard (03/22).

## Design References

DESIGN §11, §4.1, §7.3 (`hotkey.*`), §21 U6; ADR-001; research/2026-07-macos-dictation-integration.md (hotkey table).
