# ADR-004: Strictly local; the sole permitted network use is explicit model download

- Status: Accepted (2026-07-08)
- Deciders: user (2026-07-07)

## Context

kotodama's differentiation and README promise is "ローカル音声入力". Dictated content is maximally sensitive (it is the user's unfiltered speech, potentially in privacy-critical workplaces). Cloud formatting would raise quality ceilings but expands the threat surface (keys, egress consent, logging) and dilutes the core promise.

## Decision

1. Runtime rule: **no network I/O except `ModelDownloadService`**, which runs only on explicit user action (download button / onboarding confirmation), HTTPS-only, against allow-listed hosts (`huggingface.co`, its LFS CDN), with pinned revisions + SHA256 (ADR-007).
2. **No telemetry, no crash reporting, no analytics, no update phone-home** in v1. Update checking is manual (menu item opens the GitHub Releases page in the browser).
3. Enforcement is architectural, not aspirational (DESIGN §14.6): single network module; SwiftLint rule banning network imports elsewhere; debug-build network canary; PRIVACY.md documents user-side verification.
4. Any future feature requiring network (remote catalog, auto-update, opt-in cloud) requires a new ADR + explicit user approval, and must ship default-off.

## Consequences

- Strong, verifiable privacy claim usable in README/PRIVACY; minimal threat model.
- We forgo crash telemetry (diagnosis relies on user-initiated "Copy diagnostics") and automatic updates (mitigated by Homebrew cask + manual check).
- Model catalog updates ship only with app releases (accepted staleness).

## Alternatives considered

- **Local-first + opt-in cloud formatting**: raises quality ceiling; rejected by user for v1 (posture clarity outweighs).
- **Auto-update via Sparkle from day one**: valuable but is network phone-home; deferred to v2 as opt-in with its own ADR.
