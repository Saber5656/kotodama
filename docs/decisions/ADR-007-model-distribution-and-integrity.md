# ADR-007: Model distribution via bundled curated catalog; pinned revisions + SHA256; no remote catalog in v1

- Status: Accepted (2026-07-08)

## Context

Models (0.5–2.5 GB) cannot ship inside the app bundle; users must download them. Model files cross a trust boundary into C/C++ parsers with a known CVE history (GGML/GGUF loaders). We need integrity guarantees without introducing new network surfaces (ADR-004).

## Decision

1. The app bundles a **curated, versioned catalog** (`model-catalog.json`, schema in DESIGN §7.4). Only catalog models are downloadable/loadable in v1 (no arbitrary model import).
2. Every entry pins: an **immutable Hugging Face revision URL**, **SHA256**, size, license + URL, and **provenance** (who produced/quantized the artifact). Default catalog admits only Apache-2.0/MIT models from the model vendor or a pinned reputable quantizer.
3. Download pipeline: HTTPS + ATS, host allow-list (`huggingface.co` + its LFS CDN), disk preflight, streaming SHA256, verify-before-install, atomic rename, `meta.json` records the verified catalog snapshot. Checksum mismatch → delete + `downloadFailed(checksum)`; never load unverified files.
4. Integrity re-verification: on `modelLoadFailed` and on user request ("re-verify" in Models UI).
5. Catalog updates ship **only with app releases** in v1 (no remote catalog fetch). A remote catalog or user model import requires a new ADR.
6. SHA256 values are filled at implementation time from independently downloaded copies on two networks (issue 10 procedure), and reviewed in PR.

## Consequences

- Supply-chain exposure is reduced to: curators' review of pinned artifacts + HF revision immutability; MITM and silent-swap attacks are neutralized by pinning (DESIGN §14 T1/T8).
- Users cannot try arbitrary models in v1 (accepted; v2 feature with explicit warnings).
- Catalog staleness between releases is accepted (ADR-004 trade-off).

## Alternatives considered

- **Remote catalog JSON**: fresher models, but adds a phone-home-ish surface + signing complexity. Deferred (would need Ed25519-signed catalog + new ADR).
- **Bundling models in the DMG**: 3 GB installers, license redistribution questions. Rejected.
- **Checksum-less "latest" HF URLs**: mutable target, unacceptable for T1/T8. Rejected.
