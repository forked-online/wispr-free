# 0004 — Text insertion strategy

- **Status:** accepted 2026-11-23. The app compatibility matrix is filled in by spikes AP-S5 (commitText) and AP-S6 (SET_TEXT).
- **Evidence:** `research/android-platform.md`; AOSP `android.accessibilityservice.InputMethod`, `getCurrentInputConnection()`, `commitText`, and `AccessibilityServiceInfo.FLAG_INPUT_METHOD_EDITOR` were verified in the AOSP mirror on 2026-11-23.

## Decision
At record start, snapshot the focused `AccessibilityNodeInfo` (identity, text, selection, `isShowingHintText`). After transcription, call `refresh()` and decide in this order:

1. **Revalidate:** `node.equals(snapshot)` (node id + window id; never the hidden `getSourceNodeId`), still focused, still editable, text and selection unchanged. If any check fails, use **Copy fallback**.
2. **Primary, API 33+:** `InputMethod.getCurrentInputConnection()?.commitText(spacedTranscript, 1, null)` inserts at the cursor or replaces the selection and keeps formatting.
3. **Secondary:** `ACTION_SET_TEXT` with the full spliced text, passed as a **plain `String`** (Compose reads it with `getString`; a non-String `CharSequence` clears the field). Then `ACTION_SET_SELECTION` to put the cursor after the insert. Treat the text as empty when `isShowingHintText()` is true.
4. **Fallback:** a transcript sheet with a Copy button. The clipboard is written only on an explicit tap.

## Invariants (tests enforce these)
- Never write to a field other than the snapshotted one. Never send or submit.
- Spacing and punctuation joins (transcript vs. neighbouring text) are computed in `wispr-core` and shared with desktop.
- Android selection offsets are **UTF-16 code units**. The core must bounds-check them and reject any offset that splits a surrogate pair. Rejected input goes to the fallback; it never panics.
- No field content is logged, ever.
