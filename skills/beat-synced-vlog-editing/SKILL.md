---
name: beat-synced-vlog-editing
description: Cut vlogs and montages to music inside Palmier Pro — pick an anchor moment, do the drop math so the music's biggest beat lands exactly on it, put cuts on downbeats, size clips in whole beats, and keep the sync intact through later edits. Covers music entrances/exits (they belong on cuts), ducking under dialogue and diegetic sound with volume keyframes, and handing off from SFX to music. Uses detect_beats, move_clips, set_keyframes, and add_clips. Use whenever the user wants an edit synced to music — "make the cut hit the beat," "land this moment on the drop," montage pacing, day-in-the-life or travel vlog edits with a music bed, or when music feels off and cuts don't land.
---

# Beat-Synced Vlog Editing

## Core principle

The drop is the spine of the edit. Pick ONE hero moment in the footage (the reveal, the badge-in, the goal) and ONE musical moment (the drop, the first chorus downbeat) and lock them together with arithmetic, not feel. Everything else is placed relative to that lock: cuts land on downbeats, clip lengths change in whole beats, and any later edit that shifts content must shift the music by the same amount or the lock silently breaks. The second rule, learned the hard way: **audio entrances belong on cuts.** A music bed that starts mid-scene reads as unmotivated no matter how gentle the fade; move the entrance onto a picture cut and the same fade suddenly feels intentional.

---

## Step 0: Pre-flight

1. `get_timeline` — note `fps` (all beat→frame math below uses it) and total frames.
2. `get_media` — identify the music asset. `import_media` it if needed (`folder: "Audio"`).
3. `detect_beats({ mediaRef: MUSIC_ID })` — returns `bpm`, `beats[]`, and `downbeats[]` in SOURCE seconds. Runs on-device, no subscription. Two traps in the output:
   - **Treat `bpm` as approximate.** The reported bpm can disagree with the actual array spacing by ~2%, which drifts nearly a second over a minute. Derive the working grid from the arrays: `framesPerBeat = median(diff(beats)) × fps` (keep it fractional; round only final frame values), and always snap cuts to actual array members, not bpm-multiplied offsets.
   - **Expect near-duplicate and irregular entries** (downbeats 0.2s apart, uneven intro spacing). Build a consistent sub-grid: keep downbeats that continue the median spacing, drop the stragglers.
   - Cut on **downbeats** for section changes and the anchor; **beats** are for fast montage rhythm.
4. Find the drop. `detect_beats` gives the grid but no energy data — nothing in Palmier surfaces song structure. Locate the drop by listening, or by a quick external loudness pass (e.g. ffmpeg RMS per second) looking for a sustained level jump that starts on a downbeat. The drop second `D` you use below should be a member of `downbeats[]`. Keep the whole RMS-per-second map — you need it again for the entrance (Step 1).

---

## Step 1: The anchor — drop math

Choose the drop second `D` (a downbeat, from `downbeats[]`) and the hero timeline frame `H`. Locating `H` to the exact frame matters: `inspect_timeline`/`inspect_media` sample too coarsely to catch a moment that turns on within one frame (a light, a splash, an impact). For those, run a per-frame pass on the source (e.g. ffprobe `signalstats` luma per frame) and pick the first frame where the change has fully landed.

Place the music so the drop lands on the hero frame:

```
musicStartFrame = H − round(D × fps)
```

- If `musicStartFrame` ≥ 0 and it lands on (or can be nudged to) a picture cut: `add_clips({ entries: [{ mediaRef: MUSIC_ID, startFrame: musicStartFrame }] })`
- Otherwise, put the music's start on the picture cut you want the entrance on (frame `E`) and trim the source instead: `sourceStart = D − (H − E)/fps`, i.e. `source: [sourceStart, songEnd]`. (`E = 0` gives the familiar `D − H/fps`.)

**First decide: does the music start WITH the video, or AFTER it?** Abruptness is only possible when music enters after the video has begun — an entrance mid-video is an event the viewer notices, and events need motivation. Music already playing at frame 0 needs none: the viewer walked in with the song on. So the simplest correct choice is music from the very first frame, and a cold open (natural audio first, music enters later) should exist only when the footage earns it — an opening beat whose own sound carries it (an alarm, a click, a spoken line). If you use a cold open, everything below applies to the entrance; if music starts at frame 0, the build requirement relaxes (there is no "before" to contrast with), though a rising start still lands the drop better than a flat one.

