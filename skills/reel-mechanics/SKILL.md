---
name: reel-mechanics
description: Use when assembling a vertical (9:16) reel from landscape footage in Palmier Pro — fixing the aspect-ratio snap-back, cover-cropping 16:9 into 9:16, tiling clips onto a beat grid, keeping off-center subjects framed during motion, hand-building whip-pans, and screen-blending flash / light-leak / wash overlays. The integrator that ties beat-sync, motion-keyframes, transitions, and vfx-compositing into one clean build.
---

# Palmier Pro reel mechanics

Assemble-a-vertical-reel field notes — the ordering rules and exact numbers that turn raw landscape clips into a beat-synced, motion-rich 9:16 cut without redos. Read [[pro-editing]] first; this pulls in [[beat-sync]], [[motion-keyframes]], [[transitions]], [[vfx-compositing]].

## Build order (do NOT reorder — each step depends on the last)

1. `create_timeline` (keep versions intact) → `set_project_settings { aspectRatio:"9:16", quality:"1080p" }`.
2. `add_clips` **A-roll/montage first** — this is where the canvas snaps (see below). Re-assert 9:16 immediately after.
3. Mute every clip's linked source audio (`volume:0`) — grab the nested `audio.id`s from the `add_clips` response.
4. Add the music bed on its own audio track; `detect_beats`.
5. Cover-crop every video clip (below).
6. Grade (`apply_color`) → looks (`apply_effect`: clarity/dehaze, glow, grain, vignette).
7. Motion keyframes (below). 8. Transitions/overlays (below). 9. Text on **separate** tracks. 10. Trim trailing audio, then export.

## The aspect-ratio snap-back (bit everyone once)

Placing the FIRST clip **resets the project to that clip's aspect ratio** — a 16:9 source snaps the canvas back to 1920×1080 even after you set 9:16. The `add_clips` note says `"Matched timeline resolution to clip"`. Fix: `set_project_settings { aspectRatio:"9:16", quality:"1080p" }` right after the first placement, then cover-crop. Setting 9:16 *before* placing is necessary but NOT sufficient — you must re-assert after.

## Cover-crop: 16:9 → 9:16 (verified)

`transform.width` and `transform.height` are **independent canvas-fraction scale factors**, not a locked-aspect pair. A freshly placed 16:9 clip auto-fits to `height:0.316` (letterboxed band). Two-step trap: setting only `width:3.162` leaves `height:0.316` → the image is squished into a thin strip. Set **both**:

```
set_clip_properties { clipIds:[…], transform:{ width:3.162, height:1.0, centerX:0.5, centerY:0.5 } }
```

- `3.162 = (16/9) ÷ (9/16)` — the width fraction that makes a 16:9 source fill 1080×1920 height. Response then collapses to `{width:3.162}` (aspect locks) — that's the correct end state.
- Bias the crop to keep a subject: subject on source-left → raise `centerX` (~0.58) to slide the image right so its left third sits in frame. Verify with `inspect_timeline`.

## Beat-tiling the montage (see [[beat-sync]])

`beatFrame = round(beatSec × fps)` (bed starts at frame 0, no trim). Place each shot `startFrame = beatFrame_i`, `source:[inSec, inSec + (beatFrame_{i+1}−beatFrame_i)/fps]` so clips tile with no gap/drift. Pacing that reads: hold the hero over the lead-in (frame 0 → `beats[0]`), cut every beat through the drop, every 2 beats on the groove. Static locked-off angles → the in-point barely matters; vary angle, not timing.

## Motion that preserves off-center framing

Ken Burns / punch-in via `set_keyframes scale [[f,w,h]…]` (both axes, ratio-locked: `w/h ≡ 3.162`). **Scale keyframes override the static transform and recenter to 0.5** — so a clip you cover-cropped at `centerX:0.58` will drift and chop the subject. Co-keyframe `position` (top-left, normalized) to hold it:

