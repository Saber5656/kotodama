# kotodama — Issue Plan (v1)

- Status: accepted 2026-07-08; derived from [DESIGN.md](DESIGN.md) and ADR-001…007
- GitHub Issues are generated from [docs/issues/](issues/) and are derived artifacts of this plan.

## 1. v1 completion statement

When every issue 01–32 below is completed and validated (per each issue's Validation section plus §6 here), kotodama v1 as defined in DESIGN.md §3.1 is complete: a signed/notarized, strictly-local, Japanese-first menu-bar dictation app for Apple Silicon macOS 14+, with whisper.cpp ASR, llama.cpp 整文/敬語 formatting, safe system-wide insertion, history, onboarding, JA/EN localization, and published OSS docs — with no product behavior left outside this plan except the known unknowns in §8.

## 2. Issue list (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | issues/01-project-scaffold.md | Project scaffold: XcodeGen app skeleton, KotodamaKit package, lint config | 0 |
| 02 | issues/02-ci-workflow.md | CI: build, lint, and unit-test workflow on macOS arm64 | 0 |
| 03 | issues/03-core-types-state-machine.md | CoreTypes: domain models, error taxonomy, dictation state machine | 0 |
| 04 | issues/04-settings-store.md | SettingsStore: typed UserDefaults-backed settings | 0 |
| 05 | issues/05-diagnostics-logging.md | Diagnostics: content-free logging, SensitiveString, diagnostics report | 0 |
| 06 | issues/06-engine-xcframeworks.md | build-frameworks.sh: vendored whisper.cpp / llama.cpp XCFrameworks | 0 |
| 07 | issues/07-permissions-service.md | PermissionsService: microphone + Accessibility status/request | 1 |
| 08 | issues/08-audio-capture.md | AudioCapture: 16 kHz mono capture with level metering | 1 |
| 09 | issues/09-hotkey-service.md | HotkeyService: global hold/toggle hotkey via Carbon | 1 |
| 10 | issues/10-model-catalog.md | ModelCatalog: bundled catalog schema, parsing, initial entries | 2 |
| 11 | issues/11-model-download.md | ModelDownloadService: verified, resumable model downloads | 2 |
| 12 | issues/12-model-store.md | ModelStore: on-disk model layout, scan, delete, re-verify | 2 |
| 13 | issues/13-models-ui.md | Models UI: download/manage models window | 2 |
| 14 | issues/14-whisper-engine.md | WhisperASREngine: whisper.cpp wrapper actor | 3 |
| 15 | issues/15-asr-service.md | ASRService: model resolution, warm-up, lifecycle | 3 |
| 16 | issues/16-llama-engine.md | LlamaEngine: llama.cpp wrapper actor | 4 |
| 17 | issues/17-formatting-service.md | FormattingService: styles, prompts, validation, fallback | 4 |
| 18 | issues/18-formatting-eval.md | kotodama-eval: formatting quality harness + release gate | 4 |
| 19 | issues/19-insertion-service.md | InsertionService: paste-sim / AX / clipboard chain with refusals | 5 |
| 20 | issues/20-hud-overlay.md | HUD overlay: floating pipeline-state panel | 5 |
| 21 | issues/21-menu-bar-ui.md | Menu bar: status item, menu, style switcher | 5 |
| 22 | issues/22-dictation-controller.md | DictationController: end-to-end pipeline state machine | 5 |
| 23 | issues/23-history-store.md | HistoryStore: SQLite sessions, retention, delete-all | 6 |
| 24 | issues/24-history-ui.md | History UI: browse, search, copy, delete | 6 |
| 25 | issues/25-settings-ui.md | Settings UI: full preferences window | 6 |
| 26 | issues/26-onboarding.md | Onboarding: permissions + model-download walkthrough | 6 |
| 27 | issues/27-security-hardening.md | Security enforcement: network isolation, pasteboard/logging audits | 7 |
| 28 | issues/28-performance-memory.md | Performance: benchmark harness, memory policy, budgets | 7 |
| 29 | issues/29-error-ux.md | Error UX: message mapping, recovery actions, resilience paths | 7 |
| 30 | issues/30-localization.md | Localization: JA/EN String Catalog completeness | 7 |
| 31 | issues/31-release-engineering.md | Release: DMG script, sign/notarize runbook, release checklist | 7 |
| 32 | issues/32-oss-docs.md | OSS docs: README, PRIVACY, SECURITY, CONTRIBUTING, LICENSE, attributions | 7 |

## 3. Dependency table

| Issue | Blocked by | Enables |
|---|---|---|
| 01 | — | all |
| 02 | 01 | 06, 27, 31 |
| 03 | 01 | 04, 05, 07–12, 19–23 |
| 04 | 03 | 09, 15, 17, 21, 25, 26 |
| 05 | 03 | 08, 11, 22, 27 |
| 06 | 01, 02 | 14, 16 |
| 07 | 03 | 19, 22, 25, 26 |
| 08 | 03, 05 | 14 (audio contract), 22 |
| 09 | 03, 04 | 22, 25 |
| 10 | 03 | 11, 12, 13, 14, 16 |
| 11 | 05, 10 | 12, 13, 27 |
| 12 | 10, 11 | 13, 14, 15, 16 |
| 13 | 12 | 26 |
| 14 | 06, 08, 12 | 15 |
| 15 | 04, 14 | 22, 28 |
| 16 | 06, 12 | 17 |
| 17 | 04, 16 | 18, 22, 28 |
| 18 | 17 | release gate |
| 19 | 03, 07 | 22, 27 |
| 20 | 01, 03 | 22 |
| 21 | 03, 04 | 25 (entry points), 30 |
| 22 | 05, 07, 08, 09, 15, 17, 19, 20, 23 | 24 (data), 27, 28, 29 |
| 23 | 03 | 22, 24 |
| 24 | 23 | 30 |
| 25 | 04, 07, 09, 13, 21 | 26, 30 |
| 26 | 04, 07, 13 | 30 |
| 27 | 02, 05, 11, 19, 22 | release gate |
| 28 | 15, 17, 22 | release gate |
| 29 | 22 | release gate |
| 30 | 20, 21, 24, 25, 26 | release gate |
| 31 | 01, 02 | first release |
| 32 | all (final content pass) | first release |

Within a wave, issues are parallelizable unless the table says otherwise. 22 is the integration choke point — schedule it as soon as 15/17/19/20/23 land.

## 4. Implementation waves

| Wave | Theme | Issues | Exit criterion |
|---|---|---|---|
| 0 | Foundation | 01–06 | CI green on skeleton app; frameworks build reproducibly |
| 1 | Input & permissions | 07–09 | Hotkey triggers capture with permissions handled (log-level demo) |
| 2 | Model pipeline | 10–13 | Real models downloadable, verified, manageable in UI |
| 3 | ASR | 14–15 | Fixture WAV → correct JA transcript in integration test |
| 4 | Formatting | 16–18 | Eval harness passes seibun ≥ 90 % / keigo ≥ 80 % gates |
| 5 | Core loop | 19–22 | Hold-speak-release inserts formatted text into TextEdit end-to-end |
| 6 | Product surfaces | 23–26 | Fresh-install onboarding to working dictation without terminal usage |
| 7 | Hardening & release | 27–32 | DESIGN §3.1 DoD fully satisfied; signed DMG produced via runbook |

## 5. Coverage: DESIGN.md § → issues

| DESIGN section | Issues |
|---|---|
| §4.1 Recording interaction | 09, 22 |
| §4.2 HUD | 20 |
| §4.3 Menu bar | 21 |
| §4.4 Onboarding | 26 |
| §4.5 Error UX | 29 |
| §5 Architecture & layout | 01, 06 |
| §6.1–6.3 State machine & concurrency | 03, 22 |
| §6.4 Error taxonomy | 03, 29 |
| §7.1 Storage layout | 12, 23 |
| §7.2 History schema | 23, 24 |
| §7.3 Settings keys | 04, 25 |
| §7.4 Model catalog | 10 |
| §8 ASR | 14, 15 |
| §9 Formatting | 16, 17, 18 |
| §10 Insertion | 19 |
| §11 Hotkey & permissions | 07, 09 |
| §12 Model management | 10, 11, 12, 13 |
| §13 Performance & memory | 28 (budgets asserted), 15/17 (lifecycle hooks) |
| §14 Security model | 27 (enforcement), plus embedded criteria in 11, 19, 23 |
| §15 Privacy posture | 27, 32 |
| §16 Packaging & release | 31 |
| §17 Testing strategy | every issue's Validation + 18, 28 |
| §18 Dependencies | 01 |
| §19 Localization | 30 |
| §20 Diagnostics | 05 |
| §21 Open decisions | §8 below |

## 6. Whole-product validation strategy

1. **PR gate (every issue)**: CI build + SwiftLint (incl. network-ban rule once 27 lands) + unit tests; issue-specific Validation section satisfied with evidence in the PR body.
2. **Integration gates**: wave exit criteria above; engine integration tests (`KOTODAMA_IT=1`) run at wave 3/4 exits and before release.
3. **Release gate (v1)**: eval harness thresholds (18); performance budgets on M1/8 GB (28); manual insertion matrix fully executed (19's doc); security checklist + network-capture evidence (27); onboarding fresh-VM run (26); sign/notarize/staple + Gatekeeper check (31).
4. **Traceability**: every issue cites the DESIGN sections it implements; deviations discovered during implementation must update DESIGN.md in the same PR (docs are canonical).

## 7. Deferred to v2 (not planned here)

Apple SpeechAnalyzer engine option · Sparkle opt-in auto-update · per-app style profiles · custom user styles/prompts · Fn/Globe hotkey (Input Monitoring) · streaming partial transcripts · vocabulary boosting · voice commands · model import with warnings · SQLCipher history encryption · unicode-typing insertion mode · remote signed catalog · CLI companion · Intel build.

## 8. Known unknowns that may spawn new issues

Tracked in DESIGN §21 (U1–U10). Most likely to create follow-up issues during implementation:

- U3 whisper.cpp decode-quality tuning for kotoba (may need a dedicated decode-params issue after 14's A/B).
- U4 keigo quality below gate → model swap + prompt iteration issue (catalog change + 18 rerun).
- U5 XCFramework build friction on CI runners (fallback: static-lib linking issue).
- U7 Electron/IME paste failures → per-app quirk handling issue if matrix failure rate is material.
- U8 8 GB memory pressure → constrain keepWarm options issue.
- U1/U2 license & naming confirmation from the user (blocking 32 and release, respectively).
- Apple Developer ID availability for notarization (blocks 31's final validation, not its scripting).
