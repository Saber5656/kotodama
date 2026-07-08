# Title

Onboarding: permissions + model-download walkthrough

## Summary

Build the five-step first-launch onboarding (Welcome → Mic → Accessibility → Models → Try it): resumable, skippable-with-degraded-modes, driving PermissionsService and the Models preset picker, ending in a live test dictation.

## Context

DESIGN §4.4 (normative steps + degraded modes); DoD item 1. This flow decides whether a non-technical user ever reaches a working state; every exit must leave the app in a coherent, recoverable mode.

## Scope

`App/Onboarding/` + tests; uses `ModelPresetPicker` component (13), PermissionsService (07), SettingsStore (04).

## Detailed Requirements

1. Trigger: `onboarding.completedVersion < 1` at launch → open onboarding window (regular window, activates app); menu item "設定アシスタントを再実行" in Settings→情報 reopens it.
2. Steps (back/next navigation, progress dots, resumable via `onboarding.lastStep` transient — plain UserDefaults int, not in §7.3 table → add to SettingsStore in this PR with DESIGN §7.3 row appended in same PR):
   - **1 Welcome**: value prop (3 bullets: ローカル処理 / ホットキーで即入力 / 整文・敬語), privacy one-liner + PRIVACY link.
   - **2 Mic**: status-aware (undetermined→request button; denied→deep-link + recheck; granted→auto-advance after 800 ms).
   - **3 Accessibility**: explainer graphic (text ok v1) why AX is needed (自動貼り付け); prompt button; live polling checkmark; **skip allowed** → sets clipboard-only expectation banner state.
   - **4 Models**: `ModelPresetPicker` (Recommended ≈3.0 GB / Light ≈2.4 GB / ASR-only ≈0.5 GB; RAM-based default per DESIGN §4.4) with progress UI; requires ≥ 1 ASR model to proceed (or explicit "later" → app stays in setup-required state: menu shows 要セットアップ badge and dictation start surfaces the models error).
   - **5 Try it**: embedded multiline text field; instruction "⌥Space を押しながら話してください"; live HUD appears; success detection (text inserted into the field) → 完了 button enabled; troubleshooting disclosure (hotkey conflicts, mic level).
3. Degraded-mode matrix (test each): skip AX → clipboard-only banner in step 5 + menu note; skip models → setup-required; deny mic → cannot pass step 2 (explain + System Settings link; quit allowed).
4. Completion: set `onboarding.completedVersion = 1`; window closes; menu bar highlights briefly (one-time popover "メニューバーから設定できます").
5. VM tests (fakes for permissions/store/download): step gating logic, resume from step N, auto-advance on grant, skip paths set correct app-mode flags, completion writes version; snapshot per step.
6. Fresh-machine manual run (new macOS user account or VM): full happy path + AX-skip path recorded.

## Acceptance Criteria

- [ ] Fresh-account run reaches working dictation without touching Terminal or docs (recording in PR).
- [ ] All degraded paths land in the documented states (matrix table filled with evidence).
- [ ] Resume works after force-quit at each step (spot-check 2 steps).
- [ ] VM tests + snapshots green; JA/EN complete.
- [ ] Re-run entry point from Settings works post-completion.

## Validation

CI tests + fresh-account recordings (happy + AX-skip) + degraded-matrix table in PR.

## Dependencies

04, 07, 13 (22 for live step 5 — schedule after wave 5; step 5 may land behind a flag if 22 is not merged, but issue closes only with live test working).

## Non-goals

Settings panes (25), models window (13), marketing copy polish (32 may revise strings), video/animation assets.

## Design References

DESIGN §4.4 (normative), §3.1 DoD 1, §4.5, §7.3 (+ new row this PR), §19.
