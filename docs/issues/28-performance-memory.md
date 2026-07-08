# Title

Performance: benchmark harness, memory policy verification, budgets

## Summary

Build `kotodama-bench` (pipeline latency harness over fixture audio with real models), measure against DESIGN §13 budgets on reference hardware, verify keep-warm/idle-unload/memory-pressure behavior empirically, and update DESIGN §13 with measured tables (or file corrective issues).

## Context

DESIGN §13 budgets are acceptance thresholds for DoD item 11. Services already implement lifecycle hooks (15/17); this issue proves them and produces the numbers.

## Scope

`eval/` bench CLI target (`kotodama-bench`), `docs/qa/performance-report.md`, small instrumentation additions only (no behavior redesign — deviations become follow-up issues).

## Detailed Requirements

1. `kotodama-bench run --preset light|recommended --iterations 10`: feeds fixture WAVs (3 s / 10 s / 30 s JA speech from 14's fixtures) through AudioCaptureCore-conversion → ASRService → FormattingService (seibun + keigo) with real models; reports per-stage p50/p95 (arm excluded — measured separately), cold vs warm variants, peak RSS per stage (via `proc_pid_rusage`/`task_info` helper).
2. Hotkey→recording-start measurement: instrumented debug path in the app (log-mark at hotkeyDown and first audio buffer) — manual 10-sample collection procedure documented + executed.
3. Memory-policy verification scenarios (scripted where possible, else manual with `footprint`/Activity Monitor evidence):
   - idle no models < 150 MB; asrOnly warm delta ≈ documented; LLM idle-unload at 5 min actually frees (RSS drop ≥ 80 % of model size); memory-pressure simulation (`sudo memory_pressure -l critical` or documented alternative) unloads per §13; keepWarm=both on 8 GB machine — record swap/pressure behavior → recommend whether to gate the option by RAM (decision recorded, DESIGN §21 U8 closed or issue filed).
4. Reference environments: REQUIRED M1/8 GB (or closest available — document actual hardware; if none available, mark budgets provisional and file follow-up) + one 16 GB+ machine.
5. `docs/qa/performance-report.md`: tables per §13 layout, environment details, model versions, pass/fail per budget row; DESIGN §13 updated with "measured 2026-MM" column in same PR.
6. Bench determinism: fixed fixtures, ≥ 10 iterations, discard first (cold measured separately), report medians; no network.

## Acceptance Criteria

- [ ] Bench CLI runs reproducibly (two runs within 15 % on all p50s — table in PR).
- [ ] All §13 budget rows measured on the 8 GB-class machine with Light preset; failures (if any) have filed follow-up issues linked in the report.
- [ ] Memory scenarios evidenced (before/after RSS numbers); idle-unload + pressure behavior confirmed.
- [ ] DESIGN §13 updated with measured values; report committed.

## Validation

Committed report + reviewer reruns `kotodama-bench` on any Apple Silicon machine and gets sane numbers (instructions in eval/README).

## Dependencies

15, 17, 22 (fixtures from 14).

## Non-goals

Optimization work itself (follow-ups), battery instrumentation (spot-check note only), CI perf gates (manual/release-gate only), UI rendering perf.

## Design References

DESIGN §13 (normative budgets), §3.1 DoD 11, §21 U8; ADR-003.
