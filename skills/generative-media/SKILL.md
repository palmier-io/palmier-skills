---
name: generative-media
description: Use when generating AI media inside Palmier Pro — AI b-roll or a seamless continuation shot, AI music beds, TTS voiceover, sound effects, video-to-music scoring, or upscaling footage.
---

# Palmier Pro generative media

Generate video/image/audio from prompts, then place with `add_clips`. **Always `list_models` first** — models, durations, resolutions, and reference limits vary. Generation is **async and costs real money**; the call returns a placeholder `mediaRef` that becomes usable once ready (`get_media { pending:true }` to poll). Read [[pro-editing]] first.

## Models (from `list_models`, verified)

- **Video:** `seedance-2-mini` (4–15s, 480/720p, first+last frame, up to 9 image / 3 video / 3 audio refs), `grok-imagine-video` (6–15s, first frame).
- **Image:** `nano-banana-lite`, `grok-imagine` (both support an image reference).
- **Audio:** `elevenlabs-tts-v3` / `gemini-3.1-flash-tts` (TTS; Gemini takes `styleInstructions`), `elevenlabs-music` (3–600s, `instrumental`), `elevenlabs-sfx-v2` (1–30s), `sonilo-v1.1-video-to-music` (score a video), `mirelo-sfx-v1.5-video-to-audio`.

## Seamless continuation (invisible join)

A model can't "continue" from nothing — condition on the join frame:
```
generate_video { model:"seedance-2-mini", prompt:"<continue the exact scene + camera motion>",
  startFrameMediaRef:<image of the existing clip's last frame>,
  aspectRatio:"9:16", resolution:"720p", duration:6 }
```
- Use `startFrameMediaRef` (an **image asset**, not `first_frame_image`). Set `aspectRatio` + `resolution` to match the timeline — there are **no fps/width/height params**.
- **Getting the join frame as an image asset** is the tricky part: Seedance can instead take the actual clip as `referenceVideoMediaRefs` (refer to it as `@Video1` in the prompt) — prefer that when you can't easily extract a still. If you do need a still, verify the project's frame-export path rather than guessing. Never use `generate_image` for the seam — it fabricates a non-matching frame.
- Describe the continued *camera motion* in the prompt; the model conditions on the frame, not its velocity.

## AI music / VO / SFX

```
generate_audio { model:"elevenlabs-music", prompt:"<mood, tempo, instruments>", instrumental:true, duration:30 }
generate_audio { model:"elevenlabs-tts-v3", prompt:"<the VO line>", voice:"Rachel" }
generate_audio { model:"elevenlabs-sfx-v2", prompt:"whoosh transition", duration:1 }
```
No `type:` field — the **model** determines TTS vs music vs SFX. Music `instrumental:true` = no vocals. Place results on their own tracks and mix per [[audio-post]]. `sonilo`/`mirelo` score an existing timeline span (`videoSourceStartFrame`/`EndFrame`).

## Other

- **Solid color plate** (flashes / mattes / washes — see [[transitions]], [[vfx-compositing]]): prefer `import_media { source:{ matte:{ hex:"#FFFFFF", aspectRatio:"9:16" }}}` — a flat plate needs **no generation** (instant, free). Use `generate_image` only for texture/gradient a solid matte can't give.
- **Upscale:** `upscale_media { mediaRef }` to conform a low-res generation to the timeline.

## Abstract overlays for compositing (light-leaks, flares, bokeh)

For screen-blend VFX (see [[vfx-compositing]]), generate the element **on pure black** so the blend drops the background:
```
generate_video { model:"grok-imagine-video", aspectRatio:"9:16", resolution:"720p", duration:6,
  prompt:"warm anamorphic lens flares and light leaks streaking on a pure black background, drifting bokeh, silent, no people, no objects, no text" }
```
- **Mute the result's linked audio** (`volume:0` on its nested `audio.id`) after `add_clips` — generated video always carries an audio track.
- **Model choice matters here:** Grok Imagine has been reliable; **Seedance can fail** with `"failed: Output audio has sensitive content"` — its auto-generated audio trips the safety filter even though the visual is fine. If Seedance rejects, retry on Grok.
- Place on a **top track**, `blendMode:"screen"`, animate `opacity` to swell/fade. Use the cleanest ~2s window — later seconds sometimes show block artifacts.
- **Solid flash/wash plate** is cheaper than a generation: `import_media { source:{ matte:{ hex:"#FFFFFF", aspectRatio:"9:16" }}}` (white flash) or `#FF6A00` (warm wash).

## Gotchas

| Symptom | Cause | Fix |
| :-- | :-- | :-- |
| Visible jump at the join | didn't condition on the last frame | `startFrameMediaRef` (still) or `referenceVideoMediaRefs` (the clip) |
| Wrong param name | used `first_frame_image` / fps / width | `startFrameMediaRef`; `aspectRatio`+`resolution` |
| `type:"music"` rejected | no such field | pick the model (`elevenlabs-music`, etc.) |
| Asset "not ready" | generation is async | poll `get_media { pending:true }` before `add_clips` |
| Surprise cost | generation bills per call | confirm model/duration before firing |
| `failed: Output audio has sensitive content` | model's auto-generated audio tripped the filter (Seedance) | prompt "silent"; retry on a different model (Grok Imagine worked) |
| Overlay audio bleeds into the mix | forgot the generated video's linked audio track | `volume:0` on the clip's nested `audio.id` |
