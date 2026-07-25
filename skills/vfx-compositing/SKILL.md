---
name: vfx-compositing
description: Use when applying visual effects or compositing in Palmier Pro — glow/bloom, film grain, blur, sharpen, clarity/dehaze, vignette, chroma/green-screen key, light-leak or overlay compositing, blend modes, or split-screen/PIP/grid layouts.
---

# Palmier Pro VFX & compositing

`apply_effect` is the looks/FX stack (distinct from `apply_color` grading — see color-grading). Read [[pro-editing]] first.

## `apply_effect` — exact types (verified; guessing names fails)

Call shape **merges by type**: `apply_effect { clipIds:[…], effects:[{type, params}], remove:[…] }`. Passing an effect updates/adds it; omit to leave it; list its type in `remove` to delete; `enabled:false` bypasses. Effects render in a fixed canonical order regardless of call order. Returns `[{type,params}]` — copyable between clips.

| type | params (range, default) |
| :-- | :-- |
| `detail.clarity` | clarity (−1…1, 0), dehaze (−1…1, 0) |
| `blur.gaussian` | radius (0…100px, 8) |
| `blur.sharpen` | amount (0…2, 0.4) |
| `blur.noiseReduction` | amount (0…1, 0) |
| `blur.motion` | radius (0…100px, 0), angle (−180…180°, 0) |
| `stylize.grain` | amount (0…1, 0), size (0.5…4, 1.5) |
| `stylize.vignette` | amount (−1…1, 0), midpoint (0…1, 0.5), roundness (−1…1, 0), feather (0…1, 0.5) |
| `stylize.glow` | intensity (0…1, 0), radius (0…100px, 20), threshold (0…1, 0.6), warmth (0…1, 0) |
| `key.chroma` | keyHue (0…1, **0.333 = green**), tolerance (0…1, 0), softness (0…1, 0.1), spill (0…1, 0.5) |

**There is no luma key** — only chroma. Effect params are **not keyframeable**.

## Recipes

**Dreamy look:** `stylize.glow {intensity:0.5, threshold:0.7}` + `stylize.grain {amount:0.15}`. Grade separately in color-grading.

**Green-screen key:** `key.chroma` — the default `keyHue:0.333` is already green, so often just raise `tolerance` (~0.3) and set `spill` to kill green fringe. `keyHue` is a **normalized hue 0–1**, not a hex color. Put the keyed clip on a track ABOVE what shows through.

**Light-leak / overlay (only bright streaks show):** there's no luma key, so composite with a blend mode. Place the leak clip on a top track and `set_clip_properties { clipIds:[leak], blendMode:"screen" }` — black becomes invisible, bright streaks add. Blend modes: `normal, darken, multiply, colorBurn, lighten, screen, colorDodge, overlay, softLight, hardLight, difference, exclusion, hue, saturation, color, luminosity`. Animate the leak's `opacity` to swell and fade.

**Vignette:** `stylize.vignette {amount:0.5, feather:0.6}` (negative amount brightens edges).

**Cinematic reel finish (whole-timeline stack, session-tested):** apply to *every* clip for a unified filmic bloom — `detail.clarity {clarity:0.35, dehaze:0.3}` (cuts glass haze, adds bite) + `stylize.glow {intensity:0.22, radius:24, threshold:0.62, warmth:0.2}` (warm bloom on lights) + `stylize.grain {amount:0.12, size:1.5}` + `stylize.vignette {amount:0.35, midpoint:0.45, feather:0.5}`. Grade first (`apply_color`), effects after. Keep glow ≤0.25 so highlights bloom without smearing.

**Flash & wash accents (beat-synced):** a solid matte on a top track, `blendMode:"screen"`, `opacity` spiked per hit. Cheapest matte — no generation: `import_media { source:{ matte:{ hex:"#FFFFFF", aspectRatio:"9:16" }}}`. White flash on the drop/outro downbeats, 6-frame spike `[[0,0],[2,0.72],[6,0]]`; softer (~0.5) on mid downbeats. Warm wash = an orange matte (`#FF6A00`) swelling `[[0,0],[6,0.26],[18,0.14],[30,0]]` across the drop. Restraint reads pro — one accent per section, not every beat.

**Generated light-leak (organic streaks/bokeh):** generate a leak-on-black clip (see [[generative-media]]), screen-blend on a top track, animate `opacity`. **Mute its linked audio** (`volume:0`) — generated video always ships an audio track. Prompt "silent, pure black background, no people, no text"; screen drops the black so only the streaks add. Use the cleanest ~2s window (later seconds can show block artifacts).

## Split-screen / PIP / grid — `apply_layout`

Named layouts, tool computes cover-crop per slot: `full, side_by_side, top_bottom, pip_bottom_right/left, pip_top_right/left, grid_2x2, main_sidebar, three_up`. Two modes: `mediaRef` per slot (place new + `startFrame`/`endFrame`) or `clipIds` per slot (re-layout existing). `fit:"fill"` (default, cover-crop) vs `"fit"` (letterbox). Bias the crop with `anchor`/`anchorX`/`anchorY` when a face gets chopped. Prefer this over hand-set transforms for multi-clip framing.

## Gotchas

| Symptom | Cause | Fix |
| :-- | :-- | :-- |
| "Unknown effect" | guessed name (bloom/vignette/chroma_key) | use exact ids above (`stylize.glow`, `key.chroma`, …) |
| Key does nothing / wrong color keyed | passed a hex to `keyHue` | `keyHue` is 0–1 hue; green≈0.333 (default) |
| Light leak shows black box | overlay in `normal` blend | `blendMode:"screen"` (or lighten/colorDodge) |
| Effect "ramp" ignored | tried to keyframe a param | effects are static — split the clip to change intensity over time |
| Overlay covered base clip | placed on `trackIndex:0` | omit trackIndex → new top track |
| Glow does nothing on a title | `apply_effect` rejects text clips | use the text style `outline`+`shadow` for legibility instead |
| Generated leak overlay rejected | model's auto-audio hit the content filter (Seedance) | prompt "silent"; mute the overlay's audio; retry on Grok Imagine |
| Overlay audio bled into the mix | forgot the generated video's linked audio | `volume:0` on its nested `audio.id` after placing |
