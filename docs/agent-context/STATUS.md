# STATUS — Wispr Free

## Current state
- Landing page (style C, floating-overlay demo) + `PRODUCT.md` spec pushed to `origin/main` (`c864720`).
- Research done; decisions 0001–0005 accepted; end-to-end plan in `MVP-PLAN.md`. No app code yet.

## Next step
P0 repo scaffold and P1 device spikes on the iQOO Neo 10R, in parallel (`MVP-PLAN.md`). Gate 1 (mic + insertion + survival) decides go/no-go before P3.

## Ledger
- 2026-11-23 — User decisions: first test device **iQOO Neo 10R**; MVP language **English only**, Indian languages as the **final ticket**; license **GPL-3.0**; name **"Wispr Free" kept** (user accepts the trademark risk of being close to "Wispr Flow"). Proposed stack (pending verification): native Kotlin, sherpa-onnx, push-to-talk non-streaming ASR, ACTION_SET_TEXT splice insertion, Copy fallback, pinned-checksum model downloads, arm64 only.
- 2026-11-23 — New requirement: multi-device. Android first; Windows/macOS/Linux later. Stack impact under discussion (see chat recommendation: keep Android native Kotlin; share engine (sherpa-onnx), model manifest schema and golden test fixtures as platform-neutral contracts; defer a shared code core until desktop starts). Research agents were stopped before running.
- 2026-11-23 — Stack decided: Tauri v2 + React + Rust core → `decisions/0001-tauri-v2-react-rust-core.md`. Research launched: android-platform, speech-stack, tauri-architecture, distribution-device (files in `research/`).
- 2026-11-23 — Research complete (4 files in `research/`). Main session re-verified at source: crates.io `sherpa-onnx` 2.0.3 (2026-11-22, Apache-2.0, k2-fsa repo); tauri issue #15671 open; IzzyOnDroid policy text ("Vibe-coded apps will be rejected", code "should be free of" LLM text); AOSP `android.accessibilityservice.InputMethod` (top-level class, not nested) with `getCurrentInputConnection()` + `commitText`, and `AccessibilityServiceInfo.FLAG_INPUT_METHOD_EDITOR`. Correction: the Android 17 accessibility revocation applies only under opt-in Advanced Protection (low risk). Pending user sign-off: benchmark pass bar, default model size, distribution order.
- 2026-11-23 — User decisions: model tier **recommended at onboarding by device RAM** (user can override); BENCH pass bar **30 s clip ≤ 3 s, ≤ 1.2 GB peak, ≤ 8% WER**; distribution **GitHub → F-Droid → Play**, IzzyOnDroid skipped. Wrote decisions 0002–0005, `MVP-PLAN.md`, `KNOWN-GAPS.md`, agent-context README(.html).
