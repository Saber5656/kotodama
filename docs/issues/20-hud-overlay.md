# Title

HUD overlay: floating pipeline-state panel

## Summary

Build the HUD: a non-activating floating `NSPanel` hosting SwiftUI, showing recording level/elapsed time, pipeline states, target app, style chip, cancel hint, success/error toasts — driven by a `HudState` stream and never stealing focus.

## Context

DESIGN §4.2 (normative states/placement) and §14.5 (recording must always be visibly signaled — no stealth mode). The controller (22) emits `HudState`; this issue owns presentation only.

## Scope

`App/Hud/` (panel controller + SwiftUI views + view-model) + tests; `HudState` type added to `CoreTypes` (same-PR small addition, documented).

## Detailed Requirements

1. `HudState` (CoreTypes, Equatable): `hidden | recording(level: Float, elapsed: TimeInterval, target: TargetAppInfo?, style: StyleID) | transcribing(target:, style:) | formatting(style:) | modelLoading(kind: ModelKind) | inserting | success(method: InsertionMethod) | notice(messageKey: String) | error(messageKey: String, recovery: RecoveryAction)`.
2. Panel: `NSPanel(styleMask: [.nonactivatingPanel, .borderless])`, `level: .statusBar`, `collectionBehavior: [.canJoinAllSpaces, .fullScreenAuxiliary]`, `ignoresMouseEvents` false only for error state (click-through otherwise), no shadow issues (rounded SwiftUI material background), width ≤ 360 pt.
3. Placement: horizontally centered, 96 pt above bottom edge of the **screen with keyboard focus** (`NSScreen.main`); reposition on screen-param change notifications; never on top of the Dock context.
4. Content per state: recording → mic glyph + 5-bar level meter (30 Hz updates from AudioCapture levels stream), mm:ss elapsed, target app icon (NSRunningApplication.icon) + name, style chip; processing states → spinner + JA label (認識中… / 整形中… / モデル読込中… / 挿入中…); success → checkmark + method label, auto-hide 600 ms; notice/error → message + (error) recovery button, sticky 4 s or click.
5. Esc handling: while HUD visible in recording/processing states, a local+global Esc keyDown monitor emits `hudEscPressed` to the controller — global monitor requires AX which we have for insertion; if AX missing, Esc works only when… document: use `NSEvent.addGlobalMonitorForEvents(matching: .keyDown)` (listen-only, requires AX) + carbon-registered Esc alternative rejected (would swallow Esc globally) → without AX, cancellation falls back to hotkey-tap (define: in hold mode, quick second tap during processing cancels — controller concern, note in HUD copy "Esc でキャンセル" hidden when AX missing).
6. Reduced-motion respect; dark/light appearance; JA/EN strings via catalog.
7. View-model tests: state→content mapping table, auto-hide scheduling (injected clock), level throttling (≤ 30 Hz), Esc monitor lifecycle (installed only while visible).
8. Previews/snapshot tests for every state (use fixed data; snapshot via ImageRenderer acceptable).

## Acceptance Criteria

- [ ] Panel never activates the app or steals key focus (manual: type in TextEdit while HUD shows — keystrokes uninterrupted; recording proof in PR).
- [ ] All states render per spec; snapshots committed.
- [ ] Multi-display: HUD follows focused screen (manual evidence with 2 displays or documented single-display limitation test).
- [ ] Esc during fake recording emits cancel event exactly once (unit + manual).
- [ ] Recording state is impossible to hide via settings while capture is active (`hud.enabled=false` still shows a minimal recording pill — DESIGN §14.5; test + doc).

## Validation

CI view-model/snapshot tests + manual recording (focus non-theft, Esc cancel, both appearances).

## Dependencies

01, 03 (08's level stream shape).

## Non-goals

Pipeline logic (22), menu bar (21), sounds (22 wires NSSound), settings toggle UI (25).

## Design References

DESIGN §4.2 (normative), §14.5 (no stealth), §4.1 (Esc), §6.1 (states), §19.
