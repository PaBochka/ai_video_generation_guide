# 08 — Multi-LoRA Workflows

Single LoRAs are easy. Stacks are where the craft lives. This module is
about using 2–5 LoRAs together without them fighting.

## Why stack

Typical useful stacks:

- **Style + Character.** A 90s anime style LoRA + a character LoRA of
  your specific hero. Style comes from one, identity from the other.
- **Style + Environment.** A film-stock look + a specific setting
  (cyberpunk city, medieval village).
- **Style + Motion.** A visual style + a motion LoRA that biases
  toward specific camera moves or action types.
- **Style + Style (blend).** Two styles at half strength each, to get
  an interpolated look. Delicate; works sometimes.

## The basic stacked graph

```
   Checkpoint
        |
   LoraLoader(style)        strength_model 0.8 / strength_clip 0.8
        |
   LoraLoader(character)    strength_model 0.7 / strength_clip 0.6
        |
   LoraLoader(motion)       strength_model 0.5 / strength_clip 0.3
        |
   Sampler
```

Order matters less than people on forums claim — the math is additive
low-rank deltas — but organising by "general to specific" (style ->
character -> shot-specific) makes the graph easier to read.

## Conflict resolution

When two LoRAs overlap in what they change, they compete. Symptoms:

- Style degrades when character LoRA is added.
- Character identity degrades when style LoRA is raised.
- Output becomes muddy, over-saturated, or weirdly plastic — all three
  at once.

Playbook:

1. **Disable all LoRAs except the primary for the shot.** Does the
   output look right? If not, the primary is the problem; fix that
   before stacking.
2. **Add LoRAs back one at a time.** After each add, check output.
3. **If a new LoRA breaks things, drop its strength by 0.2 and
   retry.** Keep going until it contributes without breaking.
4. **If two LoRAs fundamentally disagree** (e.g. two style LoRAs with
   very different centers), accept that the stack is wrong. Pick one.

Using `rgthree-comfy`'s bypass/mute nodes makes this fast — you can
click a single LoRA on/off without rewiring.

## The "anchor + accent" pattern

Cleanest mental model for stacks of 2–3:

- **Anchor LoRA** (0.8–1.0) — the dominant look. Usually style.
- **Accent LoRA(s)** (0.3–0.6) — nudges. Character, motion, environment.

Do not run two anchors. If you want two strong styles, blend them via
image-space composition (region prompting, img2img passes) rather than
dual-LoRA overcrank.

## Region-scoped LoRAs (T2I stage)

At the T2I stage — where you make your key stills — regional LoRAs are
possible via node packs like ComfyUI's regional conditioning. You tell
the graph:

- Apply character LoRA only to the left half of the frame (where the
  character stands).
- Apply environment LoRA only to the background mask.

This is mature for SDXL/Flux and less so for LTX-2.3. In practice, use
regional LoRAs at the T2I stage for clean key stills, then I2V the
still in LTX-2.3 without regional tricks.

## Time-scoped LoRAs (video stage, experimental)

Some community workflows support scheduling LoRA strength over time
(frames). Example: strength ramps from 0.0 at frame 0 to 0.8 at frame
50, for a "style fades in" effect. Treat this as experimental in
LTX-2.3 — useful for transitions, not for production shots.

If you want to try: look for `LoraScheduler` style nodes with
time-range inputs, or LTX-2.3-specific LoRA-scheduling extensions in
community custom node packs. The node name may vary — the pattern is
"LoRA with a start/end frame input".

## Blending two styles

Workflow:

```
   Checkpoint
        |
   LoraLoader(style_A)   strength 0.5 / 0.5
        |
   LoraLoader(style_B)   strength 0.5 / 0.5
        |
   Sampler
```

Rules of thumb:

- Both at 0.5/0.5 is a starting point. Good pairs work; most pairs
  don't.
