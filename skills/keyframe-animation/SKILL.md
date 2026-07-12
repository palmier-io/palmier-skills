---
name: keyframe-animation
description: Animate a clip's motion, opacity, or volume over time inside Palmier Pro by keyframing one property at a time with set_keyframes — fades, Ken Burns push-ins, picture-in-picture slides, spins, crop reveals, and audio ducks. Drives Palmier Pro's set_keyframes and get_timeline tools directly, with the clip-relative frame model, top-left/normalized coordinate system, and linear/hold/smooth interpolation grounded in how the engine actually samples tracks. Use whenever the user wants to animate, move, pan, zoom, fade, spin, grow/shrink, reveal, or duck a clip's value over time — "make it zoom in slowly," "fade this out," "slide the PIP in from the side," "add a Ken Burns effect," even if they don't say "keyframe."
---

# Keyframe Animation

## Core principle

A keyframe track animates **one property of one clip** by pinning values at specific frames; the engine interpolates between them. `set_keyframes` **replaces the whole track** for that property in a single call — you always send the complete list of rows, never a patch. Animation needs **at least two keyframes**: one keyframe is just a constant value, and outside the first/last keyframe the engine **holds** the end value (no extrapolation), so a "start off-screen, end centered" move needs both ends stated explicitly.

## The two things people get wrong

Both are baked into the value layouts, and both differ from the intuitive guess:

1. **Frames are clip-relative.** `0` is the clip's own first frame, not timeline frame 0. Keyframes travel with the clip when you move it. A 90-frame clip animates over rows `[0 … 89]` regardless of where it sits on the timeline.
2. **`position` is the TOP-LEFT corner and `scale` is the clip's size as a fraction of the canvas — not a center point and not a zoom factor.** Coordinates are 0–1 of the canvas. A clip that fills the frame is position `(0, 0)`, scale `(1, 1)`. **`scale > 1` overfills the canvas and crops the edges — that is a zoom-*in* / push-in; `scale < 1` shrinks the clip and exposes the background behind it (a smaller PIP), a zoom-*out*.** Keeping a zoom centered means moving position in lockstep with scale by `topLeft = (1 − size) / 2`: a centered push-in to scale `1.3` needs position `−0.15` (**negative** — the top-left corner slides off-canvas as the clip grows); a centered half-size PIP is scale `0.5` at position `0.25`. Scaling without moving position anchors to the top-left corner and drifts the subject toward it.

## set_keyframes reference

