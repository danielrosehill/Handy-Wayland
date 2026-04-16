# 04 — Curated English-only model list

## Motivation

Handy currently exposes a long model menu (multiple Whisper variants, multilingual, legacy, distilled, Parakeet variants). For Daniel's workflow — English-only dictation on a single workstation with a known GPU — most of these models are noise:

- Multilingual Whisper variants offer no benefit over English-specific ones and are often slower.
- Small/tiny Whisper models produce noticeably worse output than Parakeet at the same latency budget.
- Old distilled variants are superseded.

A curated shortlist surfaces the models that actually work well and hides the rest. "Show all models" stays available behind a toggle for users who need it.

## Proposed curated list (English dictation, desktop with GPU)

| Model | Role | Why it's in the shortlist |
|---|---|---|
| **Parakeet-TDT 0.6B v2** | Default, real-time streaming-capable | Current daily driver; best quality-per-latency with GPU |
| **Whisper large-v3 English-only** | High-accuracy fallback | When latency doesn't matter and you want peak accuracy |
| **Whisper small.en** | Low-VRAM fallback | Useful on CPU-only or low-VRAM GPU targets |

Everything else → "Additional models (advanced)" collapsible section.

## Hypothesis — where this lives

- `src-tauri/src/managers/model.rs` — presumably the model catalogue and download-manifest definitions.
- `src-tauri/src/managers/transcription.rs` — engine selection and inference.
- `src/components/` — the settings UI that renders the model picker.
- `src-tauri/src/settings.rs` — the enum/discriminator for the active model.

## Things to verify first

1. Confirm the model list is data-driven (JSON manifest or Rust const) vs. hand-coded per-variant in multiple places.
2. Check whether models are registered both in Rust (backend) and TS (frontend bindings in `src/bindings.ts`) — any change needs to touch both.
3. Check whether removing a model entry breaks users who already have that model downloaded — need a fallback/migration.

## Approach options

- **A — UI-only curation.** Keep the full model catalogue in the backend; add a frontend `featured: true` flag in the manifest; render "Recommended" group + collapsible "All models" group. Lowest risk, reversible, doesn't hurt other users if upstreamed.
- **B — backend pruning.** Actually remove entries from the catalogue. Simpler list, but forces-migration for existing users and loses flexibility. Not recommended.
- **C — per-language profile.** Introduce a `language_profile = "english" | "multilingual" | "all"` setting; the UI filters the catalogue by profile. Cleaner long-term but more scope.

Recommendation: **A for the fork, propose as upstream** — benefits all users, hurts none.

## Risks

- "Recommended" is subjective. Upstream maintainer may disagree on the shortlist. If so, keep it local to the fork.
- A featured flag is a new piece of schema; needs a default for backward compatibility (`featured: false`).

## Definition of done

- Model picker shows a short "Recommended" list on top, rest collapsed under "All models".
- Currently-selected model is always visible regardless of section.
- Shortlist is data-driven (manifest flag), not hard-coded in components.
