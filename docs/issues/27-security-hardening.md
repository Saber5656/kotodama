# Title

Security enforcement: network isolation, pasteboard/logging audits

## Summary

Turn DESIGN §14's paper controls into enforced artifacts: SwiftLint network-ban rule, debug-build network canary, ATS strictness, dependency pinning checks + Dependabot, logging-content audit, pasteboard-hygiene verification, PR-template security checklist, and the release security checklist with captured evidence.

## Context

DESIGN §14.6 lists the enforcement artifacts; ADR-004 promises "no network beyond model downloads" as a *verifiable* claim. This issue makes violations mechanically loud rather than policy-only.

## Scope

`.swiftlint.yml` custom rules, `Sources/Diagnostics/NetworkCanary.swift` (debug-only), `.github/` (PR template, dependabot.yml), `docs/qa/security-checklist.md`, ATS/Info.plist review, small test additions. No feature code.

## Detailed Requirements

1. **SwiftLint custom rule** `no_network_outside_modelcatalog`: regex-match `URLSession|NWConnection|Network\.framework|CFNetwork|NSURLConnection|curl|Socket` in `Sources/**` excluding `Sources/ModelCatalog/**` and `Tests/**` → error. Companion rule `no_url_request_in_app_target` for `App/**`. Document regex limitations (comments/strings) + add a CI grep script `scripts/ci/check-network-isolation.sh` as second layer (word-boundary grep, allow-list file with justification lines).
2. **Debug network canary**: in DEBUG builds install a `URLProtocol` registered for all schemes that asserts any request's host matches the model-download allow-list AND a `ModelDownloadService.activeDownloads` flag is set; violations `assertionFailure` with the offending URL (host only in message). Unit test: fake request outside allow-list triggers canary (test hooks, no real network).
3. **ATS**: confirm Info.plist has NO `NSAppTransportSecurity` exceptions; add test reading the built Info.plist.
4. **Dependency posture**: `dependabot.yml` (swift + github-actions ecosystems, weekly, PR-only); CI step verifying `Package.resolved` pins exact revisions and `frameworks.lock` SHAs match checked-out vendor builds (`--check`); document upgrade-review policy in `docs/qa/security-checklist.md` (changelog skim + diff review for engine bumps per DESIGN §14 T7).
5. **Logging audit**: script `scripts/ci/check-logging.sh` — grep all `Logger`/`os_log` call sites for identifiers `rawText|formattedText|consumeFor` co-occurrence; manual audit of every call site recorded (file:line list with ✓) in the checklist doc.
6. **Pasteboard hygiene verification**: manual procedure — run a clipboard manager (e.g., Maccy) during 10 paste-sim insertions: verify transient items don't appear; verify restore correctness with text+image clipboard preloads; record results.
7. **Runtime network capture evidence**: procedure + first execution — fresh launch, complete 5 dictations, capture with `nettop`/Little Snitch/`tcpdump` filter on app PID: zero connections; then one model download: only allow-listed hosts. Screenshots/log excerpts into `docs/qa/security-checklist.md` evidence section.
8. **PR template** `.github/pull_request_template.md`: checklist items (touched network code? → must be ModelCatalog + justify; logs reviewed for content; new deps? → ADR note + pin; entitlements unchanged?).
9. Re-verify entitlements/hardened-runtime settings against ADR-006 (test asserting entitlement file contents).

## Acceptance Criteria

- [ ] Lint rule: seeded violation in a scratch branch fails CI (link both runs); allow-listed ModelCatalog untouched.
- [ ] Canary test green; manual seeded-violation assertion screenshot.
- [ ] ATS/entitlements tests green.
- [ ] `check-network-isolation.sh` + `check-logging.sh` wired into CI and green.
- [ ] Security checklist doc committed with executed evidence for items 5–7 (dated, environment noted).
- [ ] Dependabot + PR template live.

## Validation

CI runs (green + seeded-red links), evidence sections in `docs/qa/security-checklist.md`, reviewer replays the network-capture procedure once.

## Dependencies

02, 05, 11, 19, 22 (real surfaces to audit).

## Non-goals

New product features; SQLCipher (v2); sandboxing (ADR-006 permanent); SBOM generation (v2 nice-to-have — note in checklist).

## Design References

DESIGN §14 (normative), §14.6, §15; ADR-004, ADR-006, ADR-007.
