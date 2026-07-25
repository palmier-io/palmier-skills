---
name: beat-sync
description: Use when cutting footage to music in Palmier Pro — beat-matched montages, cutting on the beat, rhythmic pacing, hitting the drop, or syncing edits to a track's tempo and downbeats.
---

# Palmier Pro beat-sync

Cut on the beat by reading the track's beat grid, converting to frames, and placing clips so every cut lands on a beat. Read [[pro-editing]] first.

## `detect_beats` — verified output

`detect_beats { mediaRef }` returns:
```
{ beats:[0.02, 0.62, 1.2, 1.8, …],        // every beat, in SOURCE SECONDS
  downbeats:[0.02, 2.38, 4.74, …],         // bar starts (accents) — cut here for big hits
  bpm:103.4,
  units:"source seconds — multiply by fps for frame values" }
```
So `beatFrame = round(beatSeconds × timelineFps)`. **These are source seconds of the track**; the beat lands at that many seconds into the music clip. If the music clip starts at timeline frame `M` and is trimmed to start `t0` seconds in, the on-timeline cut frame is `M + round((beatSeconds − t0) × fps)`.

## Recipe — beat-tiled montage

```
1. detect_beats { mediaRef:<music> }         // grab beats + downbeats
2. get_media                                  // clip durations (each must cover its beat span)
3. choose cut points: every beat (busy), every 2nd/4th beat (breathe), or `downbeats` (on the bar)
4. convert each cut point to a frame (above)
5. add_clips: tile clips so clip i occupies [cutFrame_i, cutFrame_{i+1})
   — reuse the SAME rounded frame as clip i's end and clip i+1's start (no gaps, no drift)
6. cover-crop each clip (see pro-editing), mute clip audio, verify inspect_timeline
```

- **Anchor on the first beat, not frame 0** — songs have lead-in; `beats[0]` may be >0.
- **Hit the drop:** find the downbeat at the drop and put your hardest visual change (or a [[transitions]] whip/flash) exactly on that frame.
- Pacing: on every beat ≈ fast hype; every 2 beats ≈ standard reel; every 4 (or downbeats) ≈ cinematic.
- To retime an existing hard-cut edit onto beats, `split_clips` at beat frames or `move_clips` each boundary to the nearest beat frame.

## Gotchas

| Symptom | Cause | Fix |
| :-- | :-- | :-- |
| Cuts feel "off" by a beat | anchored at frame 0, not `beats[0]` | offset by the first beat / lead-in |
| Drift accumulates across the reel | rounded each boundary independently | share one rounded frame between adjacent clips |
| Beats land at wrong time | forgot the music clip's trim/start offset | `M + (beatSeconds − t0)×fps` |
| Cut count doesn't match beats | used `beats` when you meant bars | `downbeats` = one per bar (accents) |
