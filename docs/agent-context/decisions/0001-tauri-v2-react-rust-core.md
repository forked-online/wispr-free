# 0001 — Tauri v2 + React UI + Rust core

- **Status:** accepted 2026-11-23 (user decision)
- **Context:** Multi-device goal: Android first, then Windows/macOS/Linux (iOS considered). User wants to build Rust skills.
- **Decision:** One Tauri v2 app. React for UI (settings, onboarding, model library). Rust core for shared logic (model manifest/download/checksum, audio buffer, ASR via sherpa-onnx, transcript post-processing, insertion-splice rules). Platform-only parts stay native: Android AccessibilityService + overlay + AudioRecord + insertion in Kotlin (Tauri plugin / generated Android project); desktop hotkey + insertion in Rust.
- **Rejected:** native-Kotlin-first (fastest MVP, but duplicates logic later); React + Capacitor + Rust (two bridges, no first-class desktop).
- **Known consequences (to verify in research/):**
  - The dictation loop runs from the AccessibilityService, which can be alive without the Tauri WebView, so Kotlin → Rust is a direct JNI/UniFFI call, not Tauri IPC.
  - iOS cannot host this UX (no cross-app overlay/insertion; keyboard extensions cannot use the mic). iOS = separate keyboard product, out of MVP.
- **Unchanged:** the first on-device spike (bubble mic + safe insertion on iQOO Neo 10R) is stack-independent and comes first.
