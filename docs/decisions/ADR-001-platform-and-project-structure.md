# ADR-001: Native Swift/SwiftUI macOS app with XcodeGen + modular SwiftPM core

- Status: Accepted (2026-07-08)
- Deciders: user (platform choice, 2026-07-07) + design agent (structure)

## Context

kotodama v1 is a resident menu-bar dictation app that needs global hotkeys, CGEvent posting, Accessibility API use, AVAudioEngine capture, Metal-accelerated inference, and tight TCC-permission UX. Candidate stacks: native Swift/SwiftUI, Tauri v2 (Rust), Electron, Python. Implementation will be executed by lower-capability agents, so the project format must be editable without opaque binary-ish files.

## Decision

1. Native **Swift 6 (strict concurrency) / SwiftUI**, macOS 14.0+, **arm64 only**.
2. **XcodeGen**: `project.yml` is committed; `.xcodeproj` is generated and gitignored. Agents edit YAML, never pbxproj.
3. Business logic lives in a local SwiftPM package `Packages/KotodamaKit` with one library target per module (CoreTypes, SettingsStore, Diagnostics, PermissionsService, AudioCapture, HotkeyService, ModelCatalog, ASRService, FormattingService, InsertionService, HistoryStore, DictationController). The app target contains UI + DI wiring only.
4. Dependency direction: App → DictationController → services → CoreTypes; only ModelCatalog may touch networking (see ADR-004).

## Consequences

- Best-in-class access to macOS integration APIs; smallest permission/latency friction; single-language codebase.
- macOS-only v1 (Windows/Linux explicitly out of scope; revisiting means a separate core-extraction effort).
- Requires Xcode + `brew install xcodegen` locally and in CI (documented in issue 01/02).
- Merge conflicts on project files eliminated; module boundaries are compiler-enforced, which keeps issue scopes independent.

## Alternatives considered

- **Tauri v2**: cross-platform future, but macOS AX/CGEvent/TCC work would be custom Rust↔ObjC bridging anyway; higher v1 risk. Rejected for v1.
- **Electron**: memory footprint + distribution weight contradict a lightweight resident utility. Rejected.
- **Python (MLX)**: fastest prototyping, unfit for resident-agent UX/distribution. Rejected.
- **Plain committed xcodeproj**: hostile to agent edits and merges. Rejected.
