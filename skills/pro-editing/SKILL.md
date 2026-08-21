---
name: pro-editing
description: Use when starting or troubleshooting an edit in the Palmier Pro MCP video editor — new timeline setup, placing footage, aspect-ratio or fps surprises, black bars/letterboxing, cover-cropping a 16:9 source into 9:16, or deciding which palmier editing skill to reach for.
---

# Palmier Pro editing — core model & project setup

The router + shared reference for the Palmier Pro editing skills. Read this first; it holds the model every other skill assumes.

## Core model (verified)

- **Timeline positions = project frames** (`startFrame`, `[start,end)`). **Source positions = seconds** (`source:[s0,s1]` on `add_clips`). The tools convert — never multiply by fps yourself for placement.
- **Tracks are ordered; index 0 renders on TOP.** Video/image/text share video tracks. `add_clips`/`add_texts` with no `trackIndex` auto-creates a **new top track**; passing `trackIndex:0` onto an occupied track **overwrites** it.
- **A video clip's audio is nested** as `audio:{id,track,volume}` — edit the audio side via that nested id.
- **`set_keyframes` frames are CLIP-RELATIVE** (0 = the clip's first frame) and each call **REPLACES** that property's whole track. This is the #1 source of silent breakage — see [[transitions]] and [[audio-post]].
- **`transform`** fields are 0–1 fractions of the frame: `centerX/centerY` (position), `width/height` (size; independent — set both to keep aspect).

## Project setup — the exact order (gotchas are load-bearing)

```
- [ ] 1. set_project_settings { aspectRatio, quality } — set 9:16 FIRST
- [ ] 2. add_clips / create_timeline — placing a clip SNAPS the project to the source aspect
- [ ] 3. set_project_settings { aspectRatio:"9:16", quality:"1080p" } — RE-SET right after placement
- [ ] 4. cover-crop each 16:9 clip to fill 9:16 (below)
- [ ] 5. inspect_timeline — verify no black bars before continuing
```

**Two behaviors that silently bite:**
1. **Aspect snap-back.** Even after you set 9:16, the first placed clip re-snaps the project to that clip's aspect (a 4K 16:9 clip → project becomes 3840×2160). Re-set 9:16 after placing and confirm the returned resolution.
2. **fps rebasing.** Adding media can rebase the timeline fps (e.g. 120→60), renumbering every frame while preserving real-time durations. After adding media, `get_timeline` and read `fps`/`totalFrames` fresh before any frame math.

## Cover-crop a 16:9 source into 9:16 (no black bars)

A 16:9 (3840×2160) clip on a 9:16 (1080×1920) frame letterboxes to `height ≈ 0.316`. To COVER (fill height, crop sides):

```
set_clip_properties { clipIds:[…], transform:{ centerX:0.5, centerY:0.5, height:1, width:3.162 } }
```

`width ≈ 3.162` = 1/0.316 (source overflows and is center-cropped). Nudge `centerX` (0.4/0.6) if the subject is off-center. Or use `apply_layout` with the `full` slot to let the tool compute the cover crop.

## Export

`export_project { mode:"video", outputPath, overwrite:true }` → async job. Poll `manage_exports { action:"list" }` until `status:"completed"`. Reset `quality:"1080p"` before export if the timeline drifted to 4K.

## Which skill

| Want | Skill |
| :-- | :-- |
| Dissolve, whip pan, dip-to-black, J/L cut | [[transitions]] |
| Ken Burns, zoom, speed ramp, slow-mo, freeze | [[motion-keyframes]] |
| Glow, grain, vignette, key, blend, split/PIP | [[vfx-compositing]] |
| Cut to the beat, montage pacing | [[beat-sync]] |
| Music beds, ducking, denoise, SFX, VO | [[audio-post]] |
| AI b-roll, AI music/VO, upscale | [[generative-media]] |
| Exposure / white balance / wheels / curves / LUT | color-grading |
| Assemble → trim → caption a talking-head UGC post | ugc-editing |

## Common mistakes

- Frame math with a stale fps after a rebase → re-read `get_timeline`.
- Timeline keyframe numbers where clip-relative is required → offsets land off the clip and the animation silently no-ops.
- `trackIndex:0` for an overlay → destroys the base clip. Omit `trackIndex` for a new top track.