- If A dominates, drop A to 0.4 and raise B to 0.6. Conserve total
  "budget" at ~1.0.
- Generate a 3x3 grid with strengths (0.3, 0.5, 0.7) on each axis.
  Eyeball. Pick the sweet spot.
- Triggers: include both sets. Some blends only come alive with both
  triggers present.

## Example stacks for this course

### Track A — 90s SpiderMan (T2I key-still stage on SDXL or Flux)

```
LoRA                              model / clip
--------------------------------------------
90s-american-animation-style      0.85 / 0.80   (anchor)
spider-man-1994-character         0.65 / 0.55   (accent — only if loading a dedicated one)
comic-book-inking                 0.35 / 0.30   (accent — strengthens outlines)
```

Triggers: put style trigger + character trigger at the front, comma
separated.

### Track B — Anime (T2I key-still stage)

```
LoRA                              model / clip
--------------------------------------------
madhouse-90s-style                0.80 / 0.70   (anchor)
character-name                    0.70 / 0.60   (accent)
cel-shading-painterly             0.30 / 0.25   (accent)
```

### Track C — Mixed media

Usually a single "film look" LoRA on the live-action plates + a
distinct stylised LoRA on the animated overlays. These are done in
two separate generations, not stacked.

### LTX-2.3 side — when LoRAs exist

As of writing, LTX-2.3-compatible LoRAs exist but are fewer. A realistic
LTX-2.3-stage stack is:

```
LoRA                              model / clip
--------------------------------------------
ltxv-style-cel-animation          0.80 / 0.80   (if available for your style)
ltxv-motion-action                0.50 / 0.40   (if available)
```

Often you will run LTX-2.3 with **no LoRAs** and rely on the I2V input
image to carry the style — which it does, well, for the first 2–3
seconds. Drift past 3s is a signal to either shorten the clip or find
an LTX-2.3-side style LoRA.

## Multi-LoRA debugging table

| Symptom | First thing to try |
|---|---|
| Character features leaking into style (e.g. everyone looks like the trained char) | Lower character LoRA clip strength; move character trigger later in prompt |
| Style LoRA overpowers character | Lower style model strength by 0.1; raise character model strength by 0.1 |
| Colours all wrong | One LoRA was trained on heavily filtered data; reduce its strength |
| Output over-saturated and crunchy | Total LoRA "budget" > 1.5; rebalance |
| LoRA seems to have no effect | Wrong base model compatibility, or trigger missing, or disabled by a bypass node |

## Exercise — Build your stack

Pick your track. On the T2I side:

1. Test each LoRA individually at strength 0.8 with a neutral prompt.
   Record what it does.
2. Stack them in "anchor + accent" order. Start with the anchor at
   0.8, accents at 0.4.
3. Generate 9 images with your project prompt. Evaluate.
4. Adjust strengths. Re-generate 9. Lock in when satisfied.
5. Save this as a T2I workflow `05-keystill-track[A|B|C].json`.

On the LTX-2.3 side: load the T2I output as I2V input, run your keyframe
workflow from module 06. If an LTX-2.3 style LoRA is available and
compatible, add it at 0.8 and compare.

## The LoRA budget — a mental model

Think of LoRA strength as a budget. Each LoRA spends some; total
spend above ~1.5 tends to produce over-cooked output. Rough
accounting:

```text
style_anchor    strength 0.85  ->  0.85 budget
character       strength 0.65  ->  0.65 budget
detail_accent   strength 0.30  ->  0.30 budget
------------------------------
total spent                        1.80  (danger zone)
```

Budgets are heuristic — two LoRAs modifying disjoint aspects (one
touching colour, one touching pose) can coexist above 1.5 without
issue. Two LoRAs touching the same thing (two style anchors) blow
up at 1.2 combined. Track it informally; when output degrades, the
first thing to reduce is total spend.

### Budget recipes