```
topLeftX = centerX − w/2 ,  topLeftY = centerY − h/2   // per keyframe, matching the scale row
```
Center-framed (0.5) clips need scale only. Punch-in = start big, settle: `[[0,3.42,1.081],[7,3.162,1.0,"smooth"]]`.

## Whip-pan (see [[transitions]]) — verified recipe

Motion blur is static (not keyframeable), so **split the boundary** first — only the transition blurs, not the whole shot. `split_clips` preserves & interpolates existing scale keyframes across the pieces (confirmed).

```
split_clips { splits:[{clipId:OUT, atFrame:cut−8},{clipId:IN, atFrame:cut+8}] }
apply_effect { clipIds:[outTail, inHead], effects:[{type:"blur.motion", params:{radius:60, angle:0}}] }
set_keyframes { clipId:outTail, property:"position", keyframes:[[0, cx−w/2, cy−h/2],[8, −2.4, cy−h/2, "smooth"]] }  // slide off left
set_keyframes { clipId:inHead,  property:"position", keyframes:[[0, 0.15, 0],[8, cx−w/2, 0, "linear"]] }             // enter from right
```
`split_clips` right-half gets a NEW id; left half keeps the original. 1–2 frames of edge-black mid-whip is hidden by the blur.

## Screen-blend overlays — flash, light-leak, warm wash (see [[vfx-compositing]])

Everything bright-on-black composites with `blendMode:"screen"` on a **top track**, opacity animated per instance:
- **Flash:** white matte (`import_media { source:{ matte:{ hex:"#FFFFFF", aspectRatio:"9:16" }}}`), 6-frame spike `[[0,0],[2,0.72],[6,0]]`. Put one on the drop downbeat + outro hit; softer (~0.5) mid-downbeats. Restraint reads pro.
- **Warm wash:** solid warm matte (`#FF6A00`), screen, swell `[[0,0],[6,0.26],[18,0.14],[30,0]]` across the drop.
- **Light-leak:** a generated leak-on-black clip screen-blends beautifully. **Mute its linked audio** (`volume:0`) — and note generated *video* often ships an auto-audio track that the safety filter can reject (Seedance failed on it; **Grok Imagine worked**). Prompt "silent, no people, no text, pure black background"; use the cleanest ~2s window.

## Text: overlapping titles need SEPARATE tracks

Clips on one track are sequential — a second text clip that overlaps the first **trims/overwrites it** (a big title collapses to a flash). For a lockup + tagline + handle showing together, place each on its own text track (one `add_texts` call per track). `apply_effect` (glow) does NOT work on text clips — use the style `outline`+`shadow` for legibility instead.

## Getting source media in

`import_media` URL is **HTTPS ≤ 1 GB** (logos, generated leaks, stock — fine). Multi-GB camera originals exceed it and base64 download is a non-starter: use `source.path` (absolute local file, referenced in place). Shared-with-me Google Drive isn't in the local mount by default — the owner adds it to My Drive / downloads it, then import by path.

## Gotchas

| Symptom | Cause | Fix |
| :-- | :-- | :-- |
| Canvas went 16:9 after first clip | placing a 16:9 source snaps the project | re-`set_project_settings` 9:16, then cover-crop |
| Clip is a squished middle band | set `width` only; `height:0.316` fit remained | set `width:3.162` AND `height:1.0` |
| Zoom drifts, chops the subject | `scale` keyframes recenter to 0.5 | co-keyframe `position` to hold `centerX` |
| Whole shot is motion-blurred | blurred the un-split clip (blur is static) | `split_clips` the 8f boundary, blur only that |
| Big title flashed then vanished | overlapping text on one track overwrote it | separate text tracks for simultaneous titles |
| Overlay shows a black box | screen overlay left in `normal` blend | `blendMode:"screen"` on a top track |
| Generated leak failed / added noise | model auto-generated audio (filtered) | prompt "silent"; mute the overlay's `audio.id`; try Grok if Seedance rejects |
| Reel ends on black/silence | trailing SFX/overlay extended the timeline | trim it so max content = intended last frame |
| Bed cuts out abruptly | no fade | keyframe `volume` down over the last ~20f |
