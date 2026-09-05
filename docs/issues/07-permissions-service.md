# Title

PermissionsService: microphone + Accessibility status/request

## Summary

Implement `PermissionsService` exposing observable microphone and Accessibility permission states, request flows, System Settings deep links, and polling — the single source of permission truth for pipeline guards, onboarding, and settings.

## Context

DESIGN §11: v1 needs exactly two TCC permissions. Guards in the state machine (issue 03 context flags) and degraded modes (§4.4) all read from here. Scattered `AXIsProcessTrusted()` calls are forbidden.

## Scope

`Sources/PermissionsService/` + tests. Depends on `CoreTypes`, `Diagnostics`.

## Detailed Requirements

1. Protocol `PermissionsProviding: Sendable` with async accessors + `AsyncStream<PermissionsSnapshot>` where `PermissionsSnapshot { mic: MicPermission (undetermined|granted|denied); axTrusted: Bool }`; production impl + `FakePermissions` (in a `TestSupport` target product for reuse by 19/22/25/26 tests).
2. Microphone: status via `AVCaptureDevice.authorizationStatus(for: .audio)`; request via `AVCaptureDevice.requestAccess(for: .audio)`. Map all four AVFoundation states to the three-state enum (restricted → denied).
3. Accessibility: status via `AXIsProcessTrusted()`; `promptForAX()` uses `AXIsProcessTrustedWithOptions([kAXTrustedCheckOptionPrompt: true])`; deep links: mic `x-apple.systempreferences:com.apple.preference.security?Privacy_Microphone`, AX `…?Privacy_Accessibility` via `NSWorkspace.open`.
4. Polling: `startMonitoring(interval: 1.0)` re-checks AX (no notification API exists) and mic, emits snapshot only on change; stops when no subscribers (task cancellation). Poll only while a subscriber (onboarding/settings window) is active — never poll in steady state (battery, DESIGN §13).
5. State changes logged via `KotodamaLog.pipeline` (states only).
6. Document (doc comment): TCC resets (`tccutil reset Accessibility <bundleid>`) for manual testing.

## Acceptance Criteria

- [ ] Protocol + fake shipped in TestSupport; production type is init-injectable (AV calls behind a thin `MicAuthorityProviding` seam for unit tests).
- [ ] Unit tests: state mapping (4→3), snapshot dedup (no emission without change), monitor start/stop lifecycle.
- [ ] Manual checklist executed and pasted in PR (fresh TCC: undetermined→prompt→granted; denied→deep-link opens correct pane; AX grant while polling updates within 1.5 s).
- [ ] No polling occurs without subscribers (test with fake clock/task assertion).

## Validation

Unit tests in CI + manual checklist evidence (screen recording or step log) in PR.

## Dependencies

03 (05 for logging).

## Non-goals

Onboarding UI (26), Input Monitoring (v2), hotkey permission concerns (none needed, DESIGN §11), Settings UI presentation (25).

## Design References

DESIGN §11, §4.4 (degraded modes), §13 (no idle polling); research/2026-07-macos-dictation-integration.md (TCC matrix).