```text
Conservative (safe, minimal conflict):
  anchor 0.70 + accent 0.35           total 1.05

Standard (default for learning):
  anchor 0.80 + accent 0.45 + 0.30    total 1.55

Aggressive (only with very compatible LoRAs):
  anchor 0.95 + accent 0.60 + 0.40    total 1.95

Blend (two anchors):
  anchor_A 0.55 + anchor_B 0.50       total 1.05
```

If you don't know where you are, start Standard. Shift once you see
where the bottleneck is.

## Stack recipes — copy and adjust

Concrete known-good(ish) patterns. All are T2I-stage stacks unless
noted. Replace LoRA names with what's available for your base model.

### Recipe S1 — "90s action cartoon hero shot"

Use for Track A key stills.

```text
Base: SDXL or Flux.1
Lora 1: 90s-saturday-morning-animation       0.85 / 0.80   (anchor)
Lora 2: comic-book-inking                    0.40 / 0.30   (accent: line weight)
Lora 3: cel-shading-two-tone                 0.25 / 0.20   (accent: shadow shapes)
Triggers (prompt front): [style_trigger_1], [style_trigger_2],
Effective total spend: ~1.50
```

Watch: if colours get crunchy, drop line weight accent first; if
lines go fuzzy, raise it.

### Recipe S2 — "1990s Madhouse anime drama"

```text
Base: SDXL or Flux.1
Lora 1: madhouse-90s-style                   0.80 / 0.75   (anchor)
Lora 2: painterly-background                 0.35 / 0.25   (accent: BG only)
Lora 3: character_lora (if yours)            0.70 / 0.55   (accent: identity)
Triggers: [style_trigger], [char_trigger],
Effective total: ~1.85 (stretched — watch for overcook)
```

If character identity degrades at this spend: lower `painterly-
background` model strength to 0.25. It's the most disposable node.

### Recipe S3 — "Ghibli-esque watercolour"

```text
Base: SDXL
Lora 1: ghibli-inspired-watercolour          0.85 / 0.75
Lora 2: soft-natural-lighting                0.35 / 0.30
Total: ~1.20
```

Usually no character LoRA; Ghibli-style character faces are sensitive
and most character LoRAs blow up the look. Use prompt description
for the character instead.

### Recipe S4 — "Stylised rotoscope (mixed-media)"

```text
Base: SDXL
Lora 1: sharpie-rotoscope-ink                0.90 / 0.80
Lora 2: high-contrast-film-still             0.40 / 0.35
Total: ~1.30
```

Designed to apply to a *real-photo img2img input* at moderate denoise
(0.55–0.70). The high-contrast accent helps the rotoscope "bite"
onto photographic lighting.

### Recipe S5 — "Comic book splash page (static)"

```text
Base: SDXL
Lora 1: silver-age-comic-art                 0.75 / 0.70
Lora 2: halftone-dots                        0.25 / 0.20
Lora 3: dynamic-pose                         0.40 / 0.30
Total: ~1.40
```

Produces very strong compositions but terrible temporal consistency
when fed to I2V (halftone dots flicker). Good for the single T2I
stage; plan for that flicker downstream.

### Recipe S6 — "Two-style blend, painterly realism"

```text
Base: Flux.1
Lora 1: oil-painting-impasto                 0.55 / 0.45
Lora 2: cinematic-photograph                 0.50 / 0.45
Total: ~1.05
```

Both at roughly half strength. Produces a "photograph-of-an-oil-
painting" look. Sensitive to prompt; hero subject words drift toward
whichever style has more training precedent for that subject. Adjust
ratio per shot as needed.

## Conflict resolution — three worked case studies

### Case 1 — Character LoRA overrides style LoRA

**Symptom:** character-named LoRA at 0.7 makes everything look like
photorealistic training snapshots, wiping the anime style.

**Diagnosis:** the character LoRA was trained on photoreal data, so
its "identity" includes a photographic rendering style.

