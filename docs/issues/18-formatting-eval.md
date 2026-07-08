# Title

kotodama-eval: formatting quality harness + release gate

## Summary

Build the offline evaluation harness for formatting quality: ≥ 30 JA fixture transcripts with references, property-based checks (filler absence, です・ます consistency, length ratio, no preamble), a CLI that runs the real FormattingService against real models, and a written report deciding whether default models pass the release gate.

## Context

DESIGN §9.4 defines the gate: seibun ≥ 90 % property-pass, keigo ≥ 80 % + human spot-check. ADR-003 makes model choice eval-gated and catalog-swappable; DESIGN §21 U4 (keigo quality) is resolved by this issue.

## Scope

`eval/` directory: fixtures, checker library (SPM target `KotodamaEval` in KotodamaKit or a separate local package — choose separate `eval/Package.swift` to keep prod deps clean), CLI `kotodama-eval`, report doc.

## Detailed Requirements

1. Fixtures `eval/fixtures/transcripts.json`: ≥ 30 entries `{ id, rawText, notes, tags[] }` covering: heavy fillers; self-corrections (「明日、じゃなくて明後日」); casual contractions (〜しちゃって、〜っす); numbers/dates/amounts; person/company names; long rambling (300+ chars); short one-liners; requests/refusals (keigo-sensitive); mixed EN loanwords; already-polite input (no-op expectation). Each entry has `references: { seibun: String, keigo: String }` written by hand (JA-native-quality; author them carefully in the PR — reviewers check 10 random).
2. Property checkers (pure functions, unit-tested themselves):
   - `noFillers`: none of the filler lexicon (constant list, shared with 17's prompt doc) appear as standalone tokens.
   - `desuMasuConsistent`: sentence terminals (。！？ splits) end in です/ます/でした/ましょう/ません etc. form-class ≥ 95 % of sentences (regex + exceptions list: 体言止め allowed ≤ 1 per output).
   - `lengthRatio`: chars ratio vs raw ∈ [0.3, 3.0] (mirror of validator, sanity).
   - `noPreamble`: no lines matching preamble patterns; output ≠ starts with 「以下」「変換後」.
   - `contentPreserved` (informative, not gating): character-level similarity (e.g., normalized Levenshtein vs reference ≥ threshold logged; numbers/names from raw all present in output — gating for the numbers/names subset).
   - keigo-only: `keigoMarkers` — presence of at least one 謙譲/尊敬 marker (いたします、いただく、ご〜、お〜、〜れます class) when tags include `keigo-sensitive`.
3. CLI `kotodama-eval run --style seibun|keigo|both --model <catalogId> --out eval/reports/<date>-<model>.json`: loads FormattingService with the real engine, runs all fixtures, evaluates properties, emits JSON + markdown summary table (per-fixture pass/fail per property + aggregate rates + timings). Deterministic (temp-0); 2-run reproducibility check flag `--verify-determinism`.
4. Gate script `kotodama-eval gate <report.json>`: exit 0 iff seibun ≥ 90 % all-properties-pass rate AND keigo ≥ 80 % — used manually and by release checklist (31).
5. Human spot-check protocol in `eval/README.md`: 10 keigo outputs sampled by seeded RNG; grader marks natural/unnatural/wrong-meaning; ≥ 8 natural required; record table in report.
6. Execute the eval for **both** catalog LLMs (Qwen3-4B, sarashina 3B) on a dev machine; write `docs/qa/formatting-eval-report.md` summarizing rates/timings and the pass/fail verdict; if the default fails, file the follow-up model-swap issue per ISSUE_PLAN §8 (do not silently change defaults).

## Acceptance Criteria

- [ ] Checkers unit-tested with hand-built positive/negative snippets (≥ 3 each).
- [ ] `kotodama-eval run` completes on both models locally; JSON+MD reports committed under `eval/reports/`.
- [ ] Determinism verified (2 identical runs) for the default model.
- [ ] `docs/qa/formatting-eval-report.md` states gate verdicts + spot-check table; DESIGN §21 U4 updated (closed or follow-up filed).
- [ ] Harness runs without network (models must be pre-installed; clear error otherwise).

## Validation

Committed reports + reviewer replication instructions (exact commands) executed once by a second party (or documented single-party with logs if none available).

## Dependencies

17 (16, 12 transitively).

## Non-goals

ASR eval (14's A/B covers it), CI-blocking eval runs (manual/release-gate only), automatic prompt tuning (prompt changes are ordinary PRs bumping promptVersion + rerun), training/fine-tuning.

## Design References

DESIGN §9.4 (gate — normative), §21 U4; ADR-003; research/2026-07-formatting-llm.md.
