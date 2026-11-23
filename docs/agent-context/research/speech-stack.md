# Research — on-device speech stack (sherpa-onnx, Rust, Android-first)

Researched 2026-11-23. Evidence tags: **VERIFIED** = read in a primary source this session (repo source, release API, HF API/model card, crates.io API, or measured locally); **SUPPORTED** = secondary source only; **UNVERIFIED** = memory/inference, needs checking. "Measured locally" = downloaded the artifact and inspected it on this machine (no ASR inference was run).

## TL;DR

- **Binding path: the official `sherpa-onnx` crate (v2.0.3, Apache-2.0, maintained in k2-fsa/sherpa-onnx).** It wraps the C API, supports every offline recognizer we need (NeMo transducer/Parakeet, Whisper, Moonshine v1+v2), plus Silero VAD and punctuation. Its build script auto-downloads prebuilt libs; for Android it links **shared** only and copies `.so` files into the Tauri `jniLibs/` dir; upstream ships a Tauri v2 `hello_world` example that builds for Android. The community `sherpa-rs` is **archived/deprecated** in favour of it. No Kotlin/JNI layer needed.
- **Default MVP model: Parakeet TDT 0.6B v2 int8** (`sherpa-onnx-nemo-parakeet-tdt-0.6b-v2-int8`, 482 MB download, ~661 MB on disk). Best English accuracy among the candidates, native punctuation + capitalization, CC-BY-4.0. Caveats: big download; no published Android RTF or peak-RAM figure; the closest ARM CPU number is RTF 0.088 at 4 threads on RK3588 Cortex-A76 (the same core as a Pi 5).
- **Fallback: Parakeet TDT 110M int8** (`sherpa-onnx-nemo-parakeet_tdt_transducer_110m-en-36000-int8`, 108 MB, P&C, CC-BY-4.0). It uses the same config struct and decode path as the default, so it costs no extra code. **Ultra-light tier: Moonshine v2 tiny-en** (30 MB, MIT, P&C). Avoid Whisper as the default: sherpa-onnx cuts input at about 30 s, the `.en` models are much larger for the accuracy, and Hindi/Tamil output gets truncated unless `max_tokens_per_second` is raised.
- **Blocking unknowns: time from ✓ to text and peak RSS on the iQOO Neo 10R** (Snapdragon 8s Gen 3 / SM8635, SUPPORTED). Run the benchmark protocol at the end before freezing the default. There are no prebuilt QNN (NPU) binaries for SM8635, so the MVP is CPU-only.

## Comparison table

Sizes are in bytes or MB from the GitHub release API and the HF tree API (VERIFIED). "On disk" = sum of the model files needed at runtime.

