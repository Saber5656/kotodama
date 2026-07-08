# Title

Models UI: download/manage models window

## Summary

Build the Models window (SwiftUI): list installed/available models from the catalog with size/license/provenance, download with progress + cancel, delete, re-verify, storage usage, and a RAM-based preset suggestion banner. View-model fully unit-tested against fakes.

## Context

DESIGN §12 (UI bullet) and §4.4 step 4 reuse this surface (onboarding embeds the preset picker). Users must see license/provenance before download (OSS transparency; ADR-007).

## Scope

`App/Models/` (views + view-model), strings in `Localizable.xcstrings` (JA source + EN). View-model in `App` target but engine/services accessed via protocols from KotodamaKit TestSupport fakes for tests.

## Detailed Requirements

1. `ModelsViewModel (@MainActor, @Observable)` deps: `CatalogProviding`, `ModelStoring`, `DownloadServicing`, `MemoryInfoProviding` (all protocols exist from 10–12/04; add thin protocols there if missing — same-PR change with those modules' owners' review).
2. Window layout: two sections (音声認識モデル / 整形モデル). Each row: displayName, size (GB, 1 decimal), license badge linking licenseURL, provenance tooltip, state chip: 未ダウンロード / ダウンロード中 (progress %, cancel) / 検証中 / インストール済 (verifiedAt) / 破損 (redownload + remove).
3. Actions: download (disabled when insufficient disk — show required vs free), cancel, delete (confirm dialog; blocked-in-use message when store refuses), re-verify (spinner → result toast), reveal quarantined/orphan items with sizes + delete option.
4. Preset banner: on 8 GB machines when selected LLM is the 4B model → suggest Light preset (one-click switches `format.llmModelId` + optionally downloads). Text per DESIGN §13/§7.3.
5. Footer: total storage usage; "モデルの保存先を開く" (opens models dir in Finder).
6. Default-model indicators: chips marking current `asr.modelId` / `format.llmModelId`; clicking "使用する" on an installed model updates SettingsStore.
7. Empty/error states: no models installed → onboarding-style call-to-action; download error rows show `KotodamaError` mapped message + retry.
8. Accessibility: all controls labeled (VoiceOver pass on the window).

## Acceptance Criteria

- [ ] View-model tests (fakes): list composition (installed/available/corrupt/orphan), progress event → state chip mapping, disk-insufficient gating, preset switch writes settings, in-use delete refusal surfaces message, cancel mid-download returns row to resumable state.
- [ ] Manual: download smallest ASR model end-to-end from the window (progress, verify, installed chip) — screen recording in PR.
- [ ] All strings via catalog (lint no-hardcoded-literals passes); JA+EN present.
- [ ] VoiceOver labels verified on one full row (checklist in PR).

## Validation

CI view-model tests + manual recording + screenshots (both sections, all state chips — use fakes/preview to force states).

## Dependencies

12 (10, 11 transitively; 04 for defaults).

## Non-goals

Onboarding flow itself (26 — embeds the preset picker component built here; export it as `ModelPresetPicker`), custom model import (v2), engine load/unload (15/17 lifecycle).

## Design References

DESIGN §12, §4.4 step 4, §13 (presets), §7.3 (`asr.modelId`, `format.llmModelId`); ADR-007.
