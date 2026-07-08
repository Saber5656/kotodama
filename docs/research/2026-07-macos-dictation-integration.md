# Research: macOS Integration Techniques & Prior Art for System-wide Dictation

- Date: 2026-07-08
- Status: informs [ADR-005](../decisions/ADR-005-text-insertion.md), [ADR-006](../decisions/ADR-006-sandbox-and-signing.md) and DESIGN.md §10–11
- Question: how do resident dictation apps capture a global hotkey, insert text into the frontmost app, and behave around macOS permissions/security features?

## Prior art

| App | Stack | License | Relevance |
|---|---|---|---|
| VoiceInk | Swift, whisper.cpp (+ Parakeet via FluidAudio) | **GPL-3.0** | Closest OSS analog (menu-bar dictation, local models, "Power Mode" per-app config). **License is GPLv3: feature/UX reference only — no code may be ported into kotodama.** |
| superwhisper, Wispr Flow, Aqua Voice | closed | — | UX benchmarks: hold-to-talk + HUD + style modes; superwhisper popularized LLM post-formatting |
| OpenWhispr | Electron, local+BYOK cloud | MIT | Cross-platform reference; Electron approach rejected for kotodama |
| Explosion-Scratch/whisper-mac, luisalima/local-whisper | Swift/scripts | MIT-ish | Small references for paste-based insertion |

Takeaways: paste-simulation is the de-facto insertion mechanism across all of these; all resident dictation apps run unsandboxed with Accessibility permission; hold-to-talk + toggle are the two expected recording modes.

## Global hotkey options

| Mechanism | Hold-to-talk (press+release)? | Extra TCC permission | Notes |
|---|---|---|---|
| Carbon `RegisterEventHotKey` | Yes (`kEventHotKeyPressed`/`kEventHotKeyReleased`) | None | Battle-tested, works for modifier+key combos; used via the MIT `KeyboardShortcuts` library (sindresorhus) which also provides a SwiftUI recorder UI |
| `NSEvent.addGlobalMonitorForEvents` | Yes | Accessibility (key events) | Listen-only; cannot consume the event |
| `CGEventTap` | Yes, can consume | Input Monitoring (+ Accessibility) | Needed only for Fn/Globe-style keys or event swallowing — defer to v2 |

Decision input: Carbon path (via KeyboardShortcuts) covers v1 with zero additional permissions; Fn/Globe key support is a v2 feature requiring Input Monitoring.

## Text insertion options

| Method | Mechanism | Pros | Cons |
|---|---|---|---|
| Paste simulation (primary) | Save pasteboard → write text (marked `org.nspasteboard.TransientType`) → post ⌘V via `CGEvent` (session tap) → restore pasteboard | Works in ~all apps incl. Electron/web; fast; handles long JA text; bypasses IME composition | Requires Accessibility to post events; briefly occupies pasteboard (clipboard-manager visibility mitigated by transient marking); restore timing heuristic |
| AX direct set (fallback) | `AXUIElement` focused element: insert via `kAXSelectedTextAttribute` | No pasteboard touch | Unreliable in Electron/Chromium and many custom views; some apps expose read-only elements |
| Unicode keystroke synthesis | `CGEventKeyboardSetUnicodeString` chunked typing | No pasteboard touch | Slow for long text; interacts poorly with active IME state; v2 experiment only |
| Clipboard-only (explicit mode) | Write text, notify user to ⌘V | Zero risk, works without Accessibility | Manual step; used as degraded mode |

Ordering for kotodama: paste-sim → AX fallback → clipboard-only, per ADR-005.

## Security features to respect

- **Secure Input**: when a password field (or apps like Terminal with "Secure Keyboard Entry") activates secure input, synthetic keyboard events and key monitoring are restricted. Detect via Carbon `IsSecureEventInputEnabled()`; kotodama must refuse auto-insertion and fall back to clipboard-only with a HUD notice.
- **Pasteboard etiquette**: mark transient writes with `org.nspasteboard.TransientType` (and `ConcealedType` where appropriate) so clipboard managers skip them; restore the user's prior pasteboard contents (best-effort, common item types) shortly after the paste event is delivered.
- **TCC permissions needed (v1)**: Microphone (`NSMicrophoneUsageDescription`) and Accessibility (`AXIsProcessTrustedWithOptions` prompt; also required for CGEvent posting). Input Monitoring NOT required on the Carbon hotkey path.
- **App Sandbox**: incompatible with AX control of other apps and CGEvent posting → app must be unsandboxed; ship with Hardened Runtime, Developer ID signing, and notarization instead (ADR-006). App Store distribution is therefore out of scope.
- **Focus integrity**: capture the frontmost app at hotkey release; re-verify before pasting; if focus changed mid-pipeline, do not paste — clipboard fallback + notice (prevents dictating into the wrong app).
- **IME interaction**: pasting while the target app has uncommitted IME composition behaves inconsistently across apps; there is no public API to query composition state. Treat as a documented quirk covered by the manual insertion test matrix (includes IME-active cases).

## Engine vendoring (whisper.cpp / llama.cpp from Swift)

- Both upstreams provide Apple XCFramework build scripts (`build-xcframework.sh`); Metal is enabled by default on Apple Silicon. The legacy `whisper.spm` SPM mirror is stale (Metal sources excluded) and community SPM wrappers lag upstream — vendor XCFrameworks built from **pinned upstream tags** via a repo script instead, and cache them in CI.

## Distribution facts

- Unsandboxed + Hardened Runtime + Developer ID + `notarytool` notarization → distributable as DMG and via Homebrew cask. Signing/notarization requires the maintainer's Apple Developer ID; all secret material stays outside the repo and CI by default (manual runbook).
- Launch-at-login via `SMAppService.mainApp` (macOS 13+). Menu-bar-only presence via `LSUIElement`.

## Sources

- https://github.com/beingpax/VoiceInk (GPL-3.0 notice, feature set)
- https://github.com/sindresorhus/KeyboardShortcuts (Carbon-based hotkeys + recorder UI, MIT)
- https://hacktricks.wiki/en/macos-hardening/.../macos-input-monitoring-screen-capture-accessibility.html (TCC behavior of event taps)
- https://developer.apple.com/documentation/ (Speech/AX/ServiceManagement/Notarization references)
- https://github.com/ggml-org/whisper.cpp, https://github.com/ggml-org/llama.cpp (XCFramework build paths)
- http://nspasteboard.org (transient/concealed pasteboard type conventions)
