# 11 — Final Project

A 10–20 second stylised animated short. Pick one track from module 10.
This module is a checklist + template, not new concepts.

## Success criteria

You pass the course when you have:

1. A project folder with the structure below, populated.
2. A final rendered `.mp4` at 1080p (or upscaled to 1080p from native).
3. A short `post-mortem.md` describing what you tried, what worked,
   what you would redo.

Nobody grades this. You grade this. Be honest.

## Project folder template

Create (inside your ComfyUI output area or anywhere convenient):

```
my-short/
  storyboard/
    shot-01.png
    shot-02.png
    ...
  prompts/
    scaffold.md           # your reusable prompt scaffold
    character.md          # fixed character description block
    style.md              # fixed style description block
  keystills/
    shot-01-beat-a.png
    shot-01-beat-b.png
    shot-02-beat-a.png
    ...
  workflows/
    t2i-keystill.json
    i2v-firstframe.json
    i2v-firstlast.json
    rife-interpolate.json
    upscale.json
  raw/
    shot-01.mp4
    shot-02.mp4
    ...
  edit/
    project.drp           # DaVinci Resolve project, or equivalent
    audio/                # music, foley
    graphics/             # title cards, end card
  final/
    my-short_v1.mp4
    my-short_v2.mp4
  shots.csv               # the shot database
  post-mortem.md
```

## Step-by-step plan

Target timeline if you are new to this: 2–3 evenings for the plan,
3–5 evenings for generation, 1–2 evenings for edit + sound. Do not
rush. Waiting for a generation is not a reason to queue up another
one in panic — use the wait to review the previous one.

### Step 1 — Write a one-paragraph story

A genuine paragraph, not a logline. Who, where, when, what happens,
how it ends. Example (Track A):

> Night. A rooftop gargoyle in a fictional city. Spider-Man crouches
> low, listening. A distant alarm triggers; he looks up, eyes
> narrowing behind the mask. He leaps, swings between skyscrapers in
> a parallax blur, and lands on a darker rooftop where a silhouetted
> villain waits. Impact frame. Cut to hero landing stance, web-line
> trailing, logo-friendly composition.

20 seconds of screen time fits maybe 6–8 shots. More than that and
each shot gets starved.

### Step 2 — Break into shots

Make `shots.csv`. Columns:

```
id, start_sec, end_sec, type, description, seed, loras, key_image, status
```

Where `type` is one of `T2V` / `I2V-first` / `I2V-first-last` /
`vid2vid`. Fill in everything you can up front. `seed` can be blank
until generation.

### Step 3 — Produce style reference

Before any shot generation, produce one "look test" — a single
keystill with your full LoRA stack, at final resolution, with your
hero character in a neutral pose. This image is your north star.
Every shot should feel like it belongs in the same film as this one.

If your look test is not strong, the film cannot be. Fix the look
before proceeding.

### Step 4 — Keystill generation

For each shot, produce the beat stills (1–3 per shot, depending on
whether you are doing I2V first-only or first+last). Use the T2I
workflow. Keep the prompt scaffold consistent.

Curate ruthlessly. A decent shot from a good keystill is easier than
a heroic save from a bad keystill.

### Step 5 — Shot generation

For each shot: load keystills into the right I2V workflow, prompt
the motion, generate. Save the raw mp4 to `raw/shot-XX.mp4`. Log the
seed and any settings tweaks in `shots.csv`.

Generate in batches of 3–4 variations per shot. Pick the best.

Common mid-project judgement calls:

- A shot is almost-right. **Re-generate with a nearby seed (+/-1, +/-7)
  before changing the prompt.** Prompt changes are a bigger hammer.
- A shot is persistently wrong. **Simplify it.** Fewer elements, less
  ambitious camera, shorter duration.
- A shot is great but slightly off-model. **Accept it** if it does not
  break the sequence. Perfect is the enemy of finished.

### Step 6 — Post per shot

For each raw shot, in order:

1. Optional: RIFE to doubled fps.
2. Optional: upscale to 1080p.
3. Trim head/tail frames that drift or flicker (this is normal —
   expect to trim 2–5 frames).

### Step 7 — Edit

In DaVinci Resolve (or equivalent):

1. Lay shots on timeline in story order.
2. Lay music bed underneath.
3. Adjust shot durations to sit on musical beats.
4. Add title card (2s) and end card (1–2s).
5. Add foley: one whoosh per camera move, one hit per impact, one
   ambient bed.
6. Color grade: single adjustment node for the whole timeline first.
   Per-shot grading only if needed.
7. Export 1080p, h.264, 24 fps (or your chosen rate), 20 Mbps.

### Step 8 — Review, iterate, deliver

Watch the export on a different device (phone, TV) than the one you
generated on. Problems you didn't see on your monitor will pop out.

Re-generate shots you hate. Re-edit timing you hate. Do this twice
at most. Then deliver.

### Step 9 — Post-mortem

Write `post-mortem.md`. Sections:

- **What worked.** Be specific. "The crash-zoom prompt with the
  `90s-saturday-morning-animation` LoRA at 0.85 produced clean
  results on seed 4112."
- **What didn't.** Be specific. "Two-character shots drifted
  identity within 2 seconds — avoided by shooting coverage single."
- **What I'd change.** One or two things. Not a redesign — one or
  two things you would do differently on the next short.
- **Reusable pieces.** Prompts, workflows, LoRAs, seeds, settings
  that deserve to graduate into your permanent toolkit.

This file is the real deliverable. The video impresses once; the
post-mortem compounds.

## If you get stuck

- Read the earlier modules again. Almost every "stuck" moment maps
  back to something in modules 03–06.
- Generate *less*. A stuck project is usually over-scoped. Cut 30%
  of the shots. Cut 30% of the shots that are left.
- Show a work-in-progress to someone. Describing it out loud surfaces
  what you already know but were ignoring.

## What "done" looks like

A finished short of yours, which you will not love completely, which
has real problems you can articulate, and which is in the world. The
next one will be better. That is the entire point.

Good luck. Go make your SpiderMan swing.
