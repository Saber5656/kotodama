# Title

InsertionService: paste-sim / AX / clipboard chain with refusals

## Summary

Implement `InsertionService`: the ADR-005 strategy chain — preflight (focus/secure-input/AX checks), pasteboard snapshot→transient write→⌘V CGEvent→guarded restore, AX fallback, clipboard-only terminal mode — plus the manual app-matrix test document.

## Context

ADR-005 and DESIGN §10 are normative. This is the most security-sensitive user-facing module (DESIGN §14 T2/T3/T4): wrong-app pastes and password-field injection must be impossible by construction.

## Scope

`Sources/InsertionService/` + tests + `docs/qa/insertion-matrix.md`.

## Detailed Requirements

1. API (`InsertionServicing` + fake):
   - `func captureTarget() -> InsertionTarget` (at hotkey release: `{ bundleID, pid, appName, capturedAt }` from `NSWorkspace.frontmostApplication`).
   - `func insert(_ text: SensitiveString, target: InsertionTarget, mode: InsertionMode) async -> InsertionOutcome` where `InsertionOutcome { method: InsertionMethod; refusal: RefusalReason? }`, `RefusalReason = secureInput | targetChanged | axMissing | pasteFailed`.
2. Preflight (auto mode), in order: `mode == clipboardOnly` → skip to (5); `IsSecureEventInputEnabled()` → refuse `.secureInput` → (5-persistent); frontmost bundleID+pid ≠ target → refuse `.targetChanged` → (5-persistent); `AXIsProcessTrusted() == false` → `.axMissing` → (5-persistent).
3. Paste-sim:
   - Snapshot: current `NSPasteboard.general` items' data for types {string, rtf, html, png, tiff, fileURL} best-effort + `changeCount`.
   - Write: `clearContents()` → set string + declare `org.nspasteboard.TransientType` AND `org.nspasteboard.source` (our bundle id).
   - Post: CGEvent ⌘V keyDown+keyUp to `.cghidEventTap` (with `.maskCommand` flags), 10 ms apart.
   - Wait 150 ms (constant, tunable via internal setting) → Restore: only if `changeCount == ours` (a third-party write aborts restore); restore all snapshotted types; if snapshot was empty, `clearContents()`.
   - Verify-success heuristic: none reliable — treat CGEvent post success as `.paste` success; failures of CGEventCreate → try AX.
4. AX fallback: `AXUIElementCreateSystemWide()` → `kAXFocusedUIElementAttribute` → try set `kAXSelectedTextAttribute` to text; confirm via read-back of `kAXValueAttribute` length delta where the attribute is readable; any AX error → (5).
5. Clipboard-only terminal path: write text WITHOUT transient/source marks (must persist), no restore, outcome `.clipboard` (+refusal reason if arrived via refusal). Caller (22) shows HUD "⌘V で貼り付けてください".
6. All CGEvent/AX/NSPasteboard calls behind thin protocol seams (`EventPosting`, `PasteboardHandling`, `FocusReading`, `SecureInputChecking`, `AXWriting`) so the decision chain is 100 % unit-testable; production impls are trivial passthroughs.
7. Logging: method, refusal, target bundle id, text length only.
8. `docs/qa/insertion-matrix.md`: table apps × cases — apps: TextEdit, Notes, Pages, Mail, Safari (textarea + contenteditable), Chrome, Slack, Discord, Terminal, iTerm2, VS Code, JetBrains IDE, Word, Excel; cases: empty field / mid-text cursor / text selected (replace) / IME composing / password field (Safari login + macOS password prompt) / secure-keyboard-entry Terminal. Expected results defined per ADR-005 (password/secure → clipboard-only + notice). Execute ≥ the non-IME rows for this PR; full matrix re-run is a release-gate item (ISSUE_PLAN §6).

## Acceptance Criteria

- [ ] Decision-chain unit tests cover every branch: each refusal reason, snapshot/restore incl. third-party-write abort and empty-snapshot clear, transient marks present in auto mode and absent in clipboard-only, AX fallback ordering, CGEvent failure → AX → clipboard cascade.
- [ ] Restore never overwrites third-party pasteboard writes (explicit test).
- [ ] Secure-input and target-changed refusals occur BEFORE any pasteboard mutation (test asserts no pasteboard calls).
- [ ] Manual matrix: TextEdit/Safari/Chrome/Slack/Terminal(secure)/password rows executed with results table committed.
- [ ] No content logging (grep evidence).

## Validation

CI unit tests + committed matrix results + screen recording of one paste-sim and one secure-input refusal.

## Dependencies

03, 07 (05 for SensitiveString/logging).

## Non-goals

Pipeline wiring/HUD notices (22/20), per-app quirk workarounds (follow-up per matrix results, ISSUE_PLAN §8 U7), unicode-typing mode (v2), restoring exotic pasteboard types beyond the listed set.

## Design References

ADR-005 (normative chain); DESIGN §10, §14 T2/T3/T4, §6.1 rows 15–17; research/2026-07-macos-dictation-integration.md.
