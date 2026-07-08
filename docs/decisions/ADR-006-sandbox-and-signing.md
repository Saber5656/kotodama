# ADR-006: Unsandboxed app with Hardened Runtime, Developer ID + notarization; no App Store

- Status: Accepted (2026-07-08)

## Context

The product requires posting CGEvents to other apps and AX writes — both incompatible with App Sandbox. Distribution of an unsandboxed OSS app on modern macOS requires Developer ID signing + notarization to avoid Gatekeeper friction.

## Decision

1. **App Sandbox: OFF** (capability requirement). **Hardened Runtime: ON** with no exception entitlements (no JIT, no unsigned-memory, no library-validation disable — vendored XCFrameworks are signed as part of the app).
2. Distribution: **Developer ID signed + notarized + stapled DMG** via GitHub Releases (SHA256SUMS published), Homebrew cask after first stable release. **App Store: permanently out of scope** for this form factor.
3. Signing/notarization credentials are handled **manually by the maintainer** on a local machine (per user's global security rules); CI never holds signing secrets and produces unsigned artifacts only. A written runbook (issue 31) covers the manual steps.
4. TCC permissions used: Microphone, Accessibility. Nothing else (no Input Monitoring in v1, ADR/DESIGN §11).

## Consequences

- Users get standard double-click install with no Gatekeeper overrides; corporate MDM environments can allowlist by Team ID.
- Release requires maintainer's paid Apple Developer membership (documented prerequisite; release blocked on it — flagged in ISSUE_PLAN known unknowns if unavailable).
- Without sandbox, our security story leans on Hardened Runtime, minimal dependencies, the no-network architecture (ADR-004), and notarization's malware scan — reflected in the threat model (DESIGN §14).

## Alternatives considered

- **Sandboxed + App Store**: impossible for CGEvent/AX-based insertion. Rejected.
- **Unsigned releases ("right-click open")**: hostile UX, undermines trust for a privacy product. Rejected.
