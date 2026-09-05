# Title

OSS docs: README, PRIVACY, SECURITY, CONTRIBUTING, LICENSE, attributions

## Summary

Write the public-facing repository documents for the OSS release: bilingual README, PRIVACY.md with the verifiable no-network guarantee, SECURITY.md reporting policy, CONTRIBUTING.md, LICENSE (MIT — pending user confirmation), third-party attributions, and the app's About-window links.

## Context

DoD item 13; DESIGN §15 (privacy contract wording) and §14.6. LICENSE choice is DESIGN §21 U1 — **blocked on explicit user confirmation** (MIT proposed); everything else proceeds.

## Scope

Repo root docs + `docs/` links + About-window link wiring (trivial App change) + app icon final asset if available (else tracked follow-up).

## Detailed Requirements

1. **README.md** (JA primary + EN section, or README.md/README.en.md pair — choose single bilingual file, JA first): what/why (口語→整文/敬語 × 完全ローカル), 30-sec GIF placeholder slot, feature list, requirements (macOS 14+, Apple Silicon, RAM guidance incl. Light preset), install (DMG/GitHub Releases; cask when live), first-run walkthrough pointer, hotkey defaults, model table (sizes/licenses from catalog), privacy summary box linking PRIVACY.md, known quirks (IME/Electron paste from 19's matrix), build-from-source (BUILD.md link), license/credits.
2. **PRIVACY.md**: the §15 statement verbatim as the contract; data inventory table (audio: RAM-only; transcripts: local DB w/ controls; settings; models); network behavior (downloads only, hosts listed); **verification section**: exact steps a user can run (network monitor during dictation = zero connections; grep pointers into source: ModelCatalog is the only network module; lint rule reference); clipboard behavior disclosure (transient marking + residual-risk note re non-conforming managers); log policy.
3. **SECURITY.md**: supported versions (latest release), private reporting channel (GitHub Security Advisories), response expectation (best-effort OSS wording), scope notes (model files trust boundary, ADR-007), hall-of-fame section stub.
4. **CONTRIBUTING.md**: dev setup (BUILD.md), branch/PR conventions (PRs to main, CI green, PR-template security checklist), docs-are-canonical rule (DESIGN/ISSUE_PLAN drive work; behavior changes update DESIGN in same PR), test expectations per module, no-network architectural rule (link ADR-004), DCO or simple sign-off decision (default: none, just license agreement note — document).
5. **LICENSE**: MIT, copyright holder = user's name/handle (confirm exact attribution line with user alongside U1 approval). **Do not merge this file without recorded user confirmation** (PR review note).
6. **THIRD_PARTY_NOTICES.md**: table of runtime deps (whisper.cpp MIT, llama.cpp MIT, KeyboardShortcuts MIT, GRDB MIT) with license texts/links; **model attribution section** (kotoba-whisper Apache-2.0, Whisper/turbo MIT, Qwen3 Apache-2.0, sarashina MIT) with required notices; clarify models are downloaded by users, not distributed with the app. Note VoiceInk explicitly NOT used as code source (clean-room statement, GPL hygiene).
7. About window: version, repo link, PRIVACY link, THIRD_PARTY_NOTICES link (wired).
8. Repo hygiene: description/topics suggestions for GitHub (ja, macos, dictation, whisper, local-llm…), social preview text — listed for the user (repo settings are user-executed or via existing hardening skill, out of this issue's scope to apply).

## Acceptance Criteria

- [ ] All six documents committed; README renders correctly on GitHub (JA/EN anchors work, tables valid).
- [ ] PRIVACY verification steps executed once as written (evidence link to 27's capture) — no aspirational claims.
- [ ] LICENSE merged only with user-confirmation note (U1) recorded in PR thread.
- [ ] Attribution completeness cross-checked against Package.resolved + catalog (script or manual table diff in PR).
- [ ] About links open correct pages.
- [ ] EN sections read naturally (second-reader pass noted).

## Validation

Rendered-README screenshot, attribution cross-check evidence, U1 confirmation link, About-window recording in PR.

## Dependencies

All prior issues for accurate content (hard blockers: 27 evidence, 19 quirks, 10 model table, 31 install instructions); U1 user decision.

## Non-goals

Website/landing page, App Store metadata (n/a), localized README beyond JA/EN, marketing assets (GIF can be placeholder with follow-up issue), GitHub repo settings application (user-executed).

## Design References

DESIGN §15 (normative wording), §14.6, §3.1 DoD 13, §21 U1/U2; ADR-004; ADR-007; research (VoiceInk GPL note).
