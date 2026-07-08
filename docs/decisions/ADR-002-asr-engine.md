# ADR-002: ASR via vendored whisper.cpp; default model kotoba-whisper-v2.0 q5_0

- Status: Accepted (2026-07-08)
- Informed by: [research/2026-07-asr-engines.md](../research/2026-07-asr-engines.md)

## Context

We need fully-local Japanese ASR on Apple Silicon with ≤ ~1.5 s latency for 10 s utterances, permissive licensing, and macOS 14+ support. Candidates: whisper.cpp (+ kotoba-whisper or large-v3-turbo), Apple SpeechAnalyzer (macOS 26+), ReazonSpeech/NeMo stacks, Parakeet/FluidAudio.

## Decision

1. v1 engine: **whisper.cpp**, vendored as an XCFramework built from a **pinned upstream tag** by `scripts/build-frameworks.sh` (no stale `whisper.spm`, no prebuilt third-party binaries).
2. Default model: **kotoba-whisper-v2.0 q5_0** (Apache-2.0, JA-specialized, ~0.5 GB). Catalog alternatives: kotoba f16, large-v3-turbo q5_0/q8_0 (MIT) for JA/EN mixing.
3. `ASREngine` is a protocol; decode params (`language=ja`, `no_context=true`, timestamps off, temperature-fallback) are pinned in code after issue-14 A/B on fixture audio.
4. Apple SpeechAnalyzer is deferred to v2 as an optional second engine behind the same protocol.

## Consequences

- We own model download/integrity UX (ADR-007) and an engine-vendoring script, in exchange for model choice, JA-specialized quality, and macOS 14 reach.
- Distil-decoder quirks (repetition on silence) must be validated on fixtures before release (DESIGN §21 U3).
- Engine upgrades are deliberate PRs bumping the pinned tag (supply-chain review point, DESIGN §14 T7).

## Alternatives considered

- **SpeechAnalyzer only**: zero download and very fast, but macOS 26+, no model control, unverified casual-JA quality. Deferred.
- **ReazonSpeech NeMo/k2**: strong JA research models, no maintained mac-native inference path. Rejected.
- **Parakeet (FluidAudio)**: JA support immature. Rejected.
