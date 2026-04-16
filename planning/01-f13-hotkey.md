# 01 — F13 not accepted as hotkey

## Motivation

F13 is Daniel's preferred dictation trigger — it's an "extended" function key that doesn't collide with any normal app binding, which is exactly what a global PTT needs. Currently Handy refuses to accept F13 during hotkey binding, or accepts it but never fires.

## Hypothesis — where this lives

- `src-tauri/src/shortcut/` — `handy_keys.rs` likely defines the allow-list of key symbols; `handler.rs` / `tauri_impl.rs` wire registration through Tauri's global-shortcut plugin.
- Tauri's global-shortcut plugin on Linux ultimately goes through `global-hotkey` → evdev/X11 key symbols. F13–F24 are valid X11 keysyms (`XK_F13` = 0xFFCA…) and valid evdev codes (`KEY_F13` = 183), but they're frequently missing from hand-maintained allow-lists.
- On Wayland/KDE the actual registration path may be via XDG `GlobalShortcuts` portal or KGlobalAccel (see watch-list item 1); the portal has no problem with F13 as a string `F13`, but the internal enum/match may not emit it.

## Things to verify first

1. Try to bind F13 in the UI — observe whether the binding prompt rejects the key, records a different key, or accepts it but never triggers.
2. Check `handy_keys.rs` for an explicit F-key range — is it capped at F12?
3. Test under X11 (start session with X11 instead of Wayland) to separate "Handy doesn't know F13" from "Wayland/KDE doesn't deliver F13".
4. Confirm F13 is actually arriving at the desktop: `wev` (Wayland) or `xev` (X11) while pressing the key on the physical keyboard or via the mapped macro key.

## Approach options

- **A — extend key enum.** If `handy_keys.rs` enumerates F1–F12, extend to F13–F24. Lowest risk.
- **B — accept arbitrary keysym strings.** Let the binding UI pass through whatever evdev/keysym name the OS emits, validated only for "non-empty + round-trips through registration". More flexible, slightly more failure-mode surface area.
- **C — separate KDE path.** If registration goes via KGlobalAccel, register a `.desktop`-based action so F13 can be bound in System Settings → Shortcuts as a fallback regardless of Handy's internal allow-list.

## Risks

- F13+ keys may be remapped at the hardware or compositor level (many keyboards send F13 as `Fn+F1`). If so, the "bug" is upstream of Handy — but we should still accept the key.
- Some Tauri `global-hotkey` builds historically stripped F13+; check crate version in `Cargo.toml` and bump if needed.

## Definition of done

- F13 can be selected in the binding UI.
- PTT triggers reliably on F13 press under KDE Wayland.
- Regression test (or at minimum a manual test note) added to `tests/` or BUILD notes.
