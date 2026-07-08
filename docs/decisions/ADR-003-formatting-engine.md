# ADR-003: Formatting via vendored llama.cpp; default Qwen3-4B-Instruct-2507 Q4_K_M, Light preset sarashina2.2-3B

- Status: Accepted (2026-07-08)
- Deciders: user (local-LLM approach, 2026-07-07) + design agent (model selection)
- Informed by: [research/2026-07-formatting-llm.md](../research/2026-07-formatting-llm.md)

## Context

口語→整文/敬語 conversion needs register control and light rewriting — beyond rule-based morphology, within reach of ~3–4B instruct models. Budgets: ≤ 3.5 s per utterance warm, ≤ ~3 GB RAM loaded, OSS-clean licenses.

## Decision

1. Engine: **llama.cpp** vendored as XCFramework from a pinned tag (same pattern as ADR-002).
2. Default model: **Qwen3-4B-Instruct-2507 Q4_K_M** (Apache-2.0). **Light preset** (8 GB Macs): **sarashina2.2-3b-instruct-v0.1 Q4_K_M** (MIT). Both pinned by HF revision + SHA256; provenance recorded (prefer vendor-official GGUF, else pinned reputable quantizer).
3. Inference: temperature 0, model chat template, `n_ctx=4096`, output cap 1024 tokens; deterministic-by-default.
4. Formatting failures **never block dictation**: timeout/validation failure falls back to the raw transcript (DESIGN §9.3).
5. Final model choice is gated by the offline eval harness (DESIGN §9.4); swapping models is a catalog-only change.
6. Default catalog restricted to Apache-2.0/MIT models; bespoke-license models (Gemma etc.) stay out of the default catalog.

## Consequences

- +2–2.5 GB model download and a keep-warm/idle-unload memory policy (DESIGN §13).
- Keigo quality risk is contained by the eval gate + documented fallback order (sarashina → llm-jp-3).
- Rule-based fallback is not built in v1 (protocol allows adding one later without pipeline changes).

## Alternatives considered

- **Rule-based (Sudachi/MeCab)**: deterministic and tiny but cannot produce natural keigo register shifts. Rejected as primary; possible v2 "instant mode".
- **Hybrid rules+LLM**: two systems to maintain for marginal gain (LLM already handles cleanup). Rejected for v1.
- **Gemma-3-4B-it**: competitive quality, bespoke license complicates an OSS default catalog. Excluded from default catalog.