**Every mid-video entrance needs a build.** A song entrance feels natural only when the music RISES into its groove; enter at flat cruise or mid-chorus intensity and the song "starts out of nowhere," no matter how clean the cut or how long the fade. This does NOT mean starting the song at 0:00 — trendy edits almost never do. It means `sourceStart` must sit at the bottom of a climb. In priority order:

1. **The build before your drop.** The ideal `sourceStart` is the energy valley preceding the chosen drop — the breakdown/riser boundary, visible in the RMS map as the local minimum before the drop's level jump. Risers are buildups by design: the song spends its natural crescendo climbing under your pre-drop content and arrives at the drop with you. Runway needed = `D − riserStart` seconds; size the pre-drop content to fit it, or pick the drop whose riser length matches the content you have. This is the standard trendy-edit entry.
2. **Any other rising section.** If that riser is too short/long for your content, use the RMS map to find another climb that fits: a verse start, a post-chorus lull, the song's own intro (just the first build — right when the runway is long or the vibe is chill). The test is the same everywhere: energy at `sourceStart` should be a local minimum with a rise ahead of it.
3. **Manufactured buildup.** If the song offers no usable rise for your runway, fake one: a 2–4 bar swell (−60 dB → cruise over 4–8 s, much longer than a normal fade), ideally beginning quietly UNDER the opening shot's natural audio and swelling as it takes over — an extended handoff, so the song rises out of the scene instead of starting at it.

