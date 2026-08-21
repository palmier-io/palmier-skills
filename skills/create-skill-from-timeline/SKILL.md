---
name: create-skill-from-timeline
description: Reverse-engineer a finished Palmier Pro timeline into a reusable SKILL.md that recreates the same edit from the same footage with no prior context, or applies the same style to new footage with minimal tweaks. Use when the user asks to create a skill, save a template, distill this edit into a playbook, document how this video was built, or turn a timeline / chat session into a reusable workflow.
---

# Create a skill from a timeline

A skill is a playbook another agent can follow with **no prior context**. Superficial notes ("add captions", "make it vertical") fail. Capture exact values **and** editorial judgment — what was chosen, cut, kept, and why.

## Success criteria

A good skill must support both:

1. **Same footage, zero context** — another agent rebuilds this video from the original media using only the skill.
2. **New footage, same style** — another agent produces the same kind of edit on different sources with only light adaptation notes.

If either fails, the skill is incomplete.

## Workflow checklist

```
- [ ] 1. Confirm scope with the user (which timeline / version)
- [ ] 2. Gather history — chat + tool trail that produced the edit
- [ ] 3. Inventory the finished timeline (get_timeline, tracks, nesting)
- [ ] 4. Inventory raw library media (get_media, inspect_media)
- [ ] 5. Diff source vs timeline — selection criteria
- [ ] 6. Read the spoken arc (get_transcript) — keep/cut rules
- [ ] 7. Capture hard numbers — layout, text, color, keyframes, cams
- [ ] 8. Infer structure + pacing + tool path
- [ ] 9. Draft SKILL.md (frontmatter + exact procedure)
- [ ] 10. Self-test: could a stranger recreate this from the skill alone?
```

---

## Step 1: Confirm scope

Ask only if unclear:

- Which timeline? (active vs a named alternate / 9:16 version)
- Skill goal: recreate this piece, generalize a style, or both?
- Anything off-limits (brand assets, one-off jokes, temp VO)?

Then lock that timeline with `set_active_timeline` if needed and re-read `get_timeline`.

---

## Step 2: Recover how it was built

Prefer evidence over invention:

1. **Chat / tool history** in this session — layouts chosen, caption styles, cut passes, model prompts, rejected alternatives.
2. **User intent statements** — "tighter", "speaker cam only", "hook first" — these become rules, not vibes.
3. If history is missing, reconstruct from the timeline alone and say so in the skill under **Assumptions**.

Record the **tool path** that should be reused:

| Pattern in the edit | Prefer in the skill |
|---|---|
| Vertical / aspect reframe, PIP, split, grid | `apply_layout` (+ anchors) — never hand `set_clip_properties` transforms |
| Multicam angle program | `manage_multicam` → `change_cam` |
| Dead air / fillers | `remove_silence` then `remove_words` |
| Spoken captions | `add_captions` + exact `update_text` style block |
| Authored titles / logos | `add_texts` with exact transform + style |
| Grade / FX | `apply_color` / `apply_effect` with exact params |
| Nested versions | `create_timeline` / sequence clips |

---

## Step 3: Inventory the finished piece

```
get_timeline()
inspect_timeline({ startFrame, endFrame, maxFrames })  // key beats
```

Extract and write down:

- Canvas: fps, width, height, aspect
- Track stack (bottom → top), names, types, what's on each
- Clip order with `[start, end)`, mediaRefs, speeds, trims, links
- Nested sequences (`mediaType: 'sequence'`) and what they contain
- Multicam groups + angle program over time
- Text / caption groups: content samples, `captionGroupId`, shared style
- Color / effects / keyframes present on hero clips
- Music / VO / SFX placement

`inspect_timeline` at the hook, mid-body, and end — verify what the viewer actually sees.

---

## Step 4–5: Raw media vs timeline selection

```
get_media()
inspect_media({ mediaRef, overview: true })
inspect_media({ mediaRef, wordTimestamps: true })  // when speech drives the cut
```

Compare library assets to timeline clips and answer:

- What was **selected** vs left unused?
- Within a long take, which **source spans** were kept? (`source` seconds / trims)
- Any B-roll / stills / generated assets inserted — on which spoken beats?
- Sync relationship (multicam, dual-system, linked A/V)?

