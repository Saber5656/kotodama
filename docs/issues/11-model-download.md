# Title

ModelDownloadService: verified, resumable model downloads

## Summary

Implement `ModelDownloadService` (actor): the app's **only** networking code. URLSession download with resume, disk preflight, streaming SHA256, verify-before-install, atomic install into the model store layout, progress stream, cancellation, and the DESIGN error taxonomy.

## Context

ADR-004 (sole network surface) and ADR-007 (integrity pipeline). This module's boundaries are security-normative: issue 27 adds a lint rule banning network imports elsewhere, so ALL network code must land here.

## Scope

`Sources/ModelCatalog/Download/` + tests (module stays `ModelCatalog` so the lint boundary is one module).

## Detailed Requirements

1. API (actor `ModelDownloadService`):
   - `func download(_ entry: ModelEntry) -> AsyncThrowingStream<DownloadProgress, Error>` (`DownloadProgress { bytesReceived, totalBytes, phase: downloading|verifying|installing }`)
   - `func cancel(id: String)` (persists resume data), `func activeDownloads() -> [String]`
   - Single concurrent download (queue further requests FIFO) — simplicity + disk/network sanity.
2. Networking: `URLSession` with ephemeral configuration (`urlCache=nil`, `httpCookieStorage=nil`, `waitsForConnectivity=false`, timeoutResource 3600 s); `allowsCellularAccess` irrelevant on macOS but set false anyway. Delegate-based `downloadTask` for resume-data support. **Redirect policy**: allow only to hosts matching `huggingface.co` or `*.huggingface.co` or `*.hf.co` (LFS CDN) — verify actual CDN hosts at implementation and pin the allow-list; any other redirect → fail `downloadFailed(.network)`.
3. Preflight: free space at destination volume ≥ `sizeBytes * 1.1` else `downloadFailed(.diskFull)` before any network I/O.
4. Verify: after download completes, stream-hash the file (SHA256, CryptoKit, 4 MiB chunks) and compare to `entry.sha256`; also compare byte size to `sizeBytes`. Mismatch → delete file + resume data, throw `downloadFailed(.checksum)` (never leave unverified bytes at final path).
5. Install: `models/tmp/<id>.partial` → verified → `FileManager.moveItem` (atomic, same volume) to `models/<kind>/<id>/model.<bin|gguf>` + write `meta.json { catalogEntry, verifiedAt, appVersion }` (0644 file / 0700 dirs).
6. Resume: store resume data alongside partial; `download()` on an entry with resume data continues; corrupted resume data falls back to fresh download (test).
7. Progress: emit ≤ 10 events/s; phases in order; `verifying` phase reports hash progress.
8. Logging via `KotodamaLog.models`: id, bytes, outcome — never URLs with tokens (none exist, but rule stated).
9. Tests: `URLProtocol`-stubbed session covering success / checksum-mismatch / disk-full (injected `FreeSpaceProviding`) / cancel+resume / redirect-to-forbidden-host / 404 / connection-drop-mid-body. A manual integration doc section: real download of the smallest catalog entry with network log (executed for PR).

## Acceptance Criteria

- [ ] All stubbed tests green in CI; no test touches the real network.
- [ ] Forbidden-redirect and checksum-mismatch paths leave zero files under `models/` final paths (filesystem assertions).
- [ ] Cancel → resume continues from prior offset (stub asserts Range/resume-data behavior).
- [ ] Manual real download of kotoba q5_0 succeeds with hash verify; transcript in PR.
- [ ] Module review confirms: no networking symbol outside `ModelCatalog` module (grep `URLSession|NWConnection|Network\b` — output in PR; prepares issue 27 rule).

## Validation

CI + manual download transcript + grep output in PR.

## Dependencies

05, 10.

## Non-goals

UI (13), scan/delete/re-verify of installed models (12), remote catalog (never in v1), parallel downloads, mirror fallback.

## Design References

DESIGN §12, §7.1 (tmp/final layout), §6.4 (downloadFailed), §14 T6/T8; ADR-004; ADR-007.
