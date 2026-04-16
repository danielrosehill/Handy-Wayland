# 02 — Start/stop audio feedback is unreliable; need a distinctive start beep

## Motivation

Audio cues are the primary "mic is live" signal when Handy's overlay isn't visible (focused app, fullscreen, external monitor). Currently:

- The start and stop sounds sometimes don't play at all.
- When they do play, they aren't distinctive enough — Daniel wants an unambiguous "recording open" beep so there's no doubt the mic is hot.

Missing the start cue is the worst failure — you dictate into silence, then discover the mic was never armed.

## Hypothesis — where this lives

- `src-tauri/src/audio_feedback.rs` — `resolve_sound_path` + `SoundType::{Start,Stop}`. Playback is spawned on a thread with `rodio` + `cpal`. A few failure modes are visible even at a glance:
  - `OutputStreamBuilder` is re-created per playback — if the default output device is in a transitional state (PipeWire routing change, Bluetooth sink just woke, HDMI device appearing/disappearing) the builder can fail silently and the thread exits with no user-visible signal.
  - The thread is fire-and-forget: no retry, no fallback to a different sink, no log escalation beyond `warn!`.
  - Timing — if the sound thread races with the recording-start path, the sound can be cut off or swallowed when the mic capture claims the audio graph.
- `settings.rs` — `SoundTheme` enum and built-in theme assets (`to_start_path` / `to_stop_path`). A "highly distinctive" theme is a content problem, not a code problem.

## Things to verify first

1. Reproduce the dropout. Capture `RUST_LOG=debug` output around a failed trigger — does `audio_feedback.rs` log anything?
2. Check whether the default sink reported by `pw-cli info 0` at the moment of failure matches what `cpal::default_host().default_output_device()` returns.
3. Confirm the WAV file resolves — missing-asset path would produce a silent failure on "Custom" theme.

## Approach options

- **A — resilient playback.** Log at `error!` on every failure path; retry once with a 50ms delay; fall back to a compiled-in embedded beep if the configured theme file fails to load.
- **B — pre-opened sink.** Keep a long-lived `rodio::OutputStream` around instead of rebuilding per sound, so PipeWire routing churn doesn't kill individual plays.
- **C — dedicated "recording armed" beep.** Add a new `SoundTheme::DistinctiveStart` (or a standalone setting) that ships a short two-tone ascending chirp — unmistakable, ~150ms, peak loudness ~-6dBFS. The stop sound stays mellower.
- **D — priority ordering.** Ensure the start beep is emitted *before* the audio-capture stream is opened, not after — avoids the device-contention race.

Recommended combination: A + B + C + D. All four are small, all four compound.

## Risks

- Keeping an `OutputStream` alive holds a reference to the PipeWire sink; on sink removal it needs to be recreated. Wrap in a lazy/reset pattern rather than a single `OnceCell`.
- A louder/more distinctive beep should be user-overridable (already is via Custom theme).

## Definition of done

- `audio_feedback` never fails silently — every failure path produces a log line and triggers the embedded-beep fallback.
- New distinctive start sound available as a theme option and set as default for fresh installs.
- Manual test matrix: default PipeWire sink, Bluetooth sink, HDMI sink, sink switched mid-session.
