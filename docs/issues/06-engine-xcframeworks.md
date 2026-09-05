# Title

build-frameworks.sh: vendored whisper.cpp / llama.cpp XCFrameworks

## Summary

Create `scripts/build-frameworks.sh` that clones whisper.cpp and llama.cpp at **pinned tags**, builds macOS arm64 XCFrameworks with Metal enabled, and installs them into `Frameworks/`; wire the app target and CI caching to consume them.

## Context

ADR-002/003 vendor both engines from pinned upstream tags (stale SPM mirrors rejected; prebuilt third-party binaries rejected for supply-chain reasons, DESIGN §14 T7). Both upstreams ship an Apple build path (`build-xcframework.sh`); exact invocation must be verified against the pinned tag's docs at implementation time.

## Scope

- `scripts/build-frameworks.sh`, `scripts/frameworks.lock` (pin file), `project.yml` linkage, CI cache step (reserved in issue 02), `docs/BUILD.md` update.

## Detailed Requirements

1. `scripts/frameworks.lock` (sourceable shell or plain KEY=VALUE): `WHISPER_CPP_TAG=<latest stable tag at implementation>`, `WHISPER_CPP_SHA=<commit sha of tag>`, `LLAMA_CPP_TAG=…`, `LLAMA_CPP_SHA=…`. Script verifies the checked-out commit == pinned SHA (tag spoofing guard) and aborts otherwise.
2. Script behavior: idempotent; clones into `build/vendor/` (gitignored); builds **macOS arm64 only** XCFrameworks (trim other platforms if upstream script builds them — document how); output `Frameworks/whisper.xcframework`, `Frameworks/llama.xcframework`; writes `Frameworks/MANIFEST.json` (tags, shas, build date, Xcode version); `--check` flag exits 0 iff frameworks exist and MANIFEST matches lock file.
3. Metal must be enabled/embedded (default for both upstreams on Apple); script asserts the built framework contains the Metal artifacts expected by the pinned tag (e.g., embedded metallib or ggml-metal sources — verify per tag and encode the check).
4. `project.yml`: link+embed both XCFrameworks into the app target (embed & sign). Build phase or pre-build check that runs `build-frameworks.sh --check` and fails with a clear "run scripts/build-frameworks.sh" message.
5. CI (edit `.github/workflows/ci.yml` reserved step): cache `Frameworks/` keyed on hash of `frameworks.lock` + Xcode version; on miss run the script (accept ~10–20 min once per pin bump).
6. Smoke targets: add to KotodamaKit two tiny test-only C-interop checks proving symbol linkage: call `whisper_print_system_info()` and `llama_print_system_info()` (or the pinned tag's equivalents) via module maps from `ASRService`/`FormattingService` test targets (guard with `#if canImport`). This proves headers+libs are consumable before issues 14/16.
7. `docs/BUILD.md`: add frameworks section (one command, expected duration, disk needs, how to bump pins = edit lock + PR with upstream changelog review note per DESIGN §14 T7).
8. Fallback plan documented in script header: if XCFramework path breaks on a future tag, static-lib + modulemap approach (DESIGN §21 U5) — do not implement now.

## Acceptance Criteria

- [ ] Clean clone → `scripts/build-frameworks.sh` → `xcodegen generate` → app builds and links both frameworks; smoke tests print system info strings.
- [ ] Re-run is a no-op (< 5 s) via `--check` semantics.
- [ ] Tampering test: editing lock SHA without rebuilding makes `--check` fail (shown in PR).
- [ ] CI green with cache miss AND cache hit runs linked in PR.
- [ ] MANIFEST.json contents pasted in PR; pins are current upstream stable tags with their changelogs skimmed and linked.

## Validation

PR evidence: script transcript, CI run links (miss+hit), smoke-test output lines.

## Dependencies

01, 02.

## Non-goals

Swift API wrappers (14, 16), model files (10–12), upstream code modifications (none allowed — build scripts only), universal/Intel builds.

## Design References

ADR-002, ADR-003; DESIGN §5.2 (Frameworks/), §14 T7, §21 U5; research/2026-07-macos-dictation-integration.md (vendoring).
