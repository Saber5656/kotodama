# Title

Localization: JA/EN String Catalog completeness

## Summary

Complete the JA (source) / EN localization pass: audit every user-visible string into `Localizable.xcstrings`, translate to natural EN, add the hardcoded-literal lint, and run the locale QA checklist over all surfaces.

## Context

DESIGN §19: JA is the source language, EN ships at v1. Prior issues added strings incrementally; this issue guarantees completeness and quality before release.

## Scope

`App/Resources/Localizable.xcstrings` (+ InfoPlist strings), lint config, `docs/qa/l10n-checklist.md`. String *content* fixes only — no behavior changes.

## Detailed Requirements

1. Audit: scripted sweep (`scripts/ci/check-hardcoded-strings.sh`: regex for JA characters or quoted literals inside `Text(`, `Label(`, `.title`, `NSLocalizedString`-bypasses in `App/**` and UI-adjacent KotodamaKit sources) → every hit either catalog-referenced or allow-listed with reason (symbols, format specimens).
2. Wire the sweep into CI (warning→error once clean).
3. EN translation for 100 % of keys: natural product English (not literal); glossary fixed in `docs/qa/l10n-checklist.md`: 整文 = "Clean up", 敬語 = "Keigo (polite)", 音声入力 = "dictation", styles/menu/error terms table — apply consistently.
4. InfoPlist localizations: mic usage description JA+EN.
5. Pluralization/format checks: elapsed time, sizes (GB), counts (history delete confirm) use format styles correctly in both locales.
6. QA checklist execution: launch with `-AppleLanguages (en)` and `(ja)`: walk menu, HUD states (fake-driven), all Settings panes, onboarding, History, Models, all error-catalog HUD lines (drive via debug menu or previews) — screenshot pairs; check truncation/clipping at default window sizes; check date/number formatting.
7. messageKey drift test (from 29) extended to assert EN values non-empty and ≠ JA (crude untranslated guard).

## Acceptance Criteria

- [ ] Sweep script green in CI as error-level; allow-list justified.
- [ ] 100 % keys have EN; drift test green.
- [ ] JA+EN screenshot set committed under `docs/qa/l10n-screenshots/` (or PR attachments, linked) covering every surface listed above.
- [ ] No clipped/truncated strings at default sizes (checklist signed).
- [ ] Glossary documented; spot-review of 20 random EN strings by a second reader noted.

## Validation

CI + executed checklist with screenshots in PR.

## Dependencies

20, 21, 24, 25, 26 (29 for error strings).

## Non-goals

Additional languages (v2 via community), marketing/website copy, README translation (32), RTL.

## Design References

DESIGN §19 (normative), §3.1 DoD 13 partially, §4.5.
