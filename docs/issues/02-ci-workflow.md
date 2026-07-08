# Title

CI: build, lint, and unit-test workflow on macOS arm64

## Summary

Add `.github/workflows/ci.yml` running xcodegen, SwiftLint/SwiftFormat lint, app build, and KotodamaKit unit tests on every PR and push to main, on an Apple Silicon macOS runner.

## Context

Every later issue's PR gate is "CI green" (ISSUE_PLAN §6). The workflow must be fast, deterministic, and hold no secrets (ADR-006: signing never happens in CI).

## Scope

- `.github/workflows/ci.yml` only (plus any small scripts it needs under `scripts/ci/`).

## Detailed Requirements

1. Triggers: `pull_request` (all branches), `push` to `main`. Concurrency group cancels superseded runs per ref.
2. Runner: newest available Apple Silicon GitHub-hosted macOS image (pin the exact label, e.g. `macos-15` or later current — check availability at implementation time and record the choice in the workflow header comment). Pin Xcode via `xcode-select` to the runner's latest stable; echo versions.
3. Steps: checkout → `brew install xcodegen swiftlint swiftformat` (or brew-bundle) → `xcodegen generate` → `swiftformat --lint .` → `swiftlint --strict` → `xcodebuild build -project Kotodama.xcodeproj -scheme Kotodama -destination 'platform=macOS,arch=arm64' CODE_SIGNING_ALLOWED=NO` → `swift test --package-path Packages/KotodamaKit`.
4. **All third-party actions pinned by full commit SHA** (DESIGN §14 T7). Only `actions/checkout` and (optionally) a cache action are allowed.
5. Cache: SwiftPM caches (`Packages/KotodamaKit/.build`, `~/Library/Caches/org.swift.swiftpm`) keyed on `Package.resolved` hash. Leave a clearly-marked placeholder job/step (commented) for the frameworks cache added by issue 06.
6. No secrets consumed anywhere; workflow must set `permissions: contents: read` at top level.
7. Total runtime target: < 15 min cold, < 8 min warm (document measured times in PR).

## Acceptance Criteria

- [ ] CI runs and passes on the scaffold from issue 01 for a test PR.
- [ ] Lint failure and test failure each fail the workflow (demonstrated once with a scratch commit, then reverted).
- [ ] All actions SHA-pinned; `permissions: contents: read` present; zero secrets referenced.
- [ ] `CODE_SIGNING_ALLOWED=NO` — no signing attempted.
- [ ] Cache hit demonstrated on second run (log excerpt in PR).

## Validation

PR includes links to: one green run, one deliberately red run (lint) and one red (test) from scratch commits, plus timing numbers.

## Dependencies

01.

## Non-goals

Release/packaging jobs (31), frameworks build caching (06 wires it into the reserved step), nightly integration-test jobs (14/16 may add `workflow_dispatch` hooks later), Dependabot config (27).

## Design References

DESIGN §16 (CI outline), §14 T7 (action pinning); ADR-006 (no signing in CI); ISSUE_PLAN §6.
