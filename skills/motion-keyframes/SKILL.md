---
name: motion-keyframes
description: Use when animating a clip in Palmier Pro — Ken Burns push-in or pan, zoom, speed ramps, slow-motion, freeze-frame, or any keyframed movement of a single clip.
---

# Palmier Pro motion & keyframes

Animate one clip with `set_keyframes`; retime with splits + `speed`. Read [[pro-editing]] first.

## `set_keyframes` value layouts (verified — memorize)

Frames are **CLIP-RELATIVE** (0 = clip start). Each call **replaces** the whole track. Row = `[frame, ...values, interp?]`, interp ∈ `linear|hold|smooth` (default smooth).

| property | row | notes |
| :-- | :-- | :-- |
| `opacity` | `[f, v]` | 0–1 |
| `rotation` | `[f, deg]` | clockwise |
| `position` | `[f, topLeftX, topLeftY]` | **top-left**, 0–1 normalized — NOT center |
| `scale` | `[f, width, height]` | **normalized dimensions** (1.0 = fills that axis) — NOT a scale factor; pass BOTH |
| `crop` | `[f, top,right,bottom,left]` | source insets 0–1 |
| `volume` | `[f, v]` | 0–1 (see [[audio-post]]) |

Motion keyframes (`position/scale/rotation`) override the static `transform` while active.

## Ken Burns push-in

`scale` is normalized w/h, so a 10% push = multiply the clip's covered w/h by 1.0→1.1. For a 9:16 clip already covered at `width≈3.162,height≈1.0` (see [[pro-editing]]):
```
set_keyframes { clipId:C, property:"scale", keyframes:[[0, 3.162, 1.0],[<end>, 3.478, 1.1, "smooth"]] }
```
Scaling grows the frame; if it drifts off-center, co-keyframe `position` (top-left) toward `(−(w−1)/2, −(h−1)/2)` to keep it centered — **verify with `inspect_timeline`** and adjust, since anchor behavior is easy to get wrong. For a pan, keyframe `position` alone.

### Hold an off-center crop during a push (session-verified)

A clip cover-cropped at `centerX ≠ 0.5` (biased to keep a subject that isn't centered — see [[reel-mechanics]]) **snaps back to center** the instant `scale` keyframes take over (motion keyframes override the static transform), chopping the subject. Co-keyframe `position` with the clip's INTENDED center, one `position` row per `scale` row:
```
topLeftX = centerX − w/2 ,  topLeftY = centerY − h/2       // w,h from the matching scale row
// e.g. centerX 0.58, punch scale [[0,3.42,1.081],[7,3.162,1.0]] →
set_keyframes { clipId:C, property:"position", keyframes:[[0, -1.13, -0.0405],[7, -1.001, 0, "smooth"]] }
```
Center-framed (`0.5`) clips need `scale` alone. `split_clips` preserves & interpolates scale keyframes across pieces, so you can split a moving clip (e.g. for a whip) without losing its push.

## Speed ramp (there is NO speed keyframe)

`set_keyframes` has no speed property. Ramps are structural: **split into segments, set a constant `speed` per segment.**
```
split_clips { splits:[{clipId:X, atFrame:<t1>},{clipId:<rest>, atFrame:<t2>}] }
set_clip_properties { clipIds:[<mid>], speed:0.4 }     // slow-mo middle
// slowing stretches on-screen length: on-screen = source_len / speed; move/ripple downstream clips
```
Slowing also slows the segment's audio — set that segment's `volume:0` (or detach) if unwanted.

## Slow-motion from high-fps footage

119.88fps footage on a 60fps timeline can go to `speed ≈ 0.5` and still show ~60 real fps — buttery slow-mo with no stutter. Prefer high-fps sources for hero slow moments.

## Freeze frame

Hold the final frame: `set_clip_properties { clipIds:[X], speed:<tiny> }` on a 1-frame tail split from the clip, or place the source with a single-frame trim for the hold length. Verify duration with `get_timeline`.

## Gotchas

| Symptom | Cause | Fix |
| :-- | :-- | :-- |
| Push-in does nothing / distorts | treated `scale` as a multiplier or set one axis | pass `[f,width,height]` normalized, both axes |
| Zoom drifts off-center | `position` is top-left; scaling grows from anchor | co-keyframe `position`, verify in inspect_timeline |
| Off-center subject re-centers on push | `scale` keyframes override the static `centerX` | co-keyframe `position` = `(centerX−w/2, centerY−h/2)` per row |
| "Speed keyframe" rejected | no such property | split + per-segment `speed` |
| Downstream clips overlap after slow-mo | on-screen length grew | ripple/move later clips |
| Slow-mo audio sounds warped | audio slowed with video | `volume:0` or detach that segment |
