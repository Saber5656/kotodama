# Title

Menu bar: status item, menu, style switcher

## Summary

Replace the scaffold menu with the full DESIGN §4.3 menu bar: state-reflecting icon, enable/disable toggle, style radio group, last-result actions, and window openers (History/Models/Settings/About) — all view-model-driven and testable.

## Context

DESIGN §4.3 defines icon states and menu contents. The menu is the app's only persistent surface; it reads pipeline state from the controller's event stream and settings from SettingsStore.

## Scope

`App/MenuBar/` + tests; string catalog additions.

## Detailed Requirements

1. Icon states (template images/SF Symbols): idle `mic` / disabled `mic.slash` (50 % opacity) / recording `mic.fill` + red-dot badge / processing `mic` + spinner overlay (animated NSView or symbol-effect) / error transient `exclamationmark` badge 4 s. Driven by `MenuBarState` derived from DictationController events + `dictation.enabled`.
2. Menu items (order, keyboard-navigable):
   - 音声入力を有効にする (checkmark toggle ↔ `dictation.enabled`)
   - スタイル ▸ そのまま / 整文 / 敬語 (radio ↔ `format.styleId`; disabled + hint when LLM model missing for non-raw)
   - ---
   - 最後の結果を挿入 (re-insert last result via controller; disabled when none)
   - 最後の結果をコピー (pasteboard write, persistent, no transient mark)
   - ---
   - 履歴… / モデル… / 設定… (⌘,) / Kotodama について
   - ---
   - 終了 (⌘Q)
3. "Last result" comes from an in-memory `LastResult` published by the controller (`SensitiveString`; consumed only on user action) — survives until quit, independent of history-enabled setting (document privacy note in code: memory-only).
4. Window opening: History/Models/Settings as separate `Window` scenes (activate app transiently `NSApp.activate`); About = standard about panel with version + link to repo.
5. View-model `MenuBarViewModel` (@MainActor @Observable) with injected controller-events fake + SettingsStore(test suite) — full mapping tests: state→icon, enabled toggle writes settings, style selection writes settings, last-result enablement, LLM-missing disables non-raw styles with explanatory tooltip state.
6. MenuBarExtra style `.menu` (not window) — standard pull-down.

## Acceptance Criteria

- [ ] All items present in order with JA/EN strings; screenshots (both languages) in PR.
- [ ] Icon reflects each state (manual walkthrough with fake controller in DEBUG menu or previews; recording in PR).
- [ ] View-model tests cover every mapping above.
- [ ] Style radio blocked correctly when LLM missing (fake store) with hint.
- [ ] ⌘, opens Settings; ⌘Q quits (manual).

## Validation

CI view-model tests + PR screenshots/recording.

## Dependencies

03, 04 (20/22 provide real events later; this issue ships against the events protocol + fake — define `DictationEventStreaming` protocol here if 22 hasn't landed, coordinated in KotodamaKit).

## Non-goals

HUD (20), pipeline (22), the windows' contents (13/24/25), app icon asset (32), login-item registration (25).

## Design References

DESIGN §4.3 (normative), §7.3 (`dictation.enabled`, `format.styleId`), §19.
