# 03 — KDE Plasma + Wayland friction hardening

## Motivation

Handy is usable on KDE Plasma + Wayland but runs into a cluster of friction points — hotkey registration races, autostart ordering, text injection quirks, overlay placement, clipboard ownership. A full catalogue of hypotheses lives in [`Handy-Bugs-And-Ideas/ideas/kde-wayland-friction-points-watchlist.md`](../../my-repos/Handy-Bugs-And-Ideas/ideas/kde-wayland-friction-points-watchlist.md) — this item is the umbrella for verifying and fixing them in the fork.

## Scope (initial wave — highest ROI)

Verified-or-high-likelihood issues to address first:

1. **Autostart ordering on SDDM + Plasma Wayland.** Handy autostarts but tray icon / global hotkey frequently don't bind until a manual restart. Workaround lives in [`handy-wayland-autostart-fix`](../../my-repos/handy-wayland-autostart-fix); upstream fix is to delay first hotkey/tray registration until `StatusNotifierHost` and the compositor's shortcut portal are both ready, with exponential backoff retry.
2. **Clipboard paste producing garbled output on KDE Wayland.** Documented as a real reproduced bug in [`bugs/clipboard-paste-modifier-loss-kde-wayland.md`](../../my-repos/Handy-Bugs-And-Ideas/bugs/clipboard-paste-modifier-loss-kde-wayland.md) — modifier-key loss when synthesising Ctrl+V via enigo on kwin. Workaround: `wl-copy` + `ydotool` raw keycodes (script in same repo). Fix: make "Clipboard" typing mode on Linux use `wl-copy` + `ydotool` natively instead of enigo.
3. **Active-window detection before injection.** Enigo / ydotool don't know which window was focused at transcript-arrival time on Wayland. Need at minimum a documented contract ("we inject into whatever is focused when the transcript lands, not when PTT started") and ideally a settings toggle to defer injection until user refocuses the original window. (Related: see `focus-loss-during-dictation` in the Live-Typing-UX-Research repo.)
4. **Overlay/HUD placement.** Confirm the recording overlay uses `wlr-layer-shell` (or Tauri/winit's equivalent) and renders above fullscreen clients on kwin. Related to upstream PR #969 ("overlay not showing on non-primary monitors") already merged — verify fix holds for Daniel's multi-monitor setup.

## Things to verify first

- For each subitem, reproduce on current `main` of the fork and note exact build/version.
- For (1): check `src-tauri/src/shortcut/tauri_impl.rs` + tray init path for any retry logic — I suspect it's single-shot.
- For (2): open `src-tauri/src/input.rs` and find the Wayland branch of the clipboard-paste codepath; compare to the `handy-paste.sh` workaround.
- For (3): look for "focus" or "active window" references in `input.rs` / `transcription_coordinator.rs`.

## Approach

Each subitem becomes a tracked branch/PR. This file is the meta-tracker; detail per subitem lives in the linked bug files under `Handy-Bugs-And-Ideas/`.

Ownership split:
- **Upstream-worthy** (most subitems): fork branch → PR against `cjpais/Handy`.
- **Personal-only** (autostart systemd unit, etc.): stays in `handy-wayland-autostart-fix` repo, referenced from Handy's Linux install docs via PR.

## Risks

- KDE-Wayland-specific conditionals risk breaking GNOME/sway. Gate with compositor detection (`XDG_CURRENT_DESKTOP`, `KDE_FULL_SESSION`, or `wl_display` protocol probing) rather than generic `target_os = "linux"`.

## Definition of done

- Each subitem either: merged upstream, landed in a personal fork branch with notes, or promoted to a separate planning file.
- This file re-points to the subitem tracking locations.
