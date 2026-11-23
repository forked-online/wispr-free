# Wispr Free

**Speak instead of typing. Keep your Android keyboard.**

Wispr Free is a planned on-device dictation app. Focus a supported text field, tap the floating mic, speak, then tap ✓ to transcribe and insert your draft. Download and switch between speech models; dictation is designed to work offline once a model is installed.

## Status

This repository contains a landing page and product spec. There is no Android build or installable APK yet. The first release will be a sideloaded APK for testers; a Google Play launch comes later, subject to policy review.

## Preview

Open [`index.html`](index.html) in a browser. It works directly from disk without dependencies, a build step, or microphone access. The phone demo is a simulation: scrolling the "How" story drives it, and you can also tap the field, drag the floating mic bubble, record, and insert sample text. Rejected style explorations live in [`prototypes/`](prototypes/).

## Product

See [`PRODUCT.md`](PRODUCT.md) for the MVP scope, permission flow, compatibility limits, and release gates. Parakeet is a candidate, not a confirmed model. The first engineering milestone is validating recording and safe text insertion on an Android phone before committing to a speech runtime.
