---
name: audio-post
description: Use when mixing or cleaning audio in Palmier Pro — music beds, ducking music under speech, fade in/out, denoise, removing silence, sound effects, or balancing voiceover levels.
---

# Palmier Pro audio post

Levels and fades are `volume` keyframes; cleanup is `denoise_audio` / `remove_silence`; new audio comes from [[generative-media]]. Read [[pro-editing]] first.

## The two rules that decide correctness

1. **`volume` keyframes are CLIP-LOCAL** (0 = the clip's own start). Music at timeline `M` → a duck at timeline frame `t` is keyframe `t − M`. With the bed starting at frame 0 they coincide, but always convert for a non-zero start.
2. **`set_keyframes` REPLACES the whole track.** Fade-in, ducking, and fade-out are all the music `volume` — build them as **ONE combined envelope in one call**, not three calls that overwrite each other.

## Ducking + fades = one envelope

Get speech spans from `get_transcript` (word timings) → convert to music-local frames → one `volume` keyframe list (0–1), ~6-frame ramps so it's not clicky:
```
set_keyframes { clipId:<music>, property:"volume", keyframes:[
  [0,0],[60,1],                 // fade in 1s
  [<sp1−6>,1],[<sp1>,0.25],[<sp1end>,0.25],[<sp1end+6>,1],   // duck under speech span 1
  … repeat per speech span …
  [<end−60>,1],[<end>,0]        // fade out 1s
]}
```

## Voice cleanup — order matters

- `denoise_audio { clipIds:[voice] }` — reduces background noise, **does not change timing**. Safe to run first.
- `remove_silence` — cuts dead air and **RIPPLES** (shifts every downstream frame). Run cleanup BEFORE you read positions for ducking/SFX/fades, then `get_timeline` + `get_transcript` again — earlier frame numbers are now stale.

## SFX & sweeteners

Generate via [[generative-media]] (`generate_audio` with `elevenlabs-sfx-v2`), place on a **separate audio track** so the bed's envelope doesn't sweep it. Start a whoosh ~15f BEFORE the cut so it swells into it. `mirelo-sfx` / `sonilo` can score to a video span automatically.

## Gotchas

| Symptom | Cause | Fix |
| :-- | :-- | :-- |
| Fade/duck happens at the wrong time | used timeline frames | `volume` keyframes are clip-local (`t − clipStart`) |
| Fade-in erased the ducking (or vice-versa) | separate `set_keyframes` calls overwrote | one combined `volume` envelope |
| Ducking frames all wrong after cleanup | `remove_silence` rippled | re-read timeline/transcript AFTER cleanup |
| SFX volume jumps around | placed on the ducked bed track | put SFX on its own track |
| Clicks/pops on level changes | instant steps | ~6-frame ramps between values |
