# Title

ModelCatalog: bundled catalog schema, parsing, initial entries

## Summary

Implement catalog types + parser/validator for `model-catalog.json` (schema DESIGN §7.4) and author the initial catalog: 4 ASR entries (kotoba-whisper-v2.0 q5_0/f16, large-v3-turbo q5_0/q8_0) and 2 LLM entries (Qwen3-4B-Instruct-2507 Q4_K_M, sarashina2.2-3b-instruct Q4_K_M) with pinned revisions, real SHA256 values, sizes, licenses, provenance.

## Context

ADR-007: only curated, pinned, checksum-verified models are loadable. The catalog is a bundled resource; its correctness is a security control (DESIGN §14 T1/T8), so validation is strict and checksums are produced by a documented procedure.

## Scope

`Sources/ModelCatalog/` (catalog types + loader only — download/store are issues 11/12), `App/Resources/model-catalog.json`, `docs/qa/model-checksum-procedure.md`.

## Detailed Requirements

1. Types (`Codable`, `Sendable`): `ModelCatalog { schemaVersion: Int; models: [ModelEntry] }`, `ModelEntry` with exactly the DESIGN §7.4 fields (`id, kind, displayName, family, format, url, sha256, sizeBytes, license, licenseURL, provenance, languages, minRAMGB, recommendedFor`).
2. `CatalogLoader.load(from data: Data) throws -> ModelCatalog` — validation, each with its own thrown error: `schemaVersion == 1`; unique ids; ids match `^[a-z0-9][a-z0-9.\-]{2,63}$`; `url` https + host ∈ {`huggingface.co`} and path contains `/resolve/<40-hex-or-tag>/` where revision must be a 40-char commit hash (reject `main`); sha256 is 64-hex; sizeBytes > 0; kind/format consistent (asr→ggml, llm→gguf); license ∈ {Apache-2.0, MIT} for shipped catalog (validator flag `allowedLicenses` injectable).
3. `bundledCatalog()` loads the resource; a **unit test loads the real bundled JSON through the strict validator** (catalog CI drift-guard).
4. Author the initial catalog file. Checksum procedure (execute + document in `docs/qa/model-checksum-procedure.md`): download each artifact from the pinned revision URL on two distinct networks/machines (or network + HF web UI hash where shown), compute `shasum -a 256`, require agreement, record: URL, revision, sha256, size, date, who. For LLM GGUFs prefer the model vendor's official GGUF repo at a pinned revision; if unavailable, use a reputable quantizer repo (bartowski/unsloth) pinned by revision and record that provenance string.
5. Helper queries: `entries(kind:)`, `entry(id:)`, `defaultASR` (`kotoba-whisper-v2.0-q5_0`), `defaultLLM(forRAMGB:)` mirroring SettingsStore logic ids (`qwen3-4b-instruct-2507-q4km`, `sarashina2.2-3b-instruct-q4km`) — ids in catalog MUST equal ids referenced by SettingsStore defaults (cross-module test).
6. Display names bilingual-ready: `displayName` is the JA string; add `displayNameKey` only if trivial — otherwise document that Models UI (13) localizes via id-keyed strings (choose the latter; keep schema minimal — record decision in code comment + this issue's PR).

## Acceptance Criteria

- [ ] Validator rejects each malformed-fixture case (≥ 10 negative fixtures: dup id, http URL, `main` revision, bad sha length, wrong host, zero size, kind/format mismatch, unknown license, schemaVersion 2, bad id charset).
- [ ] Bundled catalog passes strict validation in CI; all 6 entries have real sha256 + sizeBytes (no placeholders).
- [ ] `docs/qa/model-checksum-procedure.md` committed with the actual recorded values table (evidence).
- [ ] Cross-check test: SettingsStore default model ids exist in catalog.
- [ ] Licenses of all entries ∈ {Apache-2.0, MIT} with working licenseURL (link-checked manually, listed in PR).

## Validation

CI tests + checksum procedure evidence table in PR (two independent hash sources per artifact).

## Dependencies

03 (04 for the id cross-check test).

## Non-goals

Downloading (11), disk layout (12), UI (13), remote catalog (ADR-007 v2), model quality evaluation (18).

## Design References

DESIGN §7.4, §14 T1/T8; ADR-007; research/2026-07-asr-engines.md, research/2026-07-formatting-llm.md (chosen artifacts).