| Model (sherpa artifact) | Params | Download (tar.bz2) | On disk (runtime files) | Languages | P&C native | Weight license | Speed evidence |
|---|---|---|---|---|---|---|---|
| Parakeet TDT 0.6B **v2 int8** | 600M | 482,440,461 | enc 652.3 MB + dec 7.3 + join 1.7 + tokens ≈ **661 MB** | en | **Yes** | CC-BY-4.0 | RTF 0.088 at 4 threads / 0.220 at 1 thread, RK3588 A76; RTF 0.118 at 2 threads on unnamed hardware (sherpa docs) |
| Parakeet TDT 0.6B **v3 int8** | 600M | 487,211,443 | enc 652.3 + dec 11.8 + join 6.4 ≈ **671 MB** | 25 European (no hi/ta) | Yes | CC-BY-4.0 | RTF 0.325 at 2 threads, unnamed hardware, 3.8 s clip (sherpa docs) |
| Parakeet 0.6B v2 / v3 **fp32** | 600M | no GitHub tarball; HF only | enc.onnx 41.8 MB + **encoder.weights 2,435 MB** + dec 28.9/47.2 + join 6.9/25.3 ≈ **2.5 GB** | en / 25 EU | Yes | CC-BY-4.0 | none found |
| Parakeet 0.6B v2 **fp16** | 600M | 1,120,982,957 | enc 1,239 MB + dec 14.4 + join 3.5 ≈ 1.26 GB | en | Yes | CC-BY-4.0 | none found |
| **Parakeet TDT 110M int8** (`tdt_transducer_110m-en-36000`) | ~114M | 108,035,095 | enc 131.1 + dec 4.0 + join 1.4 ≈ **137 MB** | en | **Yes** | CC-BY-4.0 | none found for sherpa; NVIDIA card WER LS-clean 2.4 / other 5.2 / AMI 15.88 |
| **Moonshine v2 tiny-en** (`quantized-2026-02-27`) | UNVERIFIED (~27M per v1 paper) | 29,858,559 | enc 13.3 + merged dec 30.4 (.ort) ≈ **44 MB** | en | **Yes** (sample output) | **MIT** (English) | RTF 0.056 at 2 threads, unnamed hardware, 3.8 s clip (sherpa docs) |
| Moonshine v2 base-en (`quantized-2026-02-27`) | UNVERIFIED (~61M per v1 paper) | 111,266,225 | enc 31.3 + dec 109.4 ≈ **141 MB** | en | Yes (sample output) | MIT (English) | RTF 0.109 at 2 threads, unnamed hardware |
| Moonshine v1 tiny/base-en int8 (2024-10-26) | 27M / 61M (UNVERIFIED) | 107,600,538 / 250,807,309 | ≈ 124 MB / ≈ 287 MB (preprocess + enc + cached + uncached dec) | en | UNVERIFIED | MIT | superseded by v2 in sherpa; skip |
| Whisper tiny.en (int8) | 39M | 118,071,777 (int8 + fp32 in one tarball) | int8: enc 12.9 + dec 89.9 ≈ **103 MB** | en | Yes (Whisper style) | MIT | RTF 0.386 (int8, 4 threads) / 0.547 (1 thread) on **Raspberry Pi 4** (sherpa docs) |
| Whisper base.en (int8) | 74M | 208,576,005 | int8: enc 29.1 + dec 130.7 ≈ 160 MB | en | Yes | MIT | none in sherpa docs |
| Whisper small.en (int8) | 244M | 635,693,775 | int8: enc 112.4 + dec 262.2 ≈ 375 MB | en | Yes | MIT | none in sherpa docs |
| Zipformer libriheavy small **punct-case** | UNVERIFIED | 309,964,940 | int8: enc 68.8 + dec 0.7 + join 0.4 ≈ 70 MB | en | Yes (by name; trained on LibriVox audiobooks) | Apache-2.0 (icefall, UNVERIFIED for weights) | none found |

Silero VAD: `silero_vad.onnx` 643,854 B; `silero_vad.int8.onnx` 212,860 B (VERIFIED, release `asr-models`).

## Q1 — Using sherpa-onnx from Rust (and alternatives)

**Official Rust API exists. VERIFIED**
- Crates `sherpa-onnx` and `sherpa-onnx-sys` are both at 2.0.3 (published 2026-11-22), Apache-2.0, repo k2-fsa/sherpa-onnx, about 0.9 M downloads each. Recent cadence: 2.0.0 (10-23), 2.0.1 (10-27), 2.0.2 (11-19), 2.0.3 (11-22). Source: crates.io API `https://crates.io/api/v1/crates/sherpa-onnx`; source tree `sherpa-onnx/rust/{sherpa-onnx,sherpa-onnx-sys}` in https://github.com/k2-fsa/sherpa-onnx.
- `src/offline_asr.rs` exposes `OfflineModelConfig` with `transducer` (used for NeMo Parakeet TDT), `nemo_ctc`, `whisper`, `moonshine`, `canary`, `sense_voice`, `omnilingual`, `cohere_transcribe`, `qwen3_asr` and more, plus `num_threads` and `provider: Option<String>`. `OfflineMoonshineModelConfig` has `encoder` + `merged_decoder` for v2 and `preprocessor`/`uncached_decoder`/`cached_decoder` for v1. Other modules: `vad.rs` (Silero + TEN VAD, `max_speech_duration`), `offline_punctuation.rs`, `online_punctuation.rs`, `resampler.rs`. Source: https://github.com/k2-fsa/sherpa-onnx/blob/master/sherpa-onnx/rust/sherpa-onnx/src/offline_asr.rs
- Examples in `rust-api-examples/` include `nemo_parakeet`, `moonshine_v2`, `whisper`, `silero_vad_remove_silence`, `offline_punctuation`, and `parakeet_tdt_simulate_streaming_microphone` (VAD + offline Parakeet). Source: https://github.com/k2-fsa/sherpa-onnx/blob/master/rust-api-examples/README.md

