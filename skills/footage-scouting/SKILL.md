---
name: footage-scouting
description: Use when scouting raw or long footage in Palmier Pro before cutting — inspecting clips efficiently with inspect_media (overview storyboards then windowed sampling), reading the audio transcription for content and drops, and building a grounded, timestamped shot list of what is ACTUALLY on screen. The "how to look" phase that feeds footage-triage (which to keep) and reel-mechanics (how to place).
---

# Palmier Pro footage scouting

Turn an unseen footage pool into a **grounded, timestamped shot list** with the fewest, cheapest looks. This is the phase *before* [[footage-triage]] (diversity/dedup) and [[reel-mechanics]] (assembly). Read [[pro-editing]] first.

## The one rule: ground everything

**List only moments that are actually on screen.** Never invent scenes, story, or a vibe the footage doesn't show. Every shot-list entry is a real timestamp you saw in `inspect_media`. If the director says "don't make scenes up," this is already how you work; if they say "skip clip X," drop it from scouting entirely — don't sample it.

## inspect_media, cheapest-look-first

Every `inspect_media` returns images (heavy). Spend looks in this order:

1. **Short clips (≤ ~30s):** one pass, `maxFrames:6–8`. Read the sampled frames + any `transcription.segments`.
2. **Long takes (minutes):** `overview:true` first — one storyboard tile-strip + the whole sentence-level transcription. Read the segments as prose to find where the good stuff is.
3. **Zoom in:** re-call with `startSeconds`/`endSeconds` windows across the clip (`maxFrames:8` each). **Windowed calls only sample/transcribe that span, so they're fast** — spread 3–4 windows over a long take instead of one dense full-length pass.
4. **Exact cut points (talking-head):** `wordTimestamps:true` over the span you'll cut, to trim on the word.

Long takes are mostly filler with a few gold moments — window-scan, don't full-sample. Log it when you cap coverage ("scanned 3 windows of an 18-min take, skipped the rest").

## Read the transcription, not just the pictures

`inspect_media` transcription (sentence segments in **source seconds**) tells you what the pictures can't:
- **Drops / hype / call-outs** — where energy peaks, for the hook and the beat-drop cut.
- **Retakes & filler** — same line twice, "um", false starts (for talking-head cutting).
- **Brand safety** — flags explicit lyrics/speech so you know NOT to use that clip's live audio under a public post (swap in a clean bed instead).

## The shot list (the deliverable)

One row per real moment, in **source seconds** (not frames — that conversion happens at placement):

```
{ mediaRef, inSec, outSec, inFrame_of_action, shotType, note }
// e.g. { EBAA5861, 196.0, 196.6, "headphone-adjust hero gesture", close, "clean, front" }
```

Tag `shotType` (establish / wide / medium / close / detail / two-shot / talent-hero) so [[footage-triage]] can dedup and coverage-plan, and so [[reel-mechanics]] can assign shots to beats. For static locked-off cameras the exact in-point barely matters (any sub-span shows the action) — capture the *kind* of moment and a rough second.

## Positions: SOURCE seconds vs TIMELINE frames

`inspect_media` timestamps, `transcription` segments, and `search_media` hits are **source seconds**. `get_timeline`/placement are **project frames**. Keep the shot list in seconds; convert only when you place (`add_clips … source:[inSec, outSec]`). Don't multiply by fps in the list.

## Gotchas

| Symptom | Cause | Fix |
| :-- | :-- | :-- |
| Burned looks (cost/time) sampling a long take | dense full-length `inspect_media` | `overview:true`, then 3–4 windowed passes |
| Shot list has moments that aren't there | described a vibe, not the frames | ground every entry to a timestamp you saw |
| Wrong caption text later | used a clip's explicit live audio as the bed | flag it in scouting; use a clean generated bed |
| Cut points feel off on talking-head | eyeballed instead of reading words | `wordTimestamps:true` over that span |
| Placement lands on the wrong moment | mixed source-seconds and project-frames | keep the list in seconds; convert only at `add_clips` |
| Scanned everything, still repetitive | scouting ≠ selection | hand the tagged list to [[footage-triage]] |
