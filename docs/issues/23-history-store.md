# Title

HistoryStore: SQLite sessions, retention, delete-all

## Summary

Implement `HistoryStore` (actor) over GRDB: DESIGN §7.2 schema, append honoring history settings, paged queries + search, retention pruning, delete-one/delete-all (VACUUM), 0600 file permissions, and privacy rules (skip-when-disabled, optional target-app column).

## Context

DESIGN §7.2 (schema normative) and §14 T5 (at-rest posture). History is the only persistence of dictated content; its privacy switches are product-level promises (§15).

## Scope

`Sources/HistoryStore/` + tests; adds pinned `GRDB.swift` dependency.

## Detailed Requirements

1. Dependency: `groue/GRDB.swift` pinned `.exact(<latest stable>)`; Package.resolved committed (DESIGN §18).
2. DB at `~/Library/Application Support/Kotodama/history.sqlite` (root injectable); create with `FileProtection` n/a on macOS → enforce POSIX 0600 on file + 0700 on dir after creation (test asserts). WAL mode; busy timeout 2 s.
3. Migration v1 = exact DESIGN §7.2 DDL via GRDB migrator (`v1` identifier; future changes append migrations only).
4. API (`HistoryStoring` + fake):
   - `func append(_ record: DictationSessionRecord, settings: HistorySettingsSnapshot) async` — no-op when `history.enabled == false` (snapshot passed by caller to avoid races); drops `targetBundleID` when `history.storeTargetApp == false`; never throws to caller (log + `storageFailure` metric only — dictation must not fail on history errors).
   - `func page(offset: Int, limit: Int) async throws -> [DictationSessionRecord]` (started_at DESC).
   - `func search(_ query: String, limit: Int) async throws -> [DictationSessionRecord]` — LIKE over raw_text/formatted_text with escaped wildcards (no FTS in v1; document).
   - `func delete(id: UUID) async throws`, `func deleteAll() async throws` (DELETE + `VACUUM`).
   - `func pruneExpired(retentionDays: Int) async throws -> Int` (0 = keep forever); called by app on launch + 24 h timer (wire in 22's startup — expose method here).
   - `func count() async throws -> Int`.
5. Mapping `DictationSessionRecord` ↔ row exactly per §7.2 (Date ↔ unix seconds; enums ↔ TEXT raw values) with round-trip tests.
6. Corruption handling: GRDB open failure → rename `history.sqlite` → `history.corrupt-<ts>.sqlite`, recreate fresh, log + surface one-time notice event (`storageFailure`) — never crash, never block dictation.
7. No content logging; queries parameterized (GRDB arguments only — lint note: string interpolation into SQL forbidden).

## Acceptance Criteria

- [ ] Temp-dir tests: round-trip fidelity; disabled→no rows; storeTargetApp=false→NULL column; paging order; search escaping (`%`,`_`,`'`); delete/deleteAll (+file size shrink after VACUUM evidenced); prune boundaries (exactly-30-days edge, retention 0 keeps); corruption path (garbage file → quarantine + fresh DB); permission bits.
- [ ] append never throws (fault-injection test with read-only dir).
- [ ] Concurrency: 100 interleaved appends+pages produce consistent counts (actor serialization test).

## Validation

CI unit tests; PR shows schema dump (`sqlite3 .schema`) matching DESIGN §7.2 verbatim.

## Dependencies

03.

## Non-goals

UI (24), encryption (v2 SQLCipher, DESIGN §3.2), FTS search, export (v2), audio storage (never).

## Design References

DESIGN §7.2 (normative), §7.1, §14 T5, §15; §7.3 (`history.*` keys).
