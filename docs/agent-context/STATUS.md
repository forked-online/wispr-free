# STATUS — Wispr Free

## Current state
- Landing page (style C, floating-overlay demo) + `PRODUCT.md` spec pushed to `origin/main` (`c864720`).
- Research done; decisions 0001–0005 accepted; end-to-end plan in `MVP-PLAN.md`. No app code yet.

## Next step
Start the three parallel tracks from the pinned epic #52 (GitHub issues #1–#51): Track B phone tests (T00 → T10…T15 Gate 1), Track A Rust learning (T02 → T20…), Track C infra (T01, T03 dependency approval → T04, T05).

## Ledger
- 2026-11-23 — User decisions: first test device **iQOO Neo 10R**; MVP language **English only**, Indian languages as the **final ticket**; license **GPL-3.0**; name **"Wispr Free" kept** (user accepts the trademark risk of being close to "Wispr Flow"). Proposed stack (pending verification): native Kotlin, sherpa-onnx, push-to-talk non-streaming ASR, ACTION_SET_TEXT splice insertion, Copy fallback, pinned-checksum model downloads, arm64 only.
- 2026-11-23 — New requirement: multi-device. Android first; Windows/macOS/Linux later. Stack impact under discussion (see chat recommendation: keep Android native Kotlin; share engine (sherpa-onnx), model manifest schema and golden test fixtures as platform-neutral contracts; defer a shared code core until desktop starts). Research agents were stopped before running.
- 2026-11-23 — Stack decided: Tauri v2 + React + Rust core → `decisions/0001-tauri-v2-react-rust-core.md`. Research launched: android-platform, speech-stack, tauri-architecture, distribution-device (files in `research/`).
- 2026-11-23 — Research complete (4 files in `research/`). Main session re-verified at source: crates.io `sherpa-onnx` 2.0.3 (2026-11-22, Apache-2.0, k2-fsa repo); tauri issue #15671 open; IzzyOnDroid policy text ("Vibe-coded apps will be rejected", code "should be free of" LLM text); AOSP `android.accessibilityservice.InputMethod` (top-level class, not nested) with `getCurrentInputConnection()` + `commitText`, and `AccessibilityServiceInfo.FLAG_INPUT_METHOD_EDITOR`. Correction: the Android 17 accessibility revocation applies only under opt-in Advanced Protection (low risk). Pending user sign-off: benchmark pass bar, default model size, distribution order.
- 2026-11-23 — User decisions: model tier **recommended at onboarding by device RAM** (user can override); BENCH pass bar **30 s clip ≤ 3 s, ≤ 1.2 GB peak, ≤ 8% WER**; distribution **GitHub → F-Droid → Play**, IzzyOnDroid skipped. Wrote decisions 0002–0005, `MVP-PLAN.md`, `KNOWN-GAPS.md`, agent-context README(.html).
- 2026-11-23 — Broke the MVP into 51 tickets (`tickets/T00`–`T80`, index + dependency graph + Rust learning order in `tickets/README.md`). Ticket headers validated by script: every dependency id exists.
- 2026-11-23 — Tickets migrated to GitHub issues #1–#51 (milestones per phase, 94 `blocked by` links, labels) under pinned epic #52 (https://github.com/forked-online/wispr-free/issues/52); `tickets/` removed from the repo. Issues are now the source of truth for ticket scope/progress.
- 2026-11-24 — T01 (#2) done on `chore/t01-license`, PR #53 (open): GPL-3.0 `LICENSE` downloaded verbatim from gnu.org (35149 bytes, sha256 `3972dc9744f6499f0f9b2dbf76696f2ae7ad8af9b23dde66d6af86c9dfb36986`, matches a second fresh GNU mirror fetch), `NOTICE.md` stub (three sections; filled later by T65/#45 and T22/#15), README License section.
