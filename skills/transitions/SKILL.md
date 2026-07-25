---
name: transitions
description: Use when adding a transition between clips in Palmier Pro — cross dissolve, fade or dip to black, whip pan, zoom/push, flash, or J-cut/L-cut audio leads — or when clips only hard-cut and you want a blended edit.
---

# Palmier Pro transitions

**There is no transition tool in Palmier Pro.** Every transition is hand-built from clip overlap + `set_keyframes`. Knowing this is the whole skill. Read [[pro-editing]] for the core model first.

## The rule that breaks every naive attempt

**`set_keyframes` frames are CLIP-RELATIVE (0 = the clip's first frame), and each call REPLACES that property's track.** A clip that sits at timeline `[120,240]` gets keyframes numbered `0..120`, NOT `120..240`. Passing timeline frames puts the keyframes off the end of the clip and the animation silently does nothing. Convert every time: `local = timelineFrame − clipStartFrame`.

Also: `opacity` is **0–1** (not 0–100). `position` is `[frame, topLeftX, topLeftY]` in **normalized top-left** coords (not centered pixels). **Effect params (blur, etc.) CANNOT be keyframed** — `set_keyframes` only animates `opacity, position, scale, rotation, crop, volume`.

## Recipes

**Cross dissolve (0.5s @ 60fps = 30f).** A dissolve needs both clips visible at once → they must OVERLAP on SEPARATE tracks.
```
1. move_clips { moves:[{clipId:B, toTrackIndex:<above A>, toStartFrame:<A.end−30>}] }   // overlap B onto A's tail
2. set_keyframes { clipId:A, property:"opacity", keyframes:[[<localStart>,1],[<localEnd>,0,"linear"]] }
3. set_keyframes { clipId:B, property:"opacity", keyframes:[[0,0],[30,1,"linear"]] }       // B local 0..30
```
Overlapping shortens the timeline by the overlap; account for it or move downstream clips.

**Dip / fade to black.** Where nothing renders on a lower track the frame is black, so fade the outgoing clip's `opacity` to 0 and the incoming from 0 — no matte needed on a single-clip stack. For a hard flash-to-white/color, put a solid image (see [[generative-media]] for making one) on a top track and keyframe its opacity up then down.

**Whip pan (~0.3s) — split so only the transition blurs.** Motion blur is static, so blurring the whole outgoing clip smears the *entire* shot. **Split off ~8 frames at the boundary** and blur only those. `split_clips` preserves & interpolates each piece's existing `scale` keyframes, so a moving clip survives the split; the right half gets a NEW id, the left half keeps the original.
```
split_clips { splits:[{clipId:OUT, atFrame:<cut−8>},{clipId:IN, atFrame:<cut+8>}] }
apply_effect { clipIds:[outTail, inHead], effects:[{type:"blur.motion", params:{radius:60, angle:0}}] }  // horizontal
set_keyframes { clipId:outTail, property:"position", keyframes:[[0, cx−w/2, 0],[8, -2.4, 0, "smooth"]] }  // slide off left
set_keyframes { clipId:inHead,  property:"position", keyframes:[[0, 0.15, 0],[8, cx−w/2, 0, "linear"]] }  // enter from right
```
`cx−w/2` is the cover-crop top-left of a covered clip (`w≈3.162`; see [[reel-mechanics]]). 1–2 frames of edge-black mid-whip are hidden by the blur. Clip-relative frames as always.

**Zoom / push transition.** Scale the outgoing clip up and the incoming from slightly larger — see `scale` mechanics in [[motion-keyframes]].

**J-cut / L-cut.** Lead or lag the audio across the cut: trim the video edit but move the nested `audio` clip earlier (J) or later (L) so sound precedes/follows picture. Edit the audio via the clip's nested `audio.id`.

## Gotchas

| Symptom | Cause | Fix |
| :-- | :-- | :-- |
| Fade/whip does nothing | keyframes in timeline frames | use clip-relative (`local = timelineFrame − clipStart`) |
| Dissolve is just a hard cut | both clips on one track (can't co-exist) | move one to a track above and overlap |
| Blur "ramp" ignored | tried to keyframe an effect param | effects aren't keyframeable — apply a static blur over the short segment (split if needed) |
| Clip jumps to a corner when scaling/moving | `position` is top-left normalized, not centered pixels | full-frame top-left is (0,0); off-screen left is x=−1 |
| Whole reel got shorter | dissolve overlap consumed frames | expected — re-place downstream clips |
| Whole shot is motion-blurred, not just the whip | blurred the un-split clip (blur is static) | split the ~8f boundary; blur only those two pieces |
| Whipped clip lost its Ken Burns push | assumed splitting drops keyframes | it doesn't — `split_clips` interpolates scale keyframes across pieces |