Write a **Selection** section: concrete rules ("keep first clean take of each claim", "drop second explanation of pricing", "B-roll on every proper noun after the hook").

---

## Step 6: Spoken arc and cut rules

```
get_transcript()
```

Read it as prose. Document:

- **Structure** — hook → body → CTA? cold open? summary then proof? straight into action?
- **Keep** — sentences / claims that define the piece
- **Cut** — fillers, false starts, repeated explanations, throat-clearing, long pauses
- **Join style** — soft landings at sentence ends vs tight mid-thought joins
- **Silence policy** — `remove_silence` defaults or explicit `minimumPauseSeconds` / `speechPaddingSeconds`
- **Word policy** — filler list, retake handling, whether "like"/"you know" stay when intentional

After describing cuts, the remaining transcript must still read as continuous sense. Put that as an explicit verification step in the skill.

---

## Step 7: Hard-code visual and timing details

Guessable adjectives are useless. Paste numbers.

### Layout / framing

- Exact `apply_layout` layout id, slot → clip mapping, `fit`, `anchor` / `anchorX` / `anchorY`
- Track order after layout (what sits on top)
- For alternate aspect ratios: `set_project_settings` / duplicate-via-`create_timeline` then layout — write that sequence

### Text and captions

- Font PostScript name, size, weight traits, tracking, lineSpacing, alignment, case
- Colors (hex), outline / shadow / background blocks exactly
- `fillMode` (`color` / `footage` / `inverted`), `style.blur`, widthScale / heightScale
- Transform: alignment-relative `x`, `y`, `rotation`, `rotationX`, `rotationY`
- Animation preset + highlightColor when used
- Caption placement relative to speech (lower-third, center punch-ins, one-word vs phrases)
- Prefer a known caption-templates preset **verbatim** when it matches; otherwise paste the full style object

### Motion / cams / grade

- Keyframe tracks: which properties, at which transcript beats or frame offsets
- Multicam: when to cut (speaker change, mid-sentence, reaction) — and what **not** to do (e.g. never cut only on sentence ends)
- Color: full `apply_color` objects worth copying; LUT / effect params

### Brand / logos / lower thirds

Exact position, size via layout or text transform, duration, entrance timing relative to speech. No "put the logo top-right".

---

## Step 8: Structure the skill document

Write `SKILL.md` with:

```markdown
---
name: <hyphenated-id-style-title>
description: <when to use; include trigger phrases; one dense paragraph>
---

# <Title>

## When to use
…

## Inputs required
- Media roles (A-cam, B-roll, mic, music, logos) — not filenames

## Editorial rules
- Structure, keep/cut, pacing, cam cadence, text landing rules

## Exact look (copy these values)
- Layout / caption / text / color blocks with numbers

## Procedure
1. Tool calls in order…
2. Verification: get_timeline + inspect_timeline + transcript sense-check

## Adapting to new footage
- What stays fixed vs what to re-derive from the new transcript
```

### Writing rules

- **Imperative, tool-named steps** — `apply_layout({...})`, not "reframe nicely"
- **One source of truth** for each value — don't restate conflicting sizes
- **Reuse existing skills** when they already encode a sub-procedure (`caption-templates`, `multi-cam-editing`, `color-grading`, `ugc-editing`) — call `read_skill` then specialize
- **No marketing language** — match Palmier's terse editor voice
- Frontmatter `description` must list trigger phrases so the agent selects this skill later

---

## Step 9: Self-test before handing off

Ask yourself:

1. Could someone rebuild this with **only** the skill + the original media refs / roles?
2. Are logo / caption / layout numbers exact enough to match `inspect_timeline`?
3. Are cut rules specific enough that two agents would remove the same retakes?
4. Does the procedure use `apply_layout` (and other preferred tools) instead of fragile manual transforms?
5. Is there an adaptation section for new footage that doesn't undo the hard-coded look?

If any answer is no, dig deeper — usually selection criteria, cam cadence, or numeric text/layout values are missing.

## Deliverable

Return the full `SKILL.md` contents (frontmatter + body) ready to paste into Settings → Skills, or into `skills/<id>/SKILL.md` for the community catalog. Keep commentary to one short sentence; the skill file is the product.
