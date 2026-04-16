# Handy fork — planning

Personal improvements tracked in this fork (`danielrosehill/handy`). Target environment: Ubuntu 25.10, KDE Plasma on Wayland, Parakeet-TDT 0.6B v2, English-only dictation.

## Items

| # | Item | File | Status |
|---|---|---|---|
| 1 | F13 not accepted as hotkey | [01-f13-hotkey.md](01-f13-hotkey.md) | draft |
| 2 | Unreliable start/stop audio feedback | [02-audio-feedback-reliability.md](02-audio-feedback-reliability.md) | draft |
| 3 | KDE Plasma + Wayland friction hardening | [03-kde-wayland-hardening.md](03-kde-wayland-hardening.md) | draft |
| 4 | Curated English-only model list | [04-curated-english-models.md](04-curated-english-models.md) | draft |
| 5 | Paste-at-cursor + streaming transcription text input | [05-text-input-paste-and-streaming.md](05-text-input-paste-and-streaming.md) | draft |
| 6 | VAD-driven hands-free recording mode | [06-vad-recording-mode.md](06-vad-recording-mode.md) | deferred (nice-to-have) |

## How we work here

- One markdown per item: motivation, hypothesis/where in code, approach options, risks.
- Promote an item to "in progress" when we start it; link the branch.
- Upstream-worthy items get mirrored as issues/PRs against `cjpais/Handy`; personal-only changes stay on a fork branch.

## Related repos

- [`Handy-Bugs-And-Ideas`](../../my-repos/Handy-Bugs-And-Ideas) — reproducible bugs and watch-list
- [`handy-wayland-autostart-fix`](../../my-repos/handy-wayland-autostart-fix) — systemd/desktop autostart workaround
- [`Handy-Ubuntu-Setup`](../../my-repos/Handy-Ubuntu-Setup) — install/setup notes
