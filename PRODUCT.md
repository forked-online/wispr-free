# Wispr Free — Android product spec

**Status:** Proposed; the landing page is a concept preview. First release: signed, sideloaded Android APK. Google Play later, pending review.

## Promise

Dictate into supported Android text fields without replacing the user’s keyboard or uploading speech for transcription. Once a model is downloaded, the dictation loop works offline. No account required.

## MVP experience

1. **Set up.** Explain why Accessibility access is needed to detect the active field and insert text. Guide the user to enable it in Android Settings. Offer a model download, then request microphone access on the first recording attempt.
2. **Start.** Show a non-focus-stealing floating mic only for supported editable fields. Hide it for password fields and when no eligible field is focused. Recording starts only after an explicit tap.
3. **Record.** Expand into a pill with an elapsed timer, a live level visualizer, Cancel, and ✓. Cancel discards the recording. Confirm stops microphone capture immediately and shows **Transcribing…**.
4. **Insert.** Transcribe locally, then revalidate the original field, cursor or selection, and draft before inserting. Preserve text outside the selection. Never submit, send, or write into a different field. If validation or insertion fails, show the transcript with an explicit **Copy** action.

## Models and privacy

- Start with two tested options: a Parakeet candidate and a smaller fallback. Select the default after on-device testing. Show model download size, supported languages, active state, and license. Let users keep multiple downloaded models, switch between them offline, and delete models they no longer need.
- Verify the exact model artifact, runtime license, redistribution terms, and download integrity before listing a model. A failed download or switch must not disable a working model.
- Keep audio and drafts on-device. Do not log field contents or retain recordings or transcript history by default. Use the clipboard only when the user explicitly taps Copy.

## Proof before release

| Gate | What to verify |
| --- | --- |
| Overlay and permissions | Prototype an Accessibility Service overlay first, without a separate “Display over other apps” grant. Show it without stealing input focus; confirm that a tap actually starts microphone capture while the editor app is foregrounded. Check permission denial, revocation, and Android’s sideloaded Accessibility restrictions on the target device. |
| Safe insertion | Insert a test phrase in WhatsApp, Gmail, and Chrome using Gboard or the installed keyboard. Test insertion in the middle of an existing draft, replacing a selection, changing focus, and editing the draft during transcription. Verify that incompatible editors get Copy instead of a risky write. |
| Offline model performance | In airplane mode, test short messages and longer dictation with each candidate. Record device/OS, model and artifact size, peak memory, and time from ✓ to inserted text. Use those results to pick support and performance targets. |
| Sideloaded test APK | Publish a signed build with its checksum, tested Android versions/devices, install and permission instructions, and known unsupported cases. Add a real download link only when the build exists. |
| Google Play | Review the current Accessibility API, disclosure/consent, and distribution requirements before submission. Validate the proposed use and complete any required declarations. Do not assume sideload compatibility guarantees approval. |

Android references: [microphone foreground-service restrictions](https://developer.android.com/develop/background-work/services/fgs/restrictions-bg-start#wiu-restrictions) and [Google Play Accessibility API policy](https://support.google.com/googleplay/android-developer/answer/10964491).

## Not in v1

Universal editor compatibility, replacement keyboards, cloud transcription, automatic sending, and local AI rewriting. Punctuated dictation is the target; rewriting can be evaluated separately for accuracy and latency.