**Linking. VERIFIED** (`rust-api-examples/for-advanced-users.md`, `sherpa-onnx-sys/build.rs`)
- Desktop: the `static` feature is the default; `shared` is opt-in. Without `SHERPA_ONNX_LIB_DIR`, `build.rs` downloads `sherpa-onnx-v{VER}-{platform}-{static|shared}-lib.tar.bz2` from GitHub releases with `ureq`.
- **Android: shared only.** `build.rs` forces shared mode for `android` and `ios`. It downloads `sherpa-onnx-v{VER}-android.tar.bz2` (all ABIs), maps `aarch64-linux-android` to `arm64-v8a`, and **copies the `.so` files into the Tauri Android project's `jniLibs/` directory**. Needs NDK r27+ and `ANDROID_NDK_HOME`. Documented commands: `cargo build --target aarch64-linux-android --release --no-default-features --features shared`, or `cargo tauri android build` from `tauri-examples/hello_world` ("Supports Linux, macOS, Windows, iOS, and Android"). cargo-ndk is **not** mentioned; the Tauri CLI path is the documented one. The two ASR Tauri examples are documented as desktop apps.
- **Supply-chain gap: `build.rs` does not verify a checksum of the downloaded archive.** It fetches the URL and unpacks it; grepping for "sha" finds only "Shared". Mitigation: in CI, set `SHERPA_ONNX_ARCHIVE_DIR` to a pre-fetched archive whose SHA-256 we pin, or set `SHERPA_ONNX_LIB_DIR`.
- Android libs are **16 KB page-aligned**: `readelf` shows LOAD align `0x4000` on `libonnxruntime.so` and `libsherpa-onnx-c-api.so` (measured locally, v2.0.3). That matters for the Play Store 16 KB requirement (the requirement itself is UNVERIFIED here). The bundled ONNX Runtime is **1.28.2** and contains the `NnapiExecutionProvider` symbols (measured locally).

**Kotlin/JNI. VERIFIED** — `sherpa-onnx/kotlin-api/` has `OfflineRecognizer.kt`, `Vad.kt`, `OfflinePunctuation.kt`, `QnnConfig.kt` and others. Release assets: `sherpa-onnx-2.0.3.aar` (52,411,426 B) and `sherpa-onnx-static-link-onnxruntime-2.0.3.aar` (39,999,115 B). Unneeded if Rust owns ASR: the Rust crate links `libsherpa-onnx-c-api.so` directly.

**Alternatives**

| Crate | Version / date | License | Engines | Android | Verdict |
|---|---|---|---|---|---|
| `sherpa-rs` (thewh1teagle) | 0.6.8, 2025-10-05 | MIT | sherpa-onnx | — | **Archived**; README says "This crate is deprecated… use the official Rust bindings" (VERIFIED) |
| `transcribe-rs` (cjpais) | 0.3.12, 2026-10-02 | MIT | `onnx` feature (via ORT): Parakeet, Canary, Cohere, Moonshine, SenseVoice, GigaAM; `whisper-cpp` feature | README doesn't mention Android (VERIFIED absence) | Viable desktop option, but a second ONNX stack; **Handy itself has moved off it** (below) |
| `transcribe-cpp` (handy-computer/transcribe.cpp) | 0.4.2, 2026-11-12 | MIT | ggml/GGUF "16+ model families" (Parakeet GGUF Q8_0 731 MB) | UNVERIFIED | Newer; Handy's current engine. Watch, don't adopt for MVP |
| `whisper-rs` | 0.16.0, 2026-03-12 | Unlicense (repo moved to codeberg) | whisper.cpp only | UNVERIFIED | Whisper only: no Parakeet, 30 s windows |
| `ort` (pykeio) | 2.0.0-rc.15 (no stable 2.x), 2026-10-19 | MIT OR Apache-2.0 | raw ONNX Runtime | **prebuilt binaries for android arm64** (docs `platform.tsx` → `PLATFORMS_WITH_BINARIES`) (VERIFIED) | We'd re-implement fbank, TDT decoding, tokenizer and VAD ourselves. No. |

Sources: crates.io API for each crate; GitHub API repo metadata; https://github.com/cjpais/transcribe-rs README; https://github.com/cjpais/Handy/blob/main/src-tauri/Cargo.toml; https://github.com/pykeio/ort/blob/main/docs/core/platform.tsx

## Q2 — sherpa-onnx license, release, lib sizes, acceleration

