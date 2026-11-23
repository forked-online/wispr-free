# KNOWN GAPS

Live list. Remove an entry once it is verified or fixed.

| Gap | Why it matters | Resolves via | Evidence |
|---|---|---|---|
| Background mic from the a11y service is unproven (docs list no exemption; AOSP source suggests one) | Core UX | AP-S1/AP-S2 on device | `research/android-platform.md` |
| vivo/OriginOS may disable the a11y service (one vendor reports ≤5 h) | Bubble disappears | SOAK + onboarding whitelist | `research/distribution-device.md` |
| Device facts (Android 16/OriginOS 6, SM8635, 16 KB pages) come from spec sites only | Build targets | `adb getprop` at P1 start | `research/distribution-device.md` |
| No published peak-RAM figure for Parakeet 0.6B int8 | Tier thresholds | BENCH | `research/speech-stack.md` |
| sherpa-onnx crate build script downloads native libs without a checksum | Supply chain | Vendor or verify in CI (P0.3/P3.2) | `research/speech-stack.md` |
| Parakeet release tarballs were re-uploaded under the same names (2026-11-07) | Pinned hashes break | Pin HF commit URLs (0003) | `research/speech-stack.md` |
| Play review of a non-`isAccessibilityTool` dictation service | Play launch | Deferred; F-Droid/GitHub first | `decisions/0005` |
| Trademark status of "Wispr"/"Wispr Flow" seen only via aggregators | Name risk (accepted by user) | None planned | `research/distribution-device.md` |
| Landing page model sizes/params predate research (0.6B / 39M Whisper / 61M Moonshine shown) | Page should match 0003 tiers | Update `index.html` model rows | `index.html`, `decisions/0003` |
