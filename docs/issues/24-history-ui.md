# Title

History UI: browse, search, copy, delete

## Summary

Build the History window: paged session list with search, detail view (raw vs formatted), copy / re-insert actions, per-row delete, delete-all with confirmation, and empty/disabled states — on a tested view-model over `HistoryStoring`.

## Context

DESIGN §4.3 (menu opens 履歴…), §7.2 (data), §15 (privacy controls surfaced prominently). History is sensitive content; actions must be deliberate (no accidental bulk operations).

## Scope

`App/History/` + tests; strings JA/EN.

## Detailed Requirements

1. `HistoryViewModel` (@MainActor @Observable), deps: `HistoryStoring`, `SettingsStore`, controller re-insert entry point (protocol from 21/22), Clock.
2. Layout: search field (debounced 300 ms) + list (50/page, infinite scroll) rows: first 60 chars of display text (formatted ?? raw), style chip, relative date, target app icon if stored, status glyph (inserted/copied/cancelled/failed). Detail pane (selection): full formatted + raw (labeled tabs or stacked disclosure), metadata table (date, duration s, models, timings ms, method, status).
3. Actions: copy formatted / copy raw (persistent pasteboard write); 再挿入 (via controller; disabled while pipeline busy); delete row (⌫ + context menu, undo NOT provided — confirm dialog); すべて削除 (toolbar, destructive confirm naming count); open Privacy settings link when history disabled.
4. Disabled state: `history.enabled == false` → explanatory screen (not an error) + button to Settings.
5. Failed sessions show error messageKey text (localized) in detail.
6. Performance: list virtualized (SwiftUI List default fine); search runs on store, not in-memory filter (test asserts store call).
7. VM tests: paging accumulation + dedup, debounce (virtual clock), action gating (busy pipeline, empty selection), delete flows call store correctly, disabled-state routing, date formatting stable (fixed locale in tests).

## Acceptance Criteria

- [ ] All actions functional against real store on dev machine (recording: search → detail → copy → delete → delete-all).
- [ ] VM test suite green; store-call assertions for search/paging.
- [ ] Delete-all confirm shows exact count; post-delete UI returns to empty state.
- [ ] JA/EN screenshots; VoiceOver labels on row + actions (checklist).
- [ ] No content ever passes through logs (review note).

## Validation

CI VM tests + manual recording + screenshots in PR.

## Dependencies

23 (21 for window scene + re-insert protocol).

## Non-goals

Export (v2), retention settings UI (25 — link only), FTS, editing entries.

## Design References

DESIGN §4.3, §7.2, §15, §4.5 (error text), §19.
