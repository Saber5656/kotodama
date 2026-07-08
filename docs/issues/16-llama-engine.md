# Title

LlamaEngine: llama.cpp wrapper actor

## Summary

Implement `LlamaEngine` (actor) wrapping the vendored llama.xcframework: GGUF load/unload, chat-template-based deterministic generation (temperature 0), token caps, stop handling, cancellation, and timing/memory reporting.

## Context

DESIGN §9.2 fixes inference parameters; ADR-003 fixes engine and defaults. FormattingService (17) supplies prompts and consumes plain strings; this issue is the only C-interop point for llama.cpp.

## Scope

`Sources/FormattingService/Llama/` + tests. Verify exact llama.cpp symbols against the pinned tag (API drifts; typical current shape: `llama_model_load_from_file`, `llama_init_from_model`, batch decode loop, `llama_chat_apply_template`).

## Detailed Requirements

1. Protocol `FormattingEngine` (define in `Sources/FormattingService/FormattingEngine.swift`):
   - `func load(modelPath: URL) async throws`
   - `func unload() async`
   - `func generate(_ request: GenerationRequest) async throws -> GenerationResult`
   - `func cancelCurrent()`
   - `var isLoaded: Bool { get async }`
   `GenerationRequest { system: String; user: String; maxTokens: Int; timeout: Duration }`, `GenerationResult { text: String; inputTokens: Int; outputTokens: Int; durationMs: Int }`.
2. Params (DESIGN §9.2, pinned): temperature 0.0, top_p 1.0, repeat-penalty engine-default, `n_ctx = 4096`, `n_batch` sensible for Metal (verify tag default), seed fixed (0) — determinism note: temp-0 + fixed seed on same build/model = reproducible (document caveat: Metal nondeterminism possible; eval harness tolerates via property checks).
3. Chat template: use the model's embedded template via `llama_chat_apply_template` (system + single user message). If a catalog model lacks a template → `formattingFailed` at load with log (all catalog models must have one; test asserts we detect absence).
4. Generation loop: token-by-token decode; stop on EOS/EOT from template; hard stop at `maxTokens`; cooperative cancellation checked each token; timeout enforced by caller (17) but engine also aborts if `request.timeout` elapses internally (belt-and-braces).
5. Memory: report `modelSizeBytes` + context memory estimate via engine API where available; free everything on unload (assert with repeated load/unload cycles in integration test — RSS delta < 200 MB after 3 cycles).
6. Threading: engine calls on actor-isolated executor; Metal offload enabled; threads cap 8.
7. Logging: `KotodamaLog.llm` — model id, token counts, durations only (prompts/outputs are content — NEVER logged; `SensitiveString` used at the 17 boundary).
8. Tests: unit (state misuse, cancellation flag, template-absence detection with a headerless fixture if craftable — else document); integration gated `KOTODAMA_IT=1` with the **smallest catalog LLM** (or a tiny public GGUF pinned in test config ≤ 300 MB — e.g. a Qwen 0.5B-class instruct; add to a test-only pinned list with sha256, NOT the product catalog): prompt "「えーと、あしたの、あした10時にいきます」を丁寧な書き言葉に直してください" → non-empty deterministic output across 2 runs (byte-equal), token caps respected, cancel returns < 300 ms.

## Acceptance Criteria

- [ ] Unit tests green in CI; integration green locally (transcript + timing + RSS table in PR).
- [ ] Two consecutive identical requests produce byte-identical outputs (determinism evidence).
- [ ] Load/unload ×3 RSS evidence within bounds.
- [ ] Grep evidence: llama symbols confined to `Sources/FormattingService/Llama/`; no content logging.

## Validation

CI units + local `KOTODAMA_IT=1` transcript in PR.

## Dependencies

06, 12.

## Non-goals

Prompt templates/styles/validation (17), model quality (18), keep-warm policy (17 service layer), sampling features beyond temp-0 (never in v1).

## Design References

DESIGN §9.2, §13, §14 T9; ADR-003.
