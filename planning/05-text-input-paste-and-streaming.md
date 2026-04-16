# 05 — Paste-at-cursor + streaming incremental transcription

## Motivation

Two related problems with text delivery:

**Problem A — simulated typing is slow for long transcripts.** Handy's current text input appears to synthesise keystrokes one character at a time (enigo-based). For a short phrase this is fine; for a paragraph it's a visible keystroke storm that takes seconds to finish and blocks cursor navigation. For long dictations, the right behaviour is to **dump the whole transcript at the cursor position in one shot**.

**Problem B — no streaming text appearance.** When dictating a long passage, the user stares at nothing until the recording stops and the full transcript arrives. Parakeet-TDT supports streaming / chunked inference natively — Handy could transcribe in ~15–20s windows and emit text incrementally so the user sees it appear as they speak.

Solving A addresses "delivery speed". Solving B addresses "feedback latency". They compose: streaming transcription would emit text chunks, each of which is delivered via paste-at-cursor (never simulated typing).

## Hypothesis — where this lives

### For Problem A (paste vs type)

- `src-tauri/src/input.rs` — already contains `send_paste_ctrl_v` and `send_paste_ctrl_shift_v` for enigo-based paste, but the comment "On Wayland, this may not work" suggests the Wayland branch falls back to simulated typing. Wayland paste needs `wl-copy` + `ydotool` (compositor-synthesised keycodes), as proven by the `handy-paste.sh` workaround in `Handy-Bugs-And-Ideas`.
- `settings.rs` — there's a "typing tool" setting (Auto / ydotool / wtype / xdotool / Clipboard). This plan assumes **Clipboard-paste should be the default on KDE Wayland** and should work reliably.

### For Problem B (streaming)

- `src-tauri/src/transcription_coordinator.rs` — orchestration of recording → transcription → injection. Currently probably a one-shot pipeline that waits for end-of-recording.
- `src-tauri/src/managers/transcription.rs` — model inference. Parakeet-TDT supports chunked decoding; the `nemo`/`parakeet` bindings expose it. Check whether Handy's Parakeet wrapper exposes incremental output or only a final transcript.

## Approach

### Phase 1 — paste-at-cursor as the default for long text (Problem A)

1. Make "Clipboard paste" the default Typing Tool on KDE Wayland, with a native pipeline: `wl-clipboard` (`wl-copy --foreground`, owner-alive) + `ydotool type --key-delay 0` for the Ctrl+V, or use a uinput device directly. Reference impl: `handy-paste.sh` in the workarounds folder.
2. Add a threshold: transcripts above N characters (default 40) always use paste; below threshold keep current behaviour. (Short phrases inside IME-aware fields still benefit from simulated typing in some apps — keep it as an escape hatch.)
3. Preserve clipboard contents: snapshot the existing selection/clipboard, paste the transcript, restore the original clipboard after a short delay.

### Phase 2 — incremental streaming transcription (Problem B)

1. Verify Parakeet-TDT streaming support in the inference backend Handy uses.
2. Introduce a `StreamingTranscriber` trait with two impls: `BatchTranscriber` (current behaviour) and `StreamingTranscriber` (chunked, ~20s windows with 2s overlap).
3. On each chunk boundary, emit a `TranscriptChunk { text, is_final }` event. `is_final: false` means "this chunk may be revised"; `is_final: true` means it's committed.
4. Delivery model: **append-only**. Only commit `is_final: true` text to the cursor via paste. Non-final chunks update the overlay so the user sees preview text but nothing touches their editor until finalised. This avoids the "text you see on screen getting deleted and rewritten" problem.
5. Alternative delivery model — **live edit**: use IME protocol (`zwp_text_input_v3`) to stream preedit text directly into the target app. Elegant but coverage is patchy (see watch-list item 5). Ship as opt-in, not default.

### Phase 3 — settings UX

Add settings:

- `Long-transcript delivery`: "Paste (default)" | "Simulated typing" | "IME preedit (experimental)"
- `Enable streaming transcription`: off by default, on for Parakeet-TDT
- `Streaming chunk size`: 15s / 20s / 30s

## Risks

- Paste restoration races: if the user Ctrl+C's something between our paste and our restore, we'd clobber it. Mitigation: restore only if clipboard owner hasn't changed.
- Parakeet streaming may require a different model export format (CTC vs TDT variants). Verify before committing to Phase 2.
- Overlapping chunk boundaries produce duplicate text at seams; need a diff-based commit strategy.
- IME preedit path is a rabbit hole — keep behind a feature flag and don't block the main plan on it.

## Definition of done

- Phase 1: default clipboard paste on KDE Wayland is reliable (ties to item 03 subitem 2) and long transcripts no longer character-by-character-type.
- Phase 2: streaming Parakeet transcription visible in the overlay with final-chunk commit to the editor; off by default, togglable in settings.
- Phase 3: settings expose the two delivery dimensions clearly; defaults are sane for KDE Wayland + Parakeet.
