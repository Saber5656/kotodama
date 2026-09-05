# Title

Error UX: message mapping, recovery actions, resilience paths

## Summary

Implement the complete user-facing error layer: JA/EN message + recovery-action mapping for every `KotodamaError` case, HUD/menu/settings surfacing rules, model-corruption re-download flow, and first-failure diagnostics prompt — closing every error row of the state machine with polished UX.

## Context

DESIGN §4.5 (rules) + §6.4 (taxonomy with messageKey/recovery already typed in 03). Services emit errors; 22 routes them; this issue writes the copy, wires recovery actions end-to-end, and hardens multi-failure paths.

## Scope

`App/Errors/` (recovery router), string catalog entries for all `error.*` keys, small wiring in 20/21/25 surfaces, `docs/qa/error-catalog.md`.

## Detailed Requirements

1. **Copy table**: for every `KotodamaError` case (enumerate `messageKey` snapshot from 03): JA + EN strings — short HUD line (≤ 45 chars JA) + longer settings/history detail line. Tone: plain, non-technical, action-first (e.g., `error.axPermissionMissing` HUD: "自動貼り付けにはアクセシビリティ許可が必要です"). Committed as `docs/qa/error-catalog.md` (source of truth for reviewers) + string catalog entries.
2. **Recovery router**: `RecoveryAction` → concrete behavior: `openSettings(pane:)` → correct Settings pane or System Settings deep link (mic/AX); `retry` → controller retry entry (re-run last failed stage where safe: download retry, insert retry via re-insert; ASR/format retry = re-dictate message instead — document per case); `downloadModel` → Models window scrolled to relevant entry; `copyInstead` → persistent clipboard write of pending text (available on insertion failures — pending text retained by controller for 60 s for this purpose; add to 22 via small PR-scoped extension).
3. **Surfacing rules** (per §4.5): HUD = transient errors during pipeline; menu icon error badge 4 s; Settings permissions rows show persistent states; Models window shows download/corruption errors inline; History detail shows per-session failure.
4. **Model corruption flow**: `modelLoadFailed` → auto `reverify` (15 already triggers) → if corrupt: HUD error with `downloadModel` recovery → Models window shows 破損 chip → one-click再ダウンロード (delete quarantine + download) — walk the full path manually with a deliberately corrupted file (truncate installed model copy) and record it.
5. **First-failure diagnostics prompt**: on the first `failed` session per launch, HUD notice offers "診断情報をコピー" (one-shot, non-nagging; setting-free).
6. **Multi-failure resilience checks** (manual chaos list, executed + recorded): model dir deleted mid-session; mic device unplugged during recording (08's deviceDisconnected → clean error, no hang); AX revoked mid-pipeline (System Settings toggle during formatting → clipboard fallback path); disk full during download; force-quit during each pipeline stage → relaunch clean (no partial state, no corrupt DB).
7. Copy review: native-JA review pass over all strings (user review requested in PR — flag).

## Acceptance Criteria

- [ ] Every taxonomy case has JA+EN HUD+detail strings (drift test: messageKey snapshot ↔ catalog keys).
- [ ] Recovery router unit tests per action incl. pane targeting; pending-text copyInstead 60 s window tested.
- [ ] Corruption re-download flow recorded end-to-end.
- [ ] Chaos list executed with results table (all recover; no crash, no stuck non-idle state > timeout).
- [ ] Error catalog doc committed; strings flagged for native review.

## Validation

CI drift/router tests + chaos results table + corruption-flow recording in PR.

## Dependencies

22 (20/21/25 surfaces; 13 for Models inline states).

## Non-goals

New error cases without DESIGN update; crash reporting (never); localization QA sweep (30); copy for non-error UI (existing issues).

## Design References

DESIGN §4.5 (normative), §6.4, §12 (re-verify), §14.5; ADR-007 §4.
