# Title

ModelStore: on-disk model layout, scan, delete, re-verify

## Summary

Implement `ModelStore` (actor) owning `~/Library/Application Support/Kotodama/models/`: scan installed models against the catalog, resolve file paths for engines, delete, re-verify checksums on demand, and report storage usage.

## Context

DESIGN §7.1 fixes the layout; ADR-007 §4 requires re-verification paths (after `modelLoadFailed`, on user request). Engines (14/16) and UI (13) consume this API; nothing else touches the models directory.

## Scope

`Sources/ModelCatalog/Store/` + tests.

## Detailed Requirements

1. Root resolution: `FileManager.urls(for: .applicationSupportDirectory)/Kotodama/models` — injectable root for tests. Create with 0700 on first use.
2. API:
   - `func installed() -> [InstalledModel]` (`InstalledModel { entry: ModelEntry; fileURL: URL; verifiedAt: Date; sizeOnDisk: Int64 }`) — scans `models/<kind>/<id>/`, reads `meta.json`, matches against bundled catalog by id; orphans (dir without meta or id not in catalog) reported as `OrphanedItem` list, never auto-deleted.
   - `func fileURL(for id: String) throws -> URL` (throws `noASRModel`/`noLLMModel` mapping by kind).
   - `func delete(id: String) async throws` — removes the id directory; refuses while engine holds it (caller contract: controller unloads first; store checks an injected `InUseChecking` callback).
   - `func reverify(id: String) async throws -> Bool` — streaming SHA256 vs catalog; on mismatch: quarantine by renaming dir to `<id>.corrupt-<timestamp>` and return false (UI offers redownload + delete-quarantine).
   - `func storageUsage() -> (byModel: [String: Int64], totalBytes: Int64, orphanBytes: Int64)`.
3. `meta.json` schema versioned (`metaVersion: 1`); unreadable meta → treat as orphan.
4. All mutations logged (ids/sizes only). No writes outside the models root (test asserts with sandboxed temp root).
5. Migration stance: none needed (first version); document that layout changes require a migration function keyed on `metaVersion`.

## Acceptance Criteria

- [ ] Temp-root tests: fresh scan empty; installed fixture (tiny dummy file + meta) listed with correct size; orphan detection (3 variants: no meta, bad meta, unknown id); delete removes only target dir; reverify pass and mismatch→quarantine paths; storageUsage sums match fixtures.
- [ ] `fileURL(for:)` error mapping tested for both kinds.
- [ ] In-use deletion refusal tested via injected checker.
- [ ] Directory/file permission bits asserted (0700 dirs).

## Validation

CI unit tests on temp roots; PR includes a real-machine `installed()` dump after issue 11's manual download.

## Dependencies

10, 11.

## Non-goals

Download (11), UI (13), automatic orphan cleanup (surfaced to UI only), engine loading (14/16), history storage (23).

## Design References

DESIGN §7.1, §12; ADR-007 §4 (re-verify), §14 T1.
