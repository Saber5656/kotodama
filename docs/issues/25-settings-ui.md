# Title

Settings UI: full preferences window

## Summary

Build the Settings window (5 panes: 一般 / 音声入力 / スタイル / プライバシー / 情報) exposing every DESIGN §7.3 setting with the hotkey recorder, mic picker, insertion mode, history/privacy controls, launch-at-login, and the diagnostics copy action.

## Context

DESIGN §4.4/§7.3; DoD item 9. Settings is the contract surface for every knob defined across issues 04–23; each control binds through SettingsStore (no direct UserDefaults).

## Scope

`App/Settings/` + tests; `SMAppService` integration for login item.

## Detailed Requirements

1. Panes & contents (TabView, ⌘, opens window):
   - **一般**: launch at login (SMAppService.mainApp register/unregister with status readback + error surface), HUD enabled (with note that recording pill always shows — DESIGN §14.5), sound feedback, UI language note (follows system).
   - **音声入力**: hotkey recorder (component from 09) + mode (hold/toggle) + conflict hints; mic device picker (CoreAudio device list via protocol from 08, refresh on hardware change notification); max utterance slider 30–300 s (step 30); ASR model picker (installed ASR models; link "モデルを管理…" → Models window).
   - **スタイル**: default style radio (raw/整文/敬語) with per-style description text from DESIGN §9.1; LLM model picker (installed LLMs + Light-preset hint); keepWarm picker (none/asrOnly/both) with RAM guidance text; styles' prompt text view (read-only disclosure, promptVersion shown).
   - **プライバシー**: history enabled; retention picker (7/30/90/無制限); store-target-app toggle; delete-all button (same confirm as 24); network policy explainer box (static text: モデルDL以外の通信なし + PRIVACY.md link); permissions status rows (mic/AX with grant buttons via 07).
   - **情報**: version/build, repo + license links, third-party attributions link (32's file), "診断情報をコピー" (DiagnosticsReport → pasteboard, persistent write), models installed summary.
2. All bindings via SettingsStore; changes apply immediately (no Save button); side-effectful ones (login item, model switches) show inline progress/error.
3. `SettingsViewModel` per pane, tested with fakes: every §7.3 key round-trips through some control (drift test enumerating SettingsKey.allCases vs a `boundKeys` set — build-time completeness check); SMAppService seam faked; device-list refresh; delete-all confirmation flow; diagnostics copy calls generator with injected providers.
4. Window resizes per pane, remembers last pane; JA/EN complete.

## Acceptance Criteria

- [ ] Drift test proves every SettingsKey is bound (or explicitly listed as UI-less with a reason — e.g., `onboarding.completedVersion`).
- [ ] Login item registers/unregisters verifiably (System Settings screenshot before/after).
- [ ] Hotkey change from Settings takes effect without relaunch (manual, recording).
- [ ] Mic device change mid-session applies to next recording (manual).
- [ ] All VM suites green; JA/EN screenshots of all 5 panes in PR.

## Validation

CI VM tests + manual recordings/screenshots listed above.

## Dependencies

04, 07, 09, 13 (Models window link), 21 (window scene wiring); 05 (diagnostics).

## Non-goals

Onboarding (26), models management UI itself (13), history browsing (24), localization sweep (30 verifies completeness).

## Design References

DESIGN §7.3 (normative keys), §4.2–4.5, §9.1 (style copy), §11, §14.5, §20.
