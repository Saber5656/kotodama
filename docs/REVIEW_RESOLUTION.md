# Review resolution record

- Repository: `Saber5656/kotodama`
- Pull request: #33
- Parent head observed before this addendum: `a4a109236a3f13ab188a935c1d2802accd32c077`
- Scope: existing review threads only; no new Bot review is requested.
- This document records design-level resolutions and focused verification gates. It does not claim implementation or test completion.

## Thread `PRRT_kwDOTNkFz86PFvyK`

### Preserve the raw transcript through formatting

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyK` identifies this contract gap.
- Normative resolution: Carry the raw ASR transcript in formatting state and through timeout/error transitions so `fallbackInsertRaw(text:)` can insert exactly the captured text.
- Focused verification before resolving this thread: Force a formatter timeout and assert the reducer retains the original transcript and enters insertion with the raw text.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkFz86PFvyM`

### Keep formatting failures on the fallback insertion path

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyM` identifies this contract gap.
- Normative resolution: Align the acceptance criteria with DESIGN row 13: formatter failure/timeout transitions to `inserting` with raw text, not directly to `idle`; completion and failure events remain explicit.
- Focused verification before resolving this thread: Exercise formatter error and timeout separately and assert both follow the raw insertion path and never discard the session silently.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkFz86PFvyN`

### Move HistoryStore before the controller wave

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyN` identifies this contract gap.
- Normative resolution: Reorder the wave/dependency graph so HistoryStore issue 23 is available before controller issue 22, or remove the dependency; the published execution order must be topologically valid.
- Focused verification before resolving this thread: Run the dependency consistency/topological-order check and assert controller has its HistoryStore producer before implementation.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkFz86PFvyP`

### Resolve shortcut ownership before adding drift tests

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyP` identifies this contract gap.
- Normative resolution: Make the ownership table authoritative: KeyboardShortcuts owns the shortcut key while SettingsStore owns only the settings it actually persists, and drift tests assert the resolved owner rather than requiring duplicate properties.
- Focused verification before resolving this thread: Run the settings/shortcut drift matrix and assert every DESIGN §7.3 row has one owner and one persistence contract.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkFz86PFvyQ`

### Point the network ban at the package sources

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyQ` identifies this contract gap.
- Normative resolution: Scope the no-network source audit to the actual `Packages/**/Sources/` trees and any app source roots used by CI, while excluding generated/build/test fixtures through an explicit rule.
- Focused verification before resolving this thread: Place a forbidden URLSession call in each package source root and a permitted test harness call, then assert only production violations fail.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkFz86PFvyR`

### Inject canary download state instead of importing ModelCatalog

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyR` identifies this contract gap.
- Normative resolution: Expose the download-state observation through a protocol or injected provider supplied by ModelCatalog/Download; Diagnostics consumes the seam and never imports ModelCatalog, avoiding a cycle.
- Focused verification before resolving this thread: Run the diagnostics canary with an injected active-download fixture and assert it observes state without a package dependency cycle.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkFz86PFvyU`

### Keep hotkey-down processing consistent with row 18

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyU` identifies this contract gap.
- Normative resolution: Treat hotkeyDown in every non-idle/non-recording state as ignored with a busy flash per row 18; reserve cancellation for the explicit Escape event and document the exception table.
- Focused verification before resolving this thread: Send hotkeyDown and Escape in transcribing, formatting, and inserting states and assert the specified action for each event.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkFz86PFvyY`

### Handle missing LLM after ASR succeeds

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyY` identifies this contract gap.
- Normative resolution: Add a post-ASR guard for a missing/deleted non-raw formatter model: emit the defined localized error and either insert raw text or terminate through the documented fallback, without leaving the state machine unmatched.
- Focused verification before resolving this thread: Delete the formatter model after prerequisite validation, complete ASR, and assert a deterministic fallback/error transition.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkFz86PFvyb`

### Persist the failure reason for history detail

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyb` identifies this contract gap.
- Normative resolution: Add a stable failure `messageKey` (and any safe parameter data) to DictationSessionRecord and persist it for every failed/timeout path so history detail can localize it after reload.
- Focused verification before resolving this thread: Fail ASR, formatting, and insertion independently, reload history, and assert each record retains the correct reason key without raw secrets.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkFz86PFvyd`

### Treat SQLite WAL sidecars as history data

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyd` identifies this contract gap.
- Normative resolution: Apply the history file permission, backup, cleanup, and exclusion policy to `history.sqlite-wal` and `history.sqlite-shm` as well as the main database; checkpoint/remove sidecars only under the defined lifecycle.
- Focused verification before resolving this thread: Write history in WAL mode and assert all three files have the required permissions and are covered by backup/cleanup guards.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkFz86PFvyh`

### Put the catalog where package tests can load it

- Finding: The existing review thread `PRRT_kwDOTNkFz86PFvyh` identifies this contract gap.
- Normative resolution: Place the canonical `model-catalog.json` in a SwiftPM resource target accessible to ModelCatalog package tests, or provide a checked fixture plus a drift check against the app resource; do not depend on an App-only path.
- Focused verification before resolving this thread: Run `swift test --package-path` from the package and assert it loads the canonical catalog and detects intentional drift.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Bot review policy

The existing Bot review is not re-triggered for this PR. Replies and thread resolution are performed only after the focused verification conditions above are recorded.