**Fix tried (and why):**
- Lower character model strength to 0.5. Identity drops, style
  returns. Acceptable if identity still reads.
- Keep character model strength at 0.7 but set character CLIP
  strength to 0.2. Identity persists via UNet, text prompts retain
  style influence.
- Regenerate the character's T2I reference image *with the anime
  style LoRA applied first*, then retrain a new character LoRA on
  those stylised images. Expensive; worth it for a long project.

### Case 2 — Two style LoRAs producing mush

**Symptom:** combining "cel-shaded" + "watercolour" at 0.5 each
produces neither — an uncanny midground with flat colour but soft
edges. Output reads as "AI slop".

**Diagnosis:** the two styles have fundamentally opposed aesthetics
(hard edges vs. soft edges, flat fills vs. bleeds). There is no
coherent midpoint for the model to interpolate through.

**Fix tried:**
- Pick one. Always the right answer for opposed styles.
- If the *intent* was a specific blended look (e.g. "cel character
  on watercolour background"), don't blend via LoRA stack — use
  **regional conditioning** (a mask per LoRA, see below) or generate
  in two passes and composite.

### Case 3 — Accent LoRA seemingly does nothing

**Symptom:** add "detail-accent" LoRA at 0.4; output looks identical
to without it.

**Diagnosis:** three possibilities, check in order.

1. Trigger missing — add it.
2. Base model mismatch — it was trained on SDXL 1.0, you're on SDXL
   Turbo. Subtle, but enough that the adapter misses its weight
   targets.
3. The LoRA is genuinely weak and needs 1.0+ to show up.

**Fix:** add trigger, bump to 1.0, then 1.3. If still nothing,
replace the LoRA — it's not well-trained.

## Regional (spatial) multi-LoRA — full example

This is SDXL/Flux-era; LTX-2.3 support for regional conditioning is
immature at the time of writing. The pattern, via a regional
conditioning node pack:

### Setup

Goal: character on left half, painted background on right half, each
gets its own LoRA stack.

### Node graph

```text
                 [Base Checkpoint]
                        |
             LoraLoader(char_anchor)  strength 0.8
                        |
             LoraLoader(bg_anchor)    strength 0.7
                        |
                        +----> model out
                        |
   [Mask: left-half]    [Mask: right-half]
          |                    |
   ConditionSetArea (char prompt + char LoRA trigger, mask: left)
          |
   ConditionSetArea (bg prompt + bg LoRA trigger, mask: right)
          |
   ConditioningCombine -----> Sampler
```

The LoRAs are loaded onto the UNet globally, but each region gets a
different *prompt* and therefore only activates the relevant LoRA's
concepts via the trigger words in that region.

### Gotchas

- The characters at the mask boundary can look sliced. Feather the
  mask edges by 40–80 pixels.
- Both LoRAs still apply globally, so expect some bleed — the left-
  half background will partially inherit the "background" LoRA's
  bias. This is usually fine and aesthetically unifying.
- Don't put more than 2–3 regions. Each region multiplies the prompt
  complexity for the model.

## Time-scoped LoRAs — concrete pattern

At the video stage, some community node packs support per-frame
strength scheduling. The pattern for "style fades in over 2 seconds":

```text
Frame 0       -> LoRA strength 0.0
Frame 24      -> LoRA strength 0.4
Frame 48      -> LoRA strength 0.8  (arrival)
Frame 48+     -> LoRA strength 0.8  (hold)
```

Use cases:

- **Style reveal.** A dream-sequence transition where reality is
  photoreal and dreams are watercolour.
- **Transformation sequences.** A character morphing style as they
  power up.
- **Lens/camera effects tied to story beats.** Ramp in a "distortion"
  LoRA as a character loses grip on reality.

Experimental territory on LTX-2.3. If unsupported natively, substitute
the hack: generate two clips (A: no LoRA, B: LoRA at 0.8) and
crossfade in the editor. Cheaper, 80% as good.

## Power-user patterns with rgthree-comfy

`rgthree-comfy` adds quality-of-life nodes that make multi-LoRA
graphs maintainable. Core patterns:

### Pattern 1 — Mute/bypass per-LoRA

Replace your chain of `LoraLoader` with `rgthree Power Lora Loader`
(or similar). Each LoRA gets an on/off toggle and a strength slider
inside the node. A/B testing becomes a click instead of rewiring.

### Pattern 2 — Context bus

Wire `Context` nodes to carry `model`, `clip`, `vae`, `positive`,
`negative`, `latent` through your graph on a single bus line. Your
canvas stops looking like spaghetti. Downstream nodes tap the bus
instead of receiving five individual wires.

### Pattern 3 — Reroutes and groups

Color-code groups: "Loaders" (checkpoint + LoRAs), "Conditioning"
(prompts), "Sampling" (sampler + VAE decode), "Output" (save +
video combine). Collapse groups you're not editing.

### Pattern 4 — Seed nodes

Dedicated `Seed` nodes feed the sampler. You can lock/unlock a
single seed node across the whole graph, and it'll display the seed
in the node title. Essential when chaining shots with shared seeds.

## Exercise A — Build the stack

(As before, unchanged.) Pick your track. On the T2I side:

1. Test each LoRA individually at strength 0.8 with a neutral prompt.
   Record what it does.
2. Stack them in "anchor + accent" order. Start with the anchor at
   0.8, accents at 0.4.
3. Generate 9 images with your project prompt. Evaluate.
4. Adjust strengths. Re-generate 9. Lock in when satisfied.
5. Save this as a T2I workflow `05-keystill-track[A|B|C].json`.

On the LTX-2.3 side: load the T2I output as I2V input, run your keyframe
workflow from module 06. If an LTX-2.3 style LoRA is available and
compatible, add it at 0.8 and compare.

## Exercise B — The 3x3 strength grid

Pick two LoRAs that plausibly go together (e.g. style anchor + detail
accent). Generate a 3x3 image grid:

```text
                 Accent 0.2    Accent 0.4    Accent 0.6
Anchor 0.5         [A5/a2]       [A5/a4]       [A5/a6]
Anchor 0.7         [A7/a2]       [A7/a4]       [A7/a6]
Anchor 0.9         [A9/a2]       [A9/a4]       [A9/a6]
```

All other settings identical (seed, prompt, steps). Pin the grid on
your wall / desktop. Your eye will pick out the sweet spot
immediately. This takes 9 generations — 15 minutes on a decent card
— and will save you hours of random fiddling.

Repeat for every new LoRA pair you commit to. Stack cards in your
`stack_cards/` folder, with the grid thumbnail next to them.

## Exercise C — Conflict simulation

Deliberately pick two LoRAs that disagree (e.g. a soft watercolour +
a hard cel-shaded style). Stack them at 0.7/0.7. Generate. Observe
the specific mush.

Then apply the conflict-resolution playbook:

1. Drop the "less-important-for-this-shot" one to 0.3.
2. If still muddy, set it to bypass and regenerate with just one.
3. Optionally, generate two outputs with each LoRA alone and
   composite the regions you want from each.

The point of this exercise is to build a felt-sense of "LoRAs
fighting". Once you can see it, you stop generating 40 images
chasing a problem that's really a stack conflict.

## What to save before moving on

- One T2I workflow per track you intend to explore.
- A "stack card" for each: which LoRAs, what strengths, what triggers,
  one representative output, and the 3x3 grid thumbnail.
- Notes on which LoRAs *didn't* make the cut and why. This is
  expensive-to-learn intuition; write it down.
- A "budget log" — total LoRA spend you used on the keepers. Your
  future stacks will start from these numbers.

Continue to [09-advanced-workflow-patterns.md](09-advanced-workflow-patterns.md).
