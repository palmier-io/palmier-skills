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
2. **`position` is the TOP-LEFT corner and `scale` is the clip's size as a fraction of the canvas — not a center point and not a zoom factor.** Coordinates are 0–1 of the canvas. A clip that fills the frame is position `(0, 0)`, scale `(1, 1)`. **`scale > 1` overfills the canvas and crops the edges — that is a zoom-*in* / push-in; `scale < 1` shrinks the clip and exposes the background behind it (a smaller PIP), a zoom-*out*.** With **no position track**, the engine keeps a scale animation **centered for you** — it derives the top-left from the clip's center (`center − size/2`) every frame, so a pure scale zoom is a naturally centered zoom and does *not* drift to a corner. You only need to animate `position` in lockstep when you want **off-center** motion, or when the clip **already has an active position track** — an active position track is taken as the literal top-left and overrides that automatic recentering, so a position pinned to a constant while scale grows is what slides the subject toward the corner. To keep an *explicit* position track centered, hold `topLeft = (1 − size) / 2`: a centered push-in to scale `1.3` means position `−0.15` (negative — the top-left slides off-canvas as the clip grows); a centered half-size PIP is scale `0.5` at position `0.25`.

## set_keyframes reference

One property, one clip, per call. Each row is `[frame, ...values, interp?]` — every row must be an array; a bare `"linear"` outside a row is rejected. `interp` ∈ `linear | hold | smooth`, **default `smooth`**, and controls the segment **leaving** that keyframe (the last row's interp is irrelevant — put the token on the row the segment *starts* from). Rows are auto-sorted by frame; on a duplicate frame the last row wins. Pass `keyframes: []` to clear the track.

| property | row shape | units | notes |
| :-- | :-- | :-- | :-- |
| `opacity` | `[frame, value]` | 0–1 | 0 = transparent, 1 = opaque. Multiplies with the clip's native fade envelope — see note below |
| `volume` | `[frame, value]` | **dB** | 0 = unchanged, negative = quieter (e.g. −12), positive = boost (capped **+15 dB**), **≤ −60 dB = full mute**; audio/linked-audio clips |
| `rotation` | `[frame, degrees]` | clockwise° | unbounded; 360 = one full turn |
| `position` | `[frame, x, y]` | 0–1 canvas | **top-left corner**, not center |
| `scale` | `[frame, w, h]` | 0–1 canvas | clip **size** as fraction of canvas (1 = fills axis); **>1 zooms in / crops edges, <1 shrinks / shows background** — not a zoom factor |
| `crop` | `[frame, top, right, bottom, left]` | 0–1 of source | side insets, clockwise from top (T, R, B, L) |

**Interpolation feel:** `smooth` eases in and out (default, natural for motion) · `linear` constant speed (mechanical moves, steady audio ramps) · `hold` freezes the value until the next keyframe (stepped/snap cuts, no in-between).

**position/scale/rotation are motion keyframes** — while active they **override the clip's static `transform`**. A static transform set via `set_clip_properties` leaves the position/scale/rotation tracks intact, and the keyframe track wins over it. **`apply_layout` is different: it DELETES the position/scale/rotation/crop keyframe tracks** and writes its own static transform — the keyframe does *not* win, the animation is gone. So run `apply_layout` **first**, then set keyframes **last** on the same clip.

## Recipes

Frames below are clip-relative; substitute the clip's real length. Defaults (`smooth`) are omitted where they read fine. Interp tokens go on the **first row of the segment they govern** (the leaving keyframe).

- **Fade in / out** — `opacity`: `[[0,0],[15,1]]` in; `[[0,1],[15,0]]` out; both: `[[0,0],[15,1],[dur-15,1],[dur-1,0]]`. (First check `get_timeline` for existing `fadeInFrames`/`fadeOutFrames` — those multiply with opacity keyframes and will double-fade; see Notes.)
- **Ken Burns push-in (centered)** — the simplest centered push-in is **scale alone** — the engine keeps it centered: `scale [[0,1,1],[dur-1,1.3,1.3]]` (add `"linear"` on the first row for a steady drift; `>1` magnifies and crops, it does **not** shrink). Reach for a paired `position` track only for an **off-center** push-in, or when the clip already has a position track: then supply the centering yourself with `position [[0,0,0],[dur-1,-0.15,-0.15]]` (top-left goes to `(1−1.3)/2 = −0.15`). A push-*out* just swaps the two ends.
- **PIP slide-in from the right** — a half-size PIP resting at `(0.45, 0.05)` sliding in from off-canvas: `position [[0,1.1,0.05],[20,0.45,0.05]]`; keep `scale` static at its layout size (set it as a matching one-value track or leave the layout's transform if you're not also animating scale).
- **Slow pan across a wide shot** — a static clip has no room to move without exposing background, so hold `scale` above 1 and pan `position`. At scale `1.2` the safe top-left range is `−0.2 … 0` and centered y is `−0.1`: `scale [[0,1.2,1.2]]` (one row = constant) **and** `position [[0,0,-0.1,"linear"],[dur-1,-0.2,-0.1]]` pans the view left→right (`"linear"` on the **first** row governs the single segment; on the last row it would be ignored).
- **Spin** — `rotation`: `[[0,0,"linear"],[dur-1,360]]` for a constant-speed spin — the `"linear"` goes on the first row (the segment-leaving keyframe); without it the row defaults to `smooth` and the spin eases in and out.
- **Crop reveal (wipe open from center)** — `crop` starting fully closed to open: `[[0,0.5,0.5,0.5,0.5],[20,0,0,0,0]]`.
- **Audio duck under a voiceover** — `volume` (in **dB**, 0 = unchanged) on the music bed: `[[0,0,"linear"],[10,-12],[dur-10,-12,"linear"],[dur-1,0]]` dips it ~12 dB under the VO and lifts it back. Each `"linear"` sits on the row that *starts* a ramp (the dip from row 0, the lift from `dur-10`); a token on the flat hold would be inert.

## Running this inside Palmier Pro

1. **Read the clip.** `get_timeline` for the clip's `id`, its length (`end − start`), and any existing animation — an active track shows its rows, a constant one appears as a static field (e.g. `crop: {left: 0.31}`). Also note any `fadeInFrames`/`fadeOutFrames` and the clip's speed/duration. This gives you the frame budget and tells you whether you're editing an existing track or creating one.
2. **Finalize speed and duration first, then keyframe.** Changing a clip's `speed` or `durationFrames` via `set_clip_properties` does **not** rescale existing keyframes — it clamps them to the new length and **silently drops** any keyframe past the new end, flattening a two-keyframe move into a constant value (looks like no animation / a hard cut). If a clip was retimed *after* keyframing, re-read `get_timeline` for the new length and resend the full track with frames recomputed against it — the old end rows are gone, not remapped.
3. **Pick the property and build the full row list.** One `set_keyframes` call per property; a Ken Burns is one call (scale) unless off-center (add position), a slide is usually one.
4. **Order of operations.** Run any `apply_layout` **before** setting keyframes — `apply_layout` clears the position/scale/rotation/crop tracks. Likewise, setting `volume`/`opacity` scalars via `set_clip_properties` clears *those* keyframe tracks (a static transform via `set_clip_properties` is merely overridden, not cleared). Keyframe **last** so the track isn't wiped.
5. **Verify visually.** `inspect_timeline` at the start, middle, and end frames of the animation (composited output, transforms applied) to confirm the motion reads — don't trust the row math alone for position/scale, where the top-left model is easy to get wrong.

## Notes

- **Seeing your animation (live preview vs. inspect_timeline):** an MCP `set_keyframes` edit updates the running in-app preview automatically — no reopen or refresh. But a keyframe track only shows change *over time*: a single paused frame renders just that one sampled value, not the motion. The preview renders the animation faithfully when you **PLAY** it or step frame-by-frame, but while you **DRAG the scrub bar** it seeks coarsely, so a short move can look like a hard cut or barely move — it snaps to the correct frame on release. `inspect_timeline` and export are always frame-exact. So judge the motion by playing (or `inspect_timeline`), never by a scrub-drag.
- **Retime drops keyframes:** see step 2 — author keyframes after speed/duration are set; the agent path clamps rather than rescales.
- **Whole-track replacement:** to nudge one keyframe you must resend every row. Read the current track from `get_timeline` first, edit the list, send it back.
- **No extrapolation:** the value is flat before the first keyframe and after the last. "Enters from off-screen" only works if frame 0 is the off-screen position.
- **Centering math:** `topLeft = (1 − size) / 2` — for a push-in `size > 1`, so the top-left goes **negative**. This formula is only needed when a **position track is present** (off-center zooms, or one already exists): an active position track overrides the engine's automatic recentering and is taken as the literal top-left, so it must carry these values. Animating **scale alone** stays centered without it.
- **Opacity keyframes ≠ native fade:** unlike motion keyframes (which override the static transform), the `opacity` track and the clip's native fade handles (`fadeInFrames`/`fadeOutFrames`) are independent systems and the compositor **multiplies** them — composited opacity = keyframed-opacity × fade-envelope. A clip that already has fade handles will **double-fade** if you also add opacity fade keyframes over the same span. Before keyframing a fade, check `get_timeline` for existing fades and clear them, or just use the native fade for simple fade in/out. (Audio clips apply the fade to volume instead.)
- **Volume dB range:** the `volume` keyframe value is decibels — `0` = untouched, negatives duck (unlike the static `volume` scalar, which is 0–1 linear). **Positive dB boosts** the level, capped at **+15 dB** (higher values silently clamp); **≤ −60 dB is a hard mute** (full silence), so a duck to −40 is nearly inaudible but present, and −60 or lower is silent. A `0` row leaves the level unchanged.
- **A moving clip needs coverage:** panning/rotating a clip at scale 1 drags the canvas edge in and exposes background. Hold `scale > 1` (e.g. 1.2) whenever `position` travels, so the clip always overfills.
- **crop order is T, R, B, L** (clockwise from top), and insets are fractions of the **source** media, not the canvas.
- **Layouts aren't animations:** a static split-screen / grid / PIP arrangement belongs to `apply_layout`, not to hand-built position/scale keyframes. Reach for keyframes only when a value must *change over time*.
- **Linked audio:** `volume` keyframes address the audio side of a clip; on a linked video clip, target the nested audio `id` reported by `get_timeline`.
- **`hold` makes a step, not a slide** — use it for snap/stutter effects or to keep a value constant across a gap before jumping.