Whichever you choose, land `sourceStart` on a downbeat (`sourceStart = 0` is exempt — the song's own first frame needs no snapping) — trim in whole bars from that musical spot, not an arbitrary offset. Choose drop and `sourceStart` TOGETHER: the runway between them is a musical decision, not a leftover of the math.
- **Read back the trim.** `add_clips` floor-rounds source seconds to `trimStartFrame` (which is in PROJECT frames, not source frames — matters for 60fps sources on a 30fps timeline), and rounding can silently move the drop by a frame. After placing, `get_timeline`, check `trimStartFrame == round(sourceStart × fps)`, and correct with `set_clip_properties({ trimStartFrame })` if not.

If the music clip is trimmed or retimed, the general beat→timeline conversion is:

```
timelineFrame = musicClip.startFrame + (beatSeconds × fps − trimStartFrame) / speed
```

Verify with `get_timeline` that the computed drop frame equals `H` before touching anything else. Beat times are usually fractional in frames, so a sub-frame residual (<0.5 frame) on non-anchor cuts is inherent — anchor the drop exactly and let the rest carry the rounding.

---

## Step 2: Cuts on downbeats, lengths in whole beats

- Snap section cuts to the nearest downbeat frame; snap montage cuts to beats. Compute the target frame with the formula above, then place/trim clips with `add_clips` `source` spans or `split_clips` + `ripple_delete_ranges`.
- When a clip needs to breathe longer or tighter, change its length by whole multiples of `framesPerBeat` (round the cumulative position, not each increment). Arbitrary extensions feel wrong even when no single cut is visibly off.
- **The action picks the length; the grid only rounds it.** A grid-legal cut can still amputate a gesture mid-motion, and no numeric check will catch it. If the motion doesn't complete within N beats, give it N+1 — never trim content to fit the bar. This matters most at the FIRST and LAST cut of the piece: verify with `inspect_media` that the source span you chose contains the complete gesture AND a settle (~0.5–1s of stillness after the motion lands) before committing the span. An opener that cuts mid-motion reads as a glitch; an ending that stops the frame a motion completes reads as chopped.
- Quick cuts are allowed to subdivide (half-beats) where speed is the joke; slow scenic shots get 2–4 bars.

---

## Step 3: Editing later without breaking the lock

Any ripple edit before the drop moves the hero frame. The prime rule: **after every structural edit, `get_timeline` and re-derive the music's actual `startFrame`/`trimStartFrame` — never assume.** Track sync-lock is on by default but not reported in `get_timeline`, so how the music reacts to an edit is only knowable by reading it back. What to do depends on WHERE the edit lands relative to the music clip:

**Edit before the music clip's start** — content and music must shift together by the same `N`:

- With `insert_clips`, sync-lock usually ripples the music automatically — verify the music moved by `N`, and do NOT also move it yourself (double-shift breaks the lock).
- With `move_clips` bulk shifts, music will NOT follow: `get_timeline`, collect every clip with `startFrame ≥ cutoff`, sort **descending** by startFrame, one `move_clips` call moving each to `startFrame + N` (descending prevents transient overlaps trimming neighbors mid-batch; linked audio follows its video), then move the music clip by `N` yourself.
- Either way, finish by recomputing the drop's timeline frame against the shifted hero frame.

**Edit inside the music clip's span (after the entrance, before the drop)** — expect breakage: `insert_clips` here **splits the music clip** and leaves a silent gap on the trailing half with a bisected keyframe envelope; the inserted clip's linked audio may also land on the music's track. Do not heal the halves in place. Rebuild:

1. Remove the split music fragments; mute or relocate any stray linked audio on that track.
2. Re-add ONE continuous music clip at the SAME `startFrame` (the entrance stays on its cut) with the source retrimmed **earlier by exactly N frames** — the same clip supplies the extra runway, so there is no skip or gap, and every beat including the drop lands `N` frames later, tracking the shifted content.
3. Make `N` a whole number of bars so downbeat phase survives (size the insert to the bar; `bpm`/`fps` are rarely commensurable, so landing within ~0.5 frame of the bar is the practical target and inaudible).
4. Re-send the full volume envelope with post-entrance points shifted `+N`.

**Edit entirely after the drop** — the lock is untouched; change nothing about the music's start or start-trim. Only re-fit the tail: `set_clip_properties({ durationFrames })` on the music clip (extending pulls more source; land the new out point on a downbeat), then re-send the FULL volume envelope with the fade-out moved to the new end — keyframe rows replace wholesale, so a partial send drops the rest of the mix.

Never include the music clip in a content bulk-move batch — move it in its own call. And after any `durationFrames` change, `trimEndFrame` in the readback goes stale (the real out point is `trimStartFrame + durationFrames`); verify such clips by rendering (`inspect_timeline`), not trim arithmetic.

One more silent hazard: a later `add_clips` that lands on the track holding the music (or holding another clip's linked audio) **overwrites** what's there without warning. Keep the music on its own track and re-check it after any bulk placement. There is no explicit create-track operation (`manage_tracks` only removes/reorders, and `add_clips` rejects an out-of-range `trackIndex`), so the sequence that gets you a dedicated music track is: place video clips first (their linked audio auto-creates its own track), then add the music with `trackIndex` omitted (auto-creates another), then `get_timeline` to confirm the separation.

---

## Step 4: Entrances, exits, and ducking (volume keyframes)

All music dynamics are `set_keyframes` on the music clip, property `volume`, frames CLIP-RELATIVE (0 = clip's first frame — keyframes survive `move_clips`).

**Unit trap, the one that silently ruins the mix:** `set_keyframes` volume values are interpreted as **decibels**, even though the tool description says 0.0–1.0. An envelope written in linear 0–1 renders as a flat ≈0–1 dB — no fade-in, no duck, no fade-out — and every non-listening check will still pass. (`set_clip_properties` volume IS linear 0–1; the two tools disagree.) Write keyframe envelopes in dB: silence ≈ −60, cruise −6 to −3, prominent −2, duck −14 to −10.

```
set_keyframes({
  clipId: MUSIC_CLIP_ID,
  property: "volume",
  keyframes: [
    [0, -60],          // silence; entrance ON a picture cut
    [45, -6],          // ~1.5s ramp at 30fps to cruise
    [105, -3.5],       // settled level
  ]
})
```

- **Entrance on a cut.** If the song's own opening has a sharp attack, lengthen the ramp (2–3.5s) rather than lowering the target level. Whenever the music's start frame moves, re-listen/re-check the entrance — a fade tuned for one entry point rarely survives a shift.
- **Ducking — separate the mechanics from the taste.** The mechanics, whenever a duck happens: drop 7–12 dB below cruise over ~0.3s, hold, recover over ~0.5s, and anchor the duck's END to the picture cut that ends the moment, so later trims inside the moment don't leave the recovery floating mid-shot. When to duck is two different questions: under **speech** (dialogue, a spoken line), ducking is universal craft — always do it. Featuring **non-speech moments** (a pour, a crowd, ambient sound) with a duck is the creator's taste, not a rule — take it from the brief, and if the brief doesn't say, ask rather than assume. A skill should carry craft; style choices belong to the person whose edit it is.
- **SFX → music handoff** (alarm, ambient intro): let the SFX own the open and fade it at the cut where the music enters. One bed at a time; overlapping beds read as mud.
- Keyframe rows replace the whole track for that property — re-send the full envelope, not a patch.

---

## Step 5: Verify

- Recompute every landed moment (drop, section cuts) numerically from `detect_beats` output + clip positions; don't trust eyeballing for sync.
- Re-read what the tools actually did. Every source-seconds→frames conversion floors: music trim (Step 1), VIDEO clip trims, and clip durations — a 60fps source span on a 30fps timeline routinely comes back one frame short, leaving 1-frame gaps (fix with `durationFrames`), and a video `trimStartFrame` can floor a half-frame early. Read back `frames` and trims on every placed clip, not just the music.
- `inspect_timeline` around the anchor for the picture side: the hero moment's first frame should be the drop frame exactly, not ±1–2 ("close" is visible at full speed).
- Export a draft (`export_project`, poll `manage_exports {action:"list"}`) and watch the drop region with sound before calling it done.
- **Verify the mix in the export, not the timeline.** Nothing on the timeline reveals a broken envelope (see the dB trap in Step 4) — every metadata check passes while the audio renders flat. If you can't listen, measure: decode the export's audio, align it against the raw song (`sourceTime = exportTime − entranceTime + trimSeconds`), and compare RMS per region. The measured gain should match each envelope segment (cruise, duck, fade) within ~1 dB, and the fade regions should actually descend.

---

## Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Drop lands early/late after edits | Content rippled, music didn't (or you double-shifted a sync-locked ripple) | Read back music start/trim, re-derive, apply the matching Step 3 case |
| Music bed gapped/split after insert | Insert landed inside the music span (splits it), or linked audio landed on the music track | Rebuild: one continuous music clip, retrim earlier by N, full envelope (Step 3) |
| Drop off by exactly 1 frame | Source-seconds trim floor-rounded | Read back `trimStartFrame`, correct via `set_clip_properties` |
| Music renders flat — no fade-in, no duck | Envelope written in linear 0–1; keyframe volume is dB | Rewrite envelope in dB (silence −60, cruise −6..−3, duck −14..−10) |
| Opener/ending feels chopped despite grid-perfect cuts | Source span amputates the gesture (grid picked the length) | Re-span to the completed motion + settle; add a beat rather than trim the action |
| Music start feels "weird" / unmotivated | Entrance mid-scene | Move entrance onto the nearest picture cut, then re-tune the fade |
| Entrance feels sudden even ON a cut | `sourceStart` has no build — flat or full-intensity material | Re-pick per Step 1: riser before the drop > another rising section > 2–4 bar swell |
| Fade felt right, now clips harshly | Music start moved after the fade was tuned | Re-check the entrance envelope after every music move |
| Duck recovery floats mid-shot | Duck anchored to content that later moved | Re-anchor duck end to the closing cut |
| Cuts feel subtly off-grid | Per-clip rounding accumulated | Round cumulative beat positions, not each increment |
| Beat math off by trim | Forgot `trimStartFrame`/`speed` in conversion | Use the full formula from Step 1 |
| Whole timeline shifts trimmed a clip | Bulk move in ascending order | Re-do batch moves sorted descending by startFrame |
| No beats returned | Track is speech/ambience, or wrong mediaRef | detect_beats needs music; check the asset |

---

## Pacing benchmarks

- Hero moment locked to the drop; everything else serves that lock.
- Section change every 4–8 bars; montage cut every 1–2 beats; scenic holds 2–4 bars.
- One music bed at a time; SFX may overlap it only during a handoff or a duck.
- Music cruising level under dialogue-free vlog footage: −6 to −2 dB. Under speech: duck to −14 to −10 dB. Whether non-speech moments get featured (and ducked for) is the creator's call — see Step 4.
