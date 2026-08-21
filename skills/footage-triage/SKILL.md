---
name: footage-triage
description: Use when selecting clips for an edit from a large or repetitive footage pool in Palmier Pro — avoiding near-duplicate/similar shots, ensuring shot-type variety, choosing among many lookalike clips, or when a cut feels repetitive because the same source got reused.
---

# Palmier Pro footage triage & shot diversity

The repetition problem is almost never the final pick — it's **skipping the sample → tag → dedup pipeline** and grabbing whatever's longest/first, then reusing a handful of sources because you under-sampled. Run this pipeline **before placing a single clip**. Read [[pro-editing]] first.

## The pipeline (do all four, in order)

**1. Sample WIDE, not by file size.** Raw camera files (`C0001…C0200`) are sequential in capture time — grabbing the 10 biggest files gives you 10 clips from one stretch of the night. Instead spread across the whole span (every Nth file + known highlight moments), and use `search_media` (semantic footage search) to pull by **concept** — `"DJ"`, `"hands in the air"`, `"bar"`, `"exterior"`, `"single person"`, `"walking in"` — so you get different rooms, songs, subjects, and light.

**2. Tag each candidate** from its `inspect_media` storyboard: **subject** (crowd-wide / crowd-tight / single subject / DJ / performer-on-mic / detail-insert / venue-establishing), **size** (wide / medium / close), **motion**, **dominant colour/light** (warm wash vs cool vs colored).

**3. Dedup — cluster and cull.** Group near-identical shots (same subject + size + colour) and keep only the **1–2 strongest per cluster**. Six warm crowd-medium-dance shots collapse to one or two slots. This is the step that kills "12 similar shots in a row."

**4. Coverage plan before filling.** Assign shot **types** to the edit's beats first (establish → build → talent → character → peak → button), then drop the best deduped clip into each. Enforce a **wide→medium→close** size rhythm and include **contrast beats** (a cool/venue/detail shot among the warm crowd) so the eye keeps getting something new.

## Selection rules (enforce while placing)

- **Reuse cap:** each source clip used **at most once** — unless it's a deliberate bookend/motif (e.g. the same chant opening and closing).
- **No adjacent twins:** two neighbouring clips must not share subject **and** size.
- **Earn every crowd-wide:** they read the same; ration them and break them up with talent/character/detail.

## Gotchas

| Symptom | Cause | Fix |
| :-- | :-- | :-- |
| Reel feels repetitive | picked many near-duplicate crowd shots | dedup by subject+size+colour, 1–2 per cluster |
| Same source appears 3× | under-sampled the pool, ran out of variety | sample wide + `search_media` by concept first |
| "All one vibe" | every clip is the warm crowd wash | add cool/venue/detail contrast beats |
| No establishing/story | grabbed only peak-energy shots | coverage plan includes establish + button + character |
| Picked by which clip is longest | size ≠ quality or variety | tag and choose by shot type, not duration |
