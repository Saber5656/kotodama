# Title

Release: DMG script, sign/notarize runbook, release checklist

## Summary

Create the release toolchain: reproducible unsigned-archive build script, DMG packaging, the maintainer-local signing + notarization + stapling runbook (secrets never in repo/CI), SHA256SUMS generation, versioning/CHANGELOG conventions, GitHub Release checklist, and a Homebrew cask draft.

## Context

ADR-006: Developer ID + Hardened Runtime + notarization, executed manually by the maintainer; CI stays secret-free. DoD item 12 requires the runbook to be executed once end-to-end.

## Scope

`scripts/release/` (build-archive.sh, make-dmg.sh, notarize-runbook.md, verify-release.sh), `CHANGELOG.md` seed, `docs/RELEASING.md`, `Casks/kotodama.rb` draft (in-repo until tap decision), optional CI `release-artifacts` job (unsigned zip on tag).

## Detailed Requirements

1. `build-archive.sh`: clean → `xcodegen` → frameworks `--check` → `xcodebuild archive` (Release, arm64) → export unsigned `.app` to `build/release/`; embeds version from `project.yml` (single-source: script reads it; tag must match — verify step).
2. `make-dmg.sh`: plain `hdiutil` (no third-party dep): app + Applications symlink + volume name `Kotodama <version>`; deterministic-ish layout; outputs `Kotodama-<version>.dmg`.
3. `notarize-runbook.md` (manual, step-by-step, copy-pasteable): prerequisites (Developer ID cert in login keychain, App Store Connect API key or app-specific password stored in **keychain profile** `notarytool store-credentials` — instructions only, no secrets recorded); steps: `codesign` app (deep, hardened runtime, timestamp; entitlements file path) → verify (`codesign -vvv --strict`, `spctl -a -vv` expected fail pre-notarization noted) → sign DMG → `xcrun notarytool submit --wait` → `xcrun stapler staple` both → final `spctl -a -vv -t install` accepted evidence. Troubleshooting appendix (common notary rejections: unsigned nested frameworks — our two XCFrameworks — signing order documented).
4. `verify-release.sh`: run on the final DMG — checksum print, signature chain, notarization ticket (stapler validate), Gatekeeper assessment, app launches headless-smoke (`open` + pgrep + quit).
5. Conventions in `docs/RELEASING.md`: SemVer; build number bump rule; CHANGELOG (Keep a Changelog, JA acceptable with EN summary); tag `v<version>`; GitHub Release template (JA+EN summary, SHA256SUMS block, macOS requirement line, privacy one-liner); release checklist referencing gates (eval 18, perf 28, security checklist 27, insertion matrix 19, l10n 30, fresh-VM onboarding 26).
6. Optional CI job on tag push: build unsigned zip artifact only (evidence for reproducibility; explicit comment why unsigned).
7. `Casks/kotodama.rb` draft: version/sha256 placeholders, `depends_on macos: ">= :sonoma"`, arm64 requirement, livecheck GitHub releases; publication decision (homebrew/cask vs own tap) recorded as open question for after first release.
8. Execute the whole flow once with the maintainer (signing steps are **user-manual** per global rules — agent prepares everything, user runs the credential-touching commands): produce a signed, notarized, stapled DMG of a pre-release build; attach `verify-release.sh` output.

## Acceptance Criteria

- [ ] Unsigned pipeline (archive→DMG) runs clean on CI-less local machine + optional tag job green.
- [ ] Runbook executed once end-to-end by maintainer; `verify-release.sh` full-pass output committed to `docs/qa/release-verification-<version>.md` (secrets redacted by construction).
- [ ] Gatekeeper accepts the stapled DMG on a second machine (or fresh VM) with default settings — evidence.
- [ ] No secret material anywhere in repo/CI logs (review + grep).
- [ ] RELEASING.md checklist complete and cross-links all release gates.

## Validation

Committed verification outputs + second-machine Gatekeeper evidence + reviewer walkthrough of runbook for completeness.

## Dependencies

01, 02, 06 (gates 18/19/26/27/28/30 for the *checklist content*, not for scripting).

## Non-goals

Auto-update (v2 ADR), App Store (never), CI-held signing (never per ADR-006), crash symbol servers.

## Design References

ADR-006 (normative); DESIGN §16, §3.1 DoD 12; global rule: secrets handled manually by user.
