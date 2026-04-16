# 06 — VAD-driven hands-free recording mode

**Priority: deferred / nice-to-have.** Not part of the initial implementation wave — captured here so it's not lost. Revisit after items 01–05 are in.

## Motivation

Handy already has Silero VAD in the audio pipeline (`src-tauri/src/audio_toolkit/vad/`) — it's used internally to trim silence and gate frames through the transcription pipeline. What's **not** exposed is a user-facing **voice-activated recording mode**: start capture automatically when speech is detected, stop automatically on a silence threshold, no hotkey required.

For long-form dictation this is a big quality-of-life win — you can pause to think mid-sentence without holding a key, and you don't need to reach for a hotkey to start. PTT remains the right tool for short bursts; VAD mode is better for drafting.

## Current state

- `audio_toolkit/vad/silero.rs` — Silero VAD implementation (ONNX, `silero_vad_v4.onnx` downloaded at build).
- `audio_toolkit/vad/smoothed.rs` — `SmoothedVad` adds hysteresis (prefill/hangover frames) on top of raw Silero.
- `managers/audio.rs` — constructs `SmoothedVad::new(Box::new(silero), 15, 15, 2)` and attaches it to `AudioRecorder::with_vad`.
- `settings.rs` — exposes only `push_to_talk: bool`. No "mode" enum, no silence-timeout setting.

So: the detector is live, but only as a frame filter inside PTT/toggle sessions. There's no mode where VAD **triggers** start/stop.

## Proposed behaviour

Add a new recording mode: `RecordingMode::VoiceActivated`. Alongside the existing PTT/toggle flows, this mode:

1. **Armed state**: after the user activates VAD mode (hotkey, tray toggle, or "enable on launch" setting), Handy opens the mic and runs VAD continuously but does **not** transcribe.
2. **Speech onset**: when VAD reports sustained speech (e.g. 3 consecutive `Speech` frames, ~90ms), Handy starts a real recording session — plays the distinctive start beep (item 02), captures audio, and begins streaming transcription if enabled (item 05).
3. **Speech end**: when VAD reports sustained silence for `silence_timeout_ms` (default 1200ms, configurable), Handy ends the session, runs final transcription, delivers text (item 05), plays stop beep.
4. **Disarm**: dedicated hotkey or tray item exits VAD mode entirely (mic released).

## Settings

- `recording_mode`: `PushToTalk` (default) | `Toggle` | `VoiceActivated`
- `vad.silence_timeout_ms`: 600 / 1200 (default) / 2000 / 3000
- `vad.speech_onset_frames`: 3 (sensitivity)
- `vad.threshold`: expose the Silero confidence threshold (currently hard-coded 0.3 in `managers/audio.rs:121`); 0.2 / 0.3 / 0.5 / 0.7
- `vad.armed_indicator`: overlay badge / tray icon variant for "mic armed but idle"

## Hypothesis — where this lives

- **Backend:**
  - `managers/audio.rs` — add an "armed listening loop" that runs VAD without writing to the recording buffer, and emits events on speech onset/offset.
  - `transcription_coordinator.rs` — handle the new "auto-start / auto-stop" events alongside the existing PTT/toggle triggers.
  - `settings.rs` — migrate `push_to_talk: bool` to a `RecordingMode` enum with backward-compatible deserialisation (old `true` → `PushToTalk`, `false` → `Toggle`).
  - `shortcut/` — add "toggle VAD mode on/off" as a bindable action distinct from PTT.
- **Frontend:**
  - `components/settings/` — new "Recording mode" section, replaces the current PTT boolean.
  - Overlay: new "armed idle" visual state distinct from "actively recording".
  - `bindings.ts` — regenerate after the Rust enum change.

## Approach

### Phase 1 — add the mode

1. Introduce `RecordingMode` enum + settings migration.
2. Build the armed-idle loop: mic open, VAD running, no recording buffer writes, low CPU.
3. Wire speech-onset → existing `start_recording` path; silence-sustained → existing `stop_recording` path.
4. UI: mode picker + VAD parameters behind "VAD settings" collapsible.

### Phase 2 — polish

1. "Armed idle" overlay/tray state so the user always knows mic is open.
2. Sensitivity presets — "quiet room", "noisy room", "push hard" — mapping onto the raw thresholds.
3. Auto-calibration: 3-second ambient noise sample on first VAD activation to suggest a threshold.
4. Pair with streaming transcription (item 05): VAD onset triggers a streaming session that commits on VAD offset.

### Phase 3 — optional

- "Wake word" gate — require a trigger phrase before entering armed state. (Rabbit hole; defer unless clearly wanted.)
- Multi-segment mode: stay armed and emit separate transcripts per utterance until user disarms, instead of disarming after every utterance.

## Risks

- **Privacy optics.** "Mic always open" is a real concern — must be visually unambiguous (tray, overlay) and easy to disarm. Document clearly in settings.
- **False triggers.** Typing sounds, coughs, background speech can all false-fire. Mitigations: threshold tuning, speech-onset frame count, optional wake-word (Phase 3).
- **CPU/battery.** Continuous VAD is cheap (ONNX Silero is tiny), but an always-open mic stream has power implications on laptops. Verify on a battery run.
- **Settings migration.** `push_to_talk: bool` is persisted; need a serde default so old configs load cleanly. Standard pattern — just don't forget.

## Definition of done

- `RecordingMode::VoiceActivated` selectable in settings; armed-idle state visible in tray/overlay.
- Speech onset reliably starts a capture; configurable silence timeout reliably ends it.
- Interoperates with streaming transcription (item 05) — onset starts stream, offset finalises.
- Old `push_to_talk: bool` configs migrate to `PushToTalk` with no user intervention.
- Manual test matrix: quiet room, noisy room, background music, background conversation.