One property, one clip, per call. Each row is `[frame, ...values, interp?]`. `interp` ∈ `linear | hold | smooth`, **default `smooth`**, and controls the segment **leaving** that keyframe (the last row's interp is irrelevant). Rows are auto-sorted by frame; on a duplicate frame the last row wins. Pass `keyframes: []` to clear the track.

| property | row shape | units | notes |
| :-- | :-- | :-- | :-- |
| `opacity` | `[frame, value]` | 0–1 | 0 = transparent, 1 = opaque |
| `volume` | `[frame, value]` | **dB** | 0 = unchanged, negative = quieter (e.g. −12); audio/linked-audio clips |
| `rotation` | `[frame, degrees]` | clockwise° | unbounded; 360 = one full turn |
| `position` | `[frame, x, y]` | 0–1 canvas | **top-left corner**, not center |
| `scale` | `[frame, w, h]` | 0–1 canvas | clip **size** as fraction of canvas (1 = fills axis); **>1 zooms in / crops edges, <1 shrinks / shows background** — not a zoom factor |
| `crop` | `[frame, top, right, bottom, left]` | 0–1 of source | side insets, clockwise from top (T, R, B, L) |

**Interpolation feel:** `smooth` eases in and out (default, natural for motion) · `linear` constant speed (mechanical moves, steady audio ramps) · `hold` freezes the value until the next keyframe (stepped/snap cuts, no in-between).

**position/scale/rotation are motion keyframes** — while active they **override the clip's static `transform`**. Don't also set a static transform (via `update_clip`) or an `apply_layout` on the same clip for the same property; the keyframe track wins and the static value is ignored.

## Recipes

Frames below are clip-relative; substitute the clip's real length. Defaults (`smooth`) are omitted where they read fine.

- **Fade in / out** — `opacity`: `[[0,0],[15,1]]` in; `[[0,1],[15,0]]` out; both: `[[0,0],[15,1],[dur-15,1],[dur-1,0]]`.
- **Ken Burns push-in (centered)** — animate `scale` and `position` together so the center stays put. Start full, end ~30% tighter: `scale [[0,1,1],[dur-1,1.3,1.3]]` **and** `position [[0,0,0],[dur-1,-0.15,-0.15]]` (top-left goes to `(1−1.3)/2 = −0.15`, negative because a push-in grows past 1). Use `linear` on both for a steady drift. A push-*out* just swaps the two ends. (Verified against the compositor: >1 magnifies and crops, it does **not** shrink.)
- **PIP slide-in from the right** — a half-size PIP resting at `(0.45, 0.05)` sliding in from off-canvas: `position [[0,1.1,0.05],[20,0.45,0.05]]`; keep `scale` static at its layout size (set it as a matching one-value track or leave the layout's transform if you're not also animating scale).
- **Slow pan across a wide shot** — a static clip has no room to move without exposing background, so hold `scale` above 1 and pan `position`. At scale `1.2` the safe top-left range is `−0.2 … 0` and centered y is `−0.1`: `scale [[0,1.2,1.2]]` (one row = constant) **and** `position [[0,0,-0.1],[dur-1,-0.2,-0.1],"linear"]` pans the view left→right.
- **Spin** — `rotation`: `[[0,0],[dur-1,360]]`, `linear` for constant spin.
- **Crop reveal (wipe open from center)** — `crop` starting fully closed to open: `[[0,0.5,0.5,0.5,0.5],[20,0,0,0,0]]`.
- **Audio duck under a voiceover** — `volume` (in **dB**, 0 = unchanged) on the music bed: `[[0,0],[10,-12,"linear"],[dur-10,-12,"linear"],[dur-1,0]]` dips it ~12 dB under the VO and lifts it back.

## Running this inside Palmier Pro

1. **Read the clip.** `get_timeline` for the clip's `id`, its length (`end − start`), and any existing animation — an active track shows its rows, a constant one appears as a static field (e.g. `crop: {left: 0.31}`). This gives you the frame budget and tells you whether you're editing an existing track or creating one.
2. **Pick the property and build the full row list.** One `set_keyframes` call per property; a Ken Burns is two calls (scale + position), a slide is usually one.
3. **Order of operations.** Set keyframes **after** any static `update_clip`/`apply_layout` on the same property — setting a scalar value (e.g. `update_clip` volume/opacity, or `apply_layout` transform) **clears** the keyframe track on that property. Keyframe last so it isn't wiped.
4. **Verify visually.** `inspect_timeline` at the start, middle, and end frames of the animation (composited output, transforms applied) to confirm the motion reads — don't trust the row math alone for position/scale, where the top-left model is easy to get wrong.

## Notes

- **Whole-track replacement:** to nudge one keyframe you must resend every row. Read the current track from `get_timeline` first, edit the list, send it back.
- **No extrapolation:** the value is flat before the first keyframe and after the last. "Enters from off-screen" only works if frame 0 is the off-screen position.
- **Centering math:** `topLeft = (1 − size) / 2` — for a push-in `size > 1`, so the top-left goes **negative**. Any zoom that should stay centered must animate `position` in lockstep with `scale`, or the subject slides toward the top-left corner.
- **A moving clip needs coverage:** panning/rotating a clip at scale 1 drags the canvas edge in and exposes background. Hold `scale > 1` (e.g. 1.2) whenever `position` travels, so the clip always overfills.
- **crop order is T, R, B, L** (clockwise from top), and insets are fractions of the **source** media, not the canvas.
- **Layouts aren't animations:** a static split-screen / grid / PIP arrangement belongs to `apply_layout`, not to hand-built position/scale keyframes. Reach for keyframes only when a value must *change over time*.
- **Volume is in dB, not 0–1:** the `volume` keyframe value is decibels (0 = untouched, negatives duck) — unlike the static `volume` scalar, which is 0–1 linear. A `0` row leaves the level unchanged, so ramp *between* 0 and negative dB.
- **Linked audio:** `volume` keyframes address the audio side of a clip; on a linked video clip, target the nested audio `id` reported by `get_timeline`.
- **`hold` makes a step, not a slide** — use it for snap/stutter effects or to keep a value constant across a gap before jumping.
