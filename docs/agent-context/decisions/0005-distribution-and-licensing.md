# 0005 — Distribution, licensing, name

- **Status:** accepted 2026-11-23 (user decisions + research)
- **Evidence:** `research/distribution-device.md`. IzzyOnDroid policy text verified at izzyondroid.org on 2026-11-23.

## Decision
- **License:** GPL-3.0 for the app. Apache-2.0 (sherpa-onnx), MIT (Tauri parts, Handy) and CC-BY-4.0 (Parakeet weights) are compatible. An in-app licenses screen lists every component and model, and credits Parakeet as "NVIDIA Parakeet, CC-BY-4.0, converted to ONNX and int8-quantized by k2-fsa/sherpa-onnx".
- **Release order:** (1) signed APK on GitHub Releases with the APK SHA-256 and signing-cert fingerprint; Obtainium for updates. (2) F-Droid via a reproducible build that publishes our signed APK; expect the `NonFreeNet` anti-feature because models are downloaded. (3) Google Play: upload the same key through Play App Signing, add an in-app accessibility disclosure and consent, and submit the declaration form. Play review is a known risk because a general dictation tool is not `isAccessibilityTool`.
- **IzzyOnDroid: not targeted.** Its policy says "Vibe-coded apps will be rejected" and app code "should be free of" LLM-generated text.
- **One release key** used everywhere, from the first public APK on. Register the package name and key fingerprint for Android developer verification (global in 2027).
- **Native libs:** 16 KB page-aligned (the sherpa-onnx prebuilts are; build with NDK r28+). arm64-v8a only.
- **Name:** "Wispr Free" kept by the user, who accepts the trademark risk of its closeness to "Wispr Flow". Store listings may flag confusingly similar names; this is a known risk, not a blocker.
- **iOS:** out of v1. iOS keyboards have no mic access, and App Review 4.4.1 makes Wispr Flow's "jump to the app" workaround risky.
- **Never reuse:** FUTO Voice Input code (not free software), non-English Moonshine (non-commercial).