- License: **Apache-2.0** (GitHub API; VERIFIED). Apache-2.0 code inside a GPL-3.0 app is generally considered compatible (FSF position; SUPPORTED, not re-read this session). Keep its LICENSE/NOTICE in the app's third-party notices.
- Latest release: **v2.0.3, published 2026-11-22T10:53:37Z**, 290 assets (VERIFIED).
- **Android arm64-v8a** (from `sherpa-onnx-v2.0.3-android.tar.bz2`, 47,988,767 B, all ABIs; measured locally): `libonnxruntime.so` 22,249,560 B + `libsherpa-onnx-c-api.so` 5,055,368 B = **~27.3 MB uncompressed** needed by Rust (the JNI and cxx-api `.so` files aren't needed).
- **Linux x64 shared** (measured locally): `libonnxruntime.so` 27,026,609 + `libsherpa-onnx-c-api.so` 5,795,392 ≈ 32.8 MB. The static-lib archive totals 148 MB of `.a` files before linking; the final binary size after static link is UNVERIFIED. Release archive sizes (VERIFIED): osx-arm64-shared-lib 8,997,173 B; win-x64 static-MT-Release-lib and others are listed in the release.
- **Execution providers** (`sherpa-onnx/csrc/provider.cc`, `session.cc`; VERIFIED): `cpu`, `cuda`, `coreml` (Apple), `xnnpack`, `nnapi` (Android API ≥ 27, falls back to CPU on failure), `trt`, `directml` (Windows only when built with `SHERPA_ONNX_ENABLE_DIRECTML`), `spacemit`, `openvino`.
- **QNN** (Qualcomm HTP NPU) is a separate native path (`csrc/qnn/`). It has Parakeet TDT, Moonshine and Whisper implementations and uses **per-SoC, fixed-length context binaries** (e.g. `…SM8850-binary-parakeet-tdt-0.6b-v2-8s-transducer`, also 3 s…30 s variants). Prebuilt binaries exist only for SM8350/8450/8475/8550/8650/8750/8845/8850 and QCS9100. **No SM8635 (8s Gen 3) binaries** (VERIFIED by listing releases `asr-models-qnn-binary{,-2,-3}`).
- NPU evidence: Parakeet v2 on SM8850 via QNN gives RTF 0.059 on a 7.4 s clip (PR #3720 log; VERIFIED). Conclusion: QNN is out of MVP scope; revisit with a self-compiled SM8635 binary.
- Whether NNAPI speeds up or slows down int8 transducers on the Neo 10R is UNVERIFIED; it's a benchmark item. NNAPI deprecation status on Android 15 is UNVERIFIED.

## Q3 — Exact artifacts, URLs, sizes; punctuation

Base URL: `https://github.com/k2-fsa/sherpa-onnx/releases/download/asr-models/<name>`. GitHub-reported SHA-256 digests are listed below (VERIFIED via the release API). I recomputed the Moonshine tiny digest locally and it matched.

| Artifact | Bytes | sha256 (GitHub `digest`) |
|---|---|---|
| `sherpa-onnx-nemo-parakeet-tdt-0.6b-v2-int8.tar.bz2` | 482,440,461 | `6fc4426b54aa549a0019b8a75b8dea59164e71790a37ac56a045ad57f915f714` |
| `sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8.tar.bz2` | 487,211,443 | `1495e91b59d7fc53ddca91f3431758ec99f8fa253fdb0e1e851be0f72b14b0f9` |
| `sherpa-onnx-nemo-parakeet_tdt_transducer_110m-en-36000-int8.tar.bz2` | 108,035,095 | `f628312e9fdf8686374cb01a69425c41732529d540860311f16f37cbc32cfe9b` |
| `sherpa-onnx-moonshine-tiny-en-quantized-2026-02-27.tar.bz2` | 29,858,559 | `9ec31b342d8fa3240c3b81b8f82e1cf7e3ac467c93ca5a999b741d5887164f8d` (also recomputed locally) |
| `sherpa-onnx-moonshine-base-en-quantized-2026-02-27.tar.bz2` | 111,266,225 | `43232c1d13013d37317163baec3135bd771a186a4356f28c889bab453bb0e891` |
| `sherpa-onnx-whisper-{tiny,base,small}.en.tar.bz2` | 118,071,777 / 208,576,005 / 635,693,775 | null (older uploads have no digest; compute ourselves) |
| `silero_vad.onnx` / `silero_vad.int8.onnx` | 643,854 / 212,860 | `9e2449e1…1fd6` / `c36d490a…3a20` |
| `punctuation-models/sherpa-onnx-online-punct-en-2024-08-06.tar.bz2` | 30,667,839 | null |

- **Release assets are mutable.** The v2-int8 and v3-int8 Parakeet tarballs were re-created on 2026-11-07, under the same names. So pin SHA-256 in our manifest and treat a mismatch as a hard error. Per-file downloads from HF can be pinned to an immutable commit instead: `csukuangfj/sherpa-onnx-nemo-parakeet-tdt-0.6b-v2-int8` revision `84b478d541cdbf0c296a7ab5f1e20a6f8f998114`, URL form `https://huggingface.co/<repo>/resolve/<sha>/<file>`. HF also avoids the bz2 decompression step on device.
- Whisper tarballs contain **both fp32 and int8** files. The app only needs `*-encoder.int8.onnx`, `*-decoder.int8.onnx` and `*-tokens.txt`, so per-file HF downloads save about half.
- Moonshine v2 sherpa packages use `.ort` files (`encoder_model.ort`, `decoder_model_merged.ort`); measured locally from the tarball listing.
- New and unevaluated: `parakeet-tdt-0.6b-ultra-int8` (Moondream post-trained v3) and `parakeet-tdt-0.6b-redux-int8` (1.58-bit v3, 327,703,955 B), both added Nov 2026 (issues/PRs #4161, #4163, #4168). License and quality are UNVERIFIED.
- **Native P&C**: Parakeet v2 ("Punctuations and Capitalizations included", NVIDIA card), v3 (same), 110M ("transcribes speech with Punctuations and Capitalizations"), Moonshine v2 (sherpa sample output "Ask not what your country can do for you. Ask what you can do for your country."), Whisper (P&C by design; SUPPORTED). All VERIFIED except Whisper.
- **Punctuation add-on exists**: `sherpa-onnx-online-punct-en-2024-08-06` (English CNN-BiLSTM, `model.int8.onnx` 7.1 MB, from frankyoujian/Edge-Punct-Casing, Apache-2.0). It adds **punctuation and casing**: "how are you i am fine thank you" becomes "How are you? I am fine. Thank you." Also `punct-ct-transformer-zh-en` (int8 64.7 MB tarball). Both are exposed in the Rust crate. Only needed for models without P&C (e.g. future Indic CTC models). VERIFIED (docs page + GitHub API license).

## Q4 — Weight licenses and duties for a GPL-3.0 app

| Weights | License | Source | Duties / flags |
|---|---|---|---|
| Parakeet TDT 0.6B v2 / v3, TDT-CTC 110M | **CC-BY-4.0**, "ready for commercial/non-commercial use" | HF cards `nvidia/parakeet-tdt-0.6b-v2`, `-v3`, `nvidia/parakeet-tdt_ctc-110m` (VERIFIED) | Attribution: credit NVIDIA, link the license, and **state changes** (sherpa's ONNX export + int8 quantization). Show this in the app's model/about screen. No non-commercial terms. |
| Moonshine **English** (v1 int8 + v2 quantized) | **MIT** | `LICENSE` inside the sherpa v2 tarball, Section 1 (measured locally) | Keep the MIT notice. **Flag:** Section 2, the "Moonshine AI Community License", is **non-commercial** (limited commercial use under US$1 M revenue) and covers *other-language* models. The upstream README (Nov 2026) says only the "legacy non-streaming models for languages other than English" stay under it. **Never ship non-English Moonshine** in a GPL app. |
| Whisper (all sizes) | **MIT** (code and weights) | https://github.com/openai/whisper README: "Whisper's code and model weights are released under the MIT License" (VERIFIED). The HF `openai/*` cards say apache-2.0 (VERIFIED discrepancy; both are permissive). | Keep the notice. |
| Silero VAD | MIT | GitHub API `snakers4/silero-vad` (VERIFIED) | Keep the notice. |
| sherpa-onnx ONNX conversions | Inherit the upstream weight license; sherpa code is Apache-2.0 | — | — |
| IndicConformer 600M multilingual | MIT (card says "released under the MIT license") but **HF-gated (`gated: auto`)** | HF API (VERIFIED) | We may need our own mirror to download without an HF login; MIT allows that. |

- GPL-3.0 interaction: the weights are data downloaded at runtime, not linked code. Our GPL obligations cover our code; the weight licenses impose only attribution (CC-BY / MIT). FSF lists CC-BY-4.0 as GPLv3-compatible (SUPPORTED, from memory; not re-read).
- If we **mirror** the files ourselves, we redistribute them: ship the LICENSE file and attribution alongside.

## Q5 — Performance evidence

Primary-source numbers (VERIFIED unless marked):
- **Parakeet v2 int8 on RK3588 Cortex-A76 (pinned with `taskset`):** RTF 0.220 at 1 thread, 0.142 at 2, 0.118 at 3, 0.088 at 4 threads. Source: https://k2-fsa.github.io/sherpa/onnx/pretrained_models/offline-transducer/nemo-transducer-models.html
  - The Pi 5 uses the same Cortex-A76 core (UNVERIFIED clock/memory parity), so this is the **"Pi 5 floor" proxy**.
  - Extrapolation (UNVERIFIED): the 8s Gen 3's X4 + A720 cores are newer than A76, so expect ≤ RTF 0.09 at 4 threads. That would make a 30 s clip take about 2.6 s and a 90 s clip about 8 s, plus model load. This is exactly what the benchmark must confirm.
- Parakeet v2 int8, unnamed hardware, 2 threads: 7.4 s clip at RTF 0.118; 334 s file **with VAD** at RTF 0.028. Same page.
- Parakeet v3 int8, unnamed hardware, 2 threads: 3.8 s clip at RTF 0.325. Same page; short clips are dominated by fixed overhead.
- Parakeet v2 on SM8850 **NPU (QNN)**: 7.4 s clip in 0.439 s, RTF 0.059 (PR #3720).
- Moonshine v2 tiny-en: RTF 0.056; base-en: RTF 0.109 (2 threads, unnamed hardware, 3.8 s clip). Source: https://k2-fsa.github.io/sherpa/onnx/moonshine/models-v2.html
- Whisper tiny.en on **Raspberry Pi 4**: int8 RTF 0.547 at 1 thread → 0.386 at 4 threads; fp32 0.685 → 0.520. Source: https://k2-fsa.github.io/sherpa/onnx/pretrained_models/whisper/tiny.en.html. Even tiny Whisper is about 4× slower per second of audio than Parakeet 0.6B on an A76-class CPU.
- Desktop, Parakeet int8 via transcribe-rs/ORT (not sherpa): MBP M4 Max ~30× real-time; Zen 3 5700X ~20×; Skylake i5-6500 ~5×; Jetson Nano CPU ~5×. Source: transcribe-rs README (VERIFIED as published claims; methodology not given).
- **Android device reports**:
  - sherpa issue #3913 (user report): the prebuilt `nemo-parakeet-v3-eu` APK (585 MB) "works" on a Snapdragon 680 with 2 GB free RAM, with VAD on 2-minute clips. `moonshine-tiny-en` (135 MB) also works. No timings.
  - soniqo.audio Android benchmark (different runtime, emulator, no NNAPI): Parakeet "v3" on a 1.5 s clip in 175 ms, RTF 0.12. Low value; SUPPORTED only.
- **Peak RAM for Parakeet 0.6B int8: no primary measurement found.** The NVIDIA card says "At least 2GB RAM for model to load" (that refers to the NeMo/PyTorch path). Floor inference (UNVERIFIED): the RSS must at least hold the 652 MB int8 encoder, plus activations. Encoder self-attention memory grows with the square of the frame count, so measure RSS at 90–120 s. **Must be measured on device.**

## Q6 — Silero VAD and input-length limits

- **Whisper:** sherpa **truncates at about 29.5 s.** `offline-recognizer-whisper-impl.h` has `max_num_frames = 3000` and logs "Only waves less than 30 seconds are supported. We process only the first 30 seconds and discard the remaining data" (VERIFIED). So Whisper needs VAD/chunking for our 2–120 s clips.
- **Whisper decoding** also caps output at `max_tokens_per_second` (default 6). That silently truncated Tamil (CER 49.6%) and Hindi (CER 68%) until it was made configurable (#4209, #4211, Nov 2026; option visible in the current impl). VERIFIED.
- **Parakeet:** no length check in `offline-recognizer-transducer-nemo-impl.h` (grep; VERIFIED absence). Upstream supports "up to 24 minutes in a single pass" with full attention on an **A100** (v2/v3 cards). On a phone the limit is memory. Our 120 s maximum is probably fine in a single pass (UNVERIFIED), but it's the main RSS risk.
- **Moonshine v2:** sherpa only skips all-silent input (`kMoonshineSilenceSampleAbsMax`); no explicit maximum found (VERIFIED absence). The practical limit is UNVERIFIED.
- **How sherpa handles long audio:** via VAD. `VoiceActivityDetector` (Silero or TEN) has `min_silence_duration`, `min_speech_duration` and **`max_speech_duration`** (Rust `vad.rs`; VERIFIED). The official long-file path is "vad-with-offline-asr", which decodes each segment separately (Parakeet: 334 s at RTF 0.028).
- **Recommendation (decide + report):**
  1. Always run Silero VAD (int8, 213 KB) on the finished clip. Trimming leading and trailing silence and dropping long pauses cuts decode time and prevents hallucinated text on silent clips.
  2. Parakeet: decode the speech-only audio in **one pass if ≤ 60 s** (best punctuation context). Otherwise split at VAD silences into segments ≤ about 30 s (`max_speech_duration`) to bound RSS, then join the texts with a space.
  3. The thresholds are benchmark-driven (see the protocol). Segment joins can produce stray periods or capitals at boundaries, so the WER/formatting check should cover the 90 s clip.
  4. Whisper (if ever used) must chunk to ≤ 28 s.

## Q7 — FOSS prior art

| Project | Runtime | Models | License | Reusable under GPL-3.0? |
|---|---|---|---|---|
| **Handy** (cjpais/Handy, 39.8k★, active) | Tauri + Rust; now `transcribe-cpp` 0.4.2 (ggml/GGUF); Silero VAD | Whisper GGML (small/medium/turbo/large), Parakeet (unified-en 0.6B GGUF Q8_0 731 MB; README lists Parakeet V3) | **MIT** | Yes. MIT code can be lifted into a GPL app with notice: push-to-talk flow, VAD use, model-manager UX, updater signing. Desktop only (VERIFIED). |
| **FUTO Voice Input** (gitlab.futo.org/keyboard/voiceinput; GitHub mirror) | whisper.cpp (TFLite in the past) | Whisper, Parakeet, "asr4all" | **FUTO Source First License 1.0** (source-available, **not OSI/FSF-free**) | **No.** Don't copy code; ideas only (VERIFIED license header). |
| **WhisperIME** (woheller69/whisperIME) | TensorFlow Lite 2.17 (`com.whispertflite`) | Whisper (~435 MB download from HF) | MIT | Yes (MIT), but TFLite/Whisper only. Useful as an Android IME/RecognitionService reference (VERIFIED). |
| **Transcribro** (soupslurpr/Transcribro) | whisper.cpp + Silero VAD | Whisper | **ISC** | Yes (permissive). Android keyboard + RecognitionService reference (VERIFIED). |
| **VoiceInk** (Beingpax/VoiceInk, macOS) | whisper.cpp; FluidAudio (Parakeet, CoreML); TranscribeCpp-swift | Whisper, Parakeet, SenseVoice | **GPL-3.0** | Yes, same license. Swift/macOS only, so mostly UX reference (VERIFIED). |
| whisper-to-input (j3soon) | remote Whisper API | Whisper | GPL-3.0 | Android keyboard reference only (VERIFIED license). |
| sherpa-onnx's own apps | Kotlin + sherpa-onnx; Tauri desktop ASR examples (`tauri-examples/non-streaming-speech-recognition-from-microphone`) | 62 models | Apache-2.0 | Yes. **Closest reference for our exact stack** (VERIFIED). |

## Q8 — Indian languages (final ticket)

- **Parakeet v3:** 25 European languages; **no Hindi or Tamil** (HF card `language` list; VERIFIED).
- **Whisper multilingual** (tiny/base/small `sherpa-onnx-whisper-{size}` tarballs, 116 MB / 208 MB / 639 MB): the card lists `hi` and `ta` (VERIFIED). Quality is poor at small sizes: Whisper Small gives Tamil CER 18.7% and Hindi 45.1% even after the token-cap workaround (#4209). Requires sherpa ≥ the #4211 fix plus `max_tokens_per_second` tuning. MIT.
- **AI4Bharat IndicConformer 600M multilingual:** 22 official Indian languages, hybrid CTC + RNNT, MIT, HF-gated. It already ships as **ONNX**, but in a custom layout: a shared encoder, `ctc_decoder.onnx`, and per-language `joint_post_net_{hi,ta,…}.onnx` heads (VERIFIED file listing). That is **not** a sherpa-onnx format; no sherpa export exists (no "indic" in the asr-models release; VERIFIED). Supporting it means a sherpa export script (NeMo CTC is a supported family, UNVERIFIED fit) or a custom ORT path. The card now points to "Indic-Transcribe" (bodhan-ai, **license "other", gated**; terms UNVERIFIED). Output punctuation is UNVERIFIED; plan on the punctuation add-on or none.
- **IndicWhisper / Vistaar** (AI4Bharat/vistaar, MIT repo): benchmarks + fine-tuned Whisper. Weight license and an ONNX/sherpa export are UNVERIFIED. A whisper-architecture fine-tune should export with sherpa's Whisper scripts (UNVERIFIED).
- **Meta Omnilingual ASR** (1600 languages, includes hi/ta per name; per-language quality UNVERIFIED) **is** packaged by sherpa: `omnilingual-asr-1600-languages-300M-ctc-v2-int8` (292,313,120 B). Apache-2.0 (upstream README; VERIFIED). CTC output, punctuation UNVERIFIED. Supported by the Rust `omnilingual` config. **Best first candidate for the Indic ticket**, then compare against IndicConformer.

## Needs on-device benchmark — iQOO Neo 10R protocol

Goal: pick the default and fallback from measured data, not from extrapolation.

**Setup**
- Device: iQOO Neo 10R. Record the SoC from `adb shell getprop ro.soc.model` (expect SM8635), the RAM SKU, the Android version and the build number. Charge ≥ 80%, airplane mode, screen on, device cool (rest 2 min between runs), no battery saver.
- Binary: a Rust benchmark CLI built against `sherpa-onnx` 2.0.3 (`--target aarch64-linux-android`, shared), pushed with `adb` to `/data/local/tmp` alongside `libsherpa-onnx-c-api.so` and `libonnxruntime.so`. Same code path as the app. Also run once inside the Tauri app to confirm parity.
- Models: Parakeet v2 int8 (default candidate), Parakeet 110M int8, Moonshine v2 tiny-en and base-en; Whisper base.en int8 as a reference for clips ≤ 28 s only.
- Providers: `cpu` at num_threads 1/2/4/6; `nnapi` once at the best thread count.

**Clips** (16 kHz mono PCM, kept in `fixtures/asr/`, SHA-256 pinned)
- 5 s, 30 s, 90 s of natural dictation (chat/email style: numbers, names, a question, a list), plus 120 s (maximum) and 10 s of silence (hallucination check).
- Fixed English WER set: 20 utterances (~5 min total). Mix: 10 LibriSpeech test-clean (CC-BY-4.0), 5 self-recorded Indian-English dictation, 5 noisy self-recordings (fan/street). Reference transcripts with punctuation stored next to the audio.

**Metrics per (model, provider, threads, clip)**, 1 warm-up + 5 measured runs, report the median and max
1. **Cold load** time: process start → recognizer created (model mmap/load).
2. **✓ → text**: wall time from the end of audio input → final string, *including* VAD trim and any segmenting, with the recognizer already loaded. Also a "first use" variant that includes load.
3. RTF = decode time / speech duration.
4. **Peak RSS**: `VmHWM` from `/proc/<pid>/status` sampled after the run, plus `dumpsys meminfo <pid>` for the app run.
5. WER (normalized: lowercase, no punctuation) and **formatted WER** (case + punctuation kept), computed with one fixed script; plus a manual 1–5 "send-ready" rating on the 30 s/90 s clips.
6. VAD comparison for Parakeet on 90 s/120 s: one pass vs `max_speech_duration` 30 s segments (time, RSS, formatted WER).
7. Thermal: rerun the 90 s clip 10× back to back; report the drift.

**Pass bar (proposed; confirm with the user)**
- ✓→text: ≤ 1.5 s for 5 s, ≤ 3 s for 30 s, ≤ 8 s for 90 s.
- Peak RSS: ≤ 1.2 GB.
- Normalized WER: ≤ 8% on the fixed set.
- Silence clip returns empty text.

Default = the most accurate model that passes. If Parakeet v2 fails on RSS or time, fall back to 110M; if that fails, Moonshine.

**Record** results in `research/speech-benchmark-neo10r.md` with the raw CSV, git SHA, model SHA-256 and the device build.

## Open / unverified (carry to KNOWN-GAPS when shipping)

- No primary RTF or RSS numbers for any of these models on Snapdragon 8-series CPUs; all on-device latency figures above are extrapolated.
- Moonshine v2 parameter counts and its practical maximum input length.
- Whether NNAPI helps int8 models on SM8635.
- Ultra/Redux Parakeet licenses and quality.
- Indic-Transcribe terms.
- IndicConformer → sherpa export feasibility.
- FSF CC-BY-4.0 compatibility statement (not re-read).
- iQOO Neo 10R SoC: SUPPORTED only (Technave/digit.in/GSMArena listings via web search; confirm with `getprop`).
