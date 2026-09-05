# Research: Local Japanese ASR Engines for kotodama

- Date: 2026-07-08
- Status: informs [ADR-002](../decisions/ADR-002-asr-engine.md) and DESIGN.md §8
- Question: which fully-local ASR engine and default model should power Japanese dictation on macOS (Apple Silicon)?

## Requirements recap

- Fully local inference (no network at runtime) — hard requirement.
- Japanese-first accuracy, including casual/spoken register.
- Short-utterance dictation latency: ≤ ~1.5 s ASR time for a 10 s utterance on M1.
- Redistribution-safe model licenses (app is OSS; models downloaded by users).
- Integratable from Swift on macOS 14+ (arm64).

## Candidates

| Engine / model | Type | License | Size (disk) | JA fit | Notes |
|---|---|---|---|---|---|
| whisper.cpp + kotoba-whisper-v2.0 (GGML) | distil-Whisper (large-v3 encoder, 2-layer decoder) | Apache-2.0 | f16 ≈ 1.5 GB, q5_0 ≈ 0.5 GB | Excellent (JA-specialized) | Official GGML conversion by kotoba-tech; trained on ReazonSpeech (7.2 M JA clips); ~6.3× faster than large-v3 at comparable JA error rate |
| whisper.cpp + large-v3-turbo (GGML) | distil-Whisper by OpenAI (4-layer decoder) | MIT (model) | f16 ≈ 1.5 GB, q8_0 ≈ 874 MiB, q5_0 ≈ 547 MiB | Good (multilingual) | Safe multilingual fallback; slightly weaker on casual JA than kotoba per community reports |
| Apple SpeechAnalyzer / DictationTranscriber | OS framework (macOS 26+) | n/a (OS) | 0 in app (OS-managed assets) | Good; JA among ~11 launch languages | On-device, reportedly ~2× faster than large-v3-turbo; zero download burden; requires macOS 26+, no model choice, quality not independently benchmarked for casual JA |
| ReazonSpeech (NeMo/k2 models) | Conformer family | Apache-2.0 | varies | Strong JA | No maintained C/C++/Swift inference path comparable to whisper.cpp; integration cost high on macOS |
| Parakeet (via FluidAudio, as used by VoiceInk) | NVIDIA Parakeet CoreML | model license needs per-model review | ~0.6–2.5 GB | EN-first; JA variants immature | Not JA-competitive as of research date |

## Key facts checked on 2026-07-08

- kotoba-tech publishes an official GGML repo (`kotoba-tech/kotoba-whisper-v2.0-ggml`) with `ggml-kotoba-whisper-v2.0.bin` and `ggml-kotoba-whisper-v2.0-q5_0.bin`, Apache-2.0. whisper.cpp runs it with sequential long-form decoding (HF pipeline uses chunked decoding; kotoba-tech found chunked slightly better for long audio — irrelevant for ≤ 2 min dictation utterances).
- whisper.cpp remains actively maintained (ggml-org), Metal on by default for Apple Silicon; upstream provides an XCFramework build path for Apple platforms. The old `whisper.spm` package is stale (Metal excluded) — do not use it.
- SpeechAnalyzer (WWDC25) ships `SpeechTranscriber`, `DictationTranscriber`, `SpeechDetector` modules; assets are OS-managed via AssetInventory; macOS 26 (Tahoe)+ only.

## Trade-off analysis

- **kotoba-whisper-v2.0 (q5_0) as default**: best JA quality-per-latency, tiny download (~0.5 GB), permissive license, engine fully under our control (works on macOS 14+). Cost: we own model download/integrity UX and whisper.cpp binding maintenance.
- **large-v3-turbo (q5_0/q8_0) as secondary**: for mixed JA/EN dictation and as an A/B baseline; same engine, so marginal cost is one catalog entry.
- **SpeechAnalyzer**: attractive later (zero download, OS-tuned) but macOS 26+ only, no control over register/vocabulary behavior, and JA casual-speech quality is unverified. Adopt as an optional second engine behind the same `ASREngine` protocol in v2.

## Recommendation

1. v1 engine: **whisper.cpp**, vendored as an XCFramework built from a pinned upstream tag.
2. Default model: **kotoba-whisper-v2.0 q5_0**; catalog also offers kotoba f16, large-v3-turbo q5_0/q8_0.
3. Design `ASREngine` as a protocol so SpeechAnalyzer can be added in v2 without touching the pipeline.
4. Known unknown (tracked in ISSUE_PLAN): distil-decoder models can be more prone to repetition loops under whisper.cpp defaults; the ASR issue must A/B kotoba vs turbo on fixture audio and tune `no_context`/temperature-fallback settings.

## Sources

- https://huggingface.co/kotoba-tech/kotoba-whisper-v2.0-ggml (license, files, whisper.cpp notes)
- https://huggingface.co/kotoba-tech/kotoba-whisper-v2.0 (training data, speed/accuracy claims)
- https://github.com/ggml-org/whisper.cpp and models/README.md (GGML sizes, Metal, XCFramework)
- https://developer.apple.com/documentation/speech/speechanalyzer and WWDC25 session 277 (SpeechAnalyzer modules, on-device)
- https://www.macstories.net/stories/hands-on-how-apples-new-speech-apis-outpace-whisper-for-lightning-fast-transcription/ (speed comparison, launch languages)
- https://github.com/beingpax/VoiceInk (Parakeet-via-FluidAudio precedent)
