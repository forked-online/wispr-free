# 0003 — Speech engine and models

- **Status:** accepted 2026-11-23. The RAM thresholds are filled in by the BENCH spike.
- **Evidence:** `research/speech-stack.md`. The crate version was re-verified on crates.io on 2026-11-23.

## Decision
- **Engine:** official `sherpa-onnx` Rust crate (2.0.3, Apache-2.0, k2-fsa). Offline (non-streaming) recognizer; the whole clip is transcribed after ✓.
- **English model tiers** (same NeMo transducer code path for both Parakeet models):

| Tier | Model | Download | License | Notes |
|---|---|---|---|---|
| high | Parakeet TDT 0.6B v2 int8 | ~482 MB (~661 MB on disk) | CC-BY-4.0 | Punctuation and casing built in |
| standard | Parakeet 110M int8 | ~108 MB | CC-BY-4.0 (verify on its card) | Same code path |
| lite | Moonshine v2 tiny-en | ~30 MB | MIT | Ultra-light; check punctuation output in BENCH |

- **Default selection (user decision):** at onboarding the app *recommends* a tier from the device's total RAM plus the BENCH results, and the user can override it. The thresholds come from measured peak RSS (BENCH), not guesses. Until BENCH runs, `recommend_model()` has no thresholds, and the plan blocks that ticket on BENCH.
- **Pass bar on the iQOO Neo 10R (user decision):** for a 30 s clip, ✓ to text in ≤ 3 s, peak RAM ≤ 1.2 GB, WER ≤ 8% on the fixed 20-clip English set. A tier is offered as "recommended" on a device class only if it passes.
- **Whisper:** not a default, because sherpa-onnx truncates input at about 30 s.
- **Long clips:** cap recording at 2 min. Whether long clips are transcribed in one pass or split with Silero VAD is decided by the BENCH comparison.
- **Integrity:** every manifest entry pins a Hugging Face **commit hash** URL plus SHA-256 and byte size. Never pin by GitHub release tag name: the Parakeet int8 tarballs were re-uploaded under the same names on 2026-11-07.
- **Supply chain:** the crate's build script downloads native libraries without checksum verification. Vendor or verify them in CI (KNOWN-GAPS).
- **Acceleration:** CPU-only for the MVP. NNAPI is measured in BENCH as information only; there are no QNN builds for SM8635.

## Indian languages (final ticket)
First candidate: Meta Omnilingual 300M int8 (Apache-2.0; sherpa-onnx packages it). Parakeet v3 covers neither Hindi nor Tamil. IndicConformer's ONNX layout can't be loaded by sherpa-onnx. Never use non-English Moonshine (non-commercial license).
