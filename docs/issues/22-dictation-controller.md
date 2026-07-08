# Title

DictationController: end-to-end pipeline state machine

## Summary

Implement `DictationController` (actor): executes the CoreTypes `reduce()` state machine against real services — hotkey→audio→ASR→formatting→insertion→history — with timeout scheduling, cancellation, concurrency guard, HUD/menu event publication, sounds, and timing records. This is the integration centerpiece.

## Context

DESIGN §6.1 (18-row table) is already encoded as a pure function (issue 03); this issue owns *effect execution* and wall-clock concerns. Every service arrives via protocol + fake (07/08/09/15/17/19/23), so the controller is fully unit-testable.

## Scope

`Sources/DictationController/` + tests; DI wiring in `App/` composition root.

## Detailed Requirements

1. Inputs: `HotkeyProviding.events`, HUD Esc events, menu commands (re-insert last result, enable/disable). Context assembly per event from: SettingsStore, PermissionsProviding snapshot, ModelStore installed-check (cached, refreshed on model changes).
2. Effect executor: map each `Effect` from `reduce()` to service calls:
   - `startAudio` → `audio.prepare()` (during arming) + `start(deviceUID:maxDuration:)`; `warmASR` → `asr.warm()` (fire-and-forget task).
   - `stopAudioAndTranscribe` → `audio.stop()` → silence gate → `asr.transcribe` (30 s timeout owned here; race + cancel).
   - `format(text:style:)` → `formatting.format` (service owns its 20 s timeout; controller maps `.fallbackRaw` to row-13 semantics + HUD notice).
   - `insert(text:)` → `insertion.captureTarget()` was taken at hotkeyUp (store on transition 4) → `insertion.insert(text, target, mode)`; outcome→events (rows 15–17).
   - `writeHistory(status)` → build `DictationSessionRecord` (UUID/Clock injected) → `history.append` (skip when disabled — HistoryStore owns the check; controller always calls).
   - `hudShow/hudError` → publish `HudState` on `events` stream; sounds: start/stop/success/error via `SoundPlaying` protocol (NSSound impl; respects `audio.soundFeedback`).
2. Timeout scheduling: controller owns wall-clock timers (injected `Clock` + `Task.sleep` abstraction) for asrTimeout/formatTimeout only as backstops if services hang beyond their own budgets (+5 s grace), firing the corresponding events.
3. Cancellation: Esc / second-tap-during-processing (hold mode) / disable-toggle mid-pipeline → emit esc event → reduce → cancel in-flight service calls (`asr`/`formatting` cancel, audio.cancel) — verify no history row status other than `cancelled` is written.
4. Concurrency guard: events are serialized through the actor; hotkeyDown in non-idle states follows row 18 (busy flash) — no queuing.
5. `LastResult` (menu feature): retain final text as `SensitiveString` in memory; `reinsertLast()` runs capture-target + insert directly (bypasses ASR states; uses a mini-flow: idle→inserting→idle reusing rows 15–17 semantics).
6. Publication: `var events: AsyncStream<DictationUIEvent>` (`hud(HudState)`, `menuState(MenuBarState)`, `lastResultChanged`) — single source for 20/21.
7. Startup sequence: restore enabled state, register hotkey, prewarm per keepWarm settings, log app-start diagnostics line.
8. Tests (all fakes + virtual clock) — minimum suite:
   - Happy paths: raw / seibun / keigo styles (verify service call order + history row content + timings recorded).
   - Every error row: mic denied, AX missing (→clipboard outcome), no models, asr fail/timeout/empty, format fallback, secure input, target changed, insert fail.
   - Cancellations at each state (recording/transcribing/formatting/inserting-before-commit).
   - Tap guard (<300 ms), toggle mode start/stop, max-duration path, busy-flash row 18.
   - keepWarm interactions (warm called during arming; LLM prewarm only when `both`).
   - Timeout backstops fire when a fake hangs.
   - Re-insert last result (success + target-changed refusal).
   Target ≥ 95 % line coverage of the module.

## Acceptance Criteria

- [ ] Full fake-based suite green in CI with coverage evidence.
- [ ] Manual end-to-end on dev machine: hold ⌥Space → speak → keigo text lands in TextEdit; recording + logs (timings visible, no content) in PR. Repeat for clipboard-only mode without AX.
- [ ] No effect executes outside its table row (review checklist mapping code branches → row numbers).
- [ ] Sounds respect setting; HUD events sequence matches §4.2 for happy path (test asserts sequence).

## Validation

CI suite + manual E2E recording + timing log excerpt in PR (this closes ISSUE_PLAN wave-5 exit criterion).

## Dependencies

05, 07, 08, 09, 15, 17, 19, 20 (HudState type), 23.

## Non-goals

UI rendering (20/21), performance tuning (28), error-message copy (29), onboarding gating (26 — controller only exposes prerequisites via context).

## Design References

DESIGN §6.1–6.3 (normative), §4.1–4.3, §13 (warm hooks), §14 T3 (target capture timing); ADR-005.
