# 07 — LoRA Fundamentals for Video

## What a LoRA is, briefly

A LoRA (Low-Rank Adaptation) is a small set of additional weights that
slot into an existing model to bias it toward some subject, style, or
motion. Concretely: a few tens of MB of numbers that, when combined
with the base model at inference, say "please lean this way".

Two things to internalise:

- **A LoRA is its training data.** The visual, stylistic, and motion
  biases of the training images/videos are what you get. A SpiderMan
  LoRA trained on 1994 cels will not give you Raimi-era SpiderMan.
- **A LoRA is a *nudge*, not a transform.** It biases probabilities; it
  does not guarantee outputs. Strength, prompt, and base model all
  fight with the LoRA. You are a conductor, not a button-pusher.

## LoRAs for video — what is actually compatible

Critical: LoRAs are **model-specific**. A LoRA trained on SDXL does not
load into LTX-2.3. For the animation track, you need:

- **Style LoRAs trained for LTX-2.3.** Fewest in the wild because the
  model is new. Check the Lightricks / civitai / HF pages filtered
  specifically to "LTX-2" / "LTX-2.3" — LoRAs labelled only "LTXV"
  or "LTX Video" usually target the older 0.9.x line and **will not
  load into LTX-2.3**.
- **Character LoRAs trained for LTX-2.3.** Even rarer. For named
  characters you will often train your own, or combine T2I character
  LoRAs (on SDXL/Flux) for the *key still images* with LTX-2.3 motion.
- **Motion LoRAs** (trained to bias the motion distribution — camera
  moves, action types). Emerging category.

If a LoRA's model card does not explicitly say **LTX-2** (v2.x / 2.3)
it will almost certainly not work in LTX-2.3. Do not waste time trying.

### Practical two-stage pipeline

Because LTX-2.3 LoRAs are limited, the common pattern is:

```
   [T2I stage]   Flux/SDXL + style LoRA + character LoRA  --> key still
                                                              |
   [Video stage] LTX-2.3 + (optional LTX-2.3 LoRAs) + I2V  <--------+
```

The still image carries the *look*. LTX-2.3 carries the *motion*.

This is why module 06's keyframe workflow is so important: it is how
you get "LoRA-quality look" from LTX-2.3 even when no LTX-2.3 LoRAs exist
for your target style.

## The LoRA loader node

In ComfyUI, a LoRA is attached with a `LoraLoader` (or an LTX-2.3-specific
variant) placed between the checkpoint loader and the sampler. It takes:

- `model` in, `model` out
- `clip` in, `clip` out on image models; for LTX-2.3 this connects to
  the **Gemma encoder** output instead, under whatever pin name the
  node pack uses (often just labelled as a text-conditioning input)
- `strength_model` — how much the LoRA influences the denoising
- `strength_clip` — how much it influences text conditioning

```
   Checkpoint -> LoraLoader(style) -> LoraLoader(character) -> Sampler
        |            |                     |
        +-- clip ----+---------------------+-----> text encode ...
```

Stacking is just more loaders in series.

## Strengths — how to think about them

- **1.0** — the LoRA as the author intended.
- **0.7–0.9** — "presence" — style reads strongly but base model
  personality survives.
- **0.4–0.6** — "flavour" — hint of the style, doesn't dominate.
- **0.2–0.3** — barely there, often below noise.
- **>1.0** — "overcrank" — frequently over-saturated, over-fit, bad.
  Sometimes needed for subtle LoRAs; try 1.1–1.3 before giving up.

**Start at 0.8** for any new LoRA. Move from there based on output.

### Model vs CLIP strength

You can set them differently:

- Higher `strength_model`, lower `strength_clip` — look is applied but
  prompt understanding is less warped. Useful when a style LoRA
  aggressively re-interprets words (e.g. "a man" always becoming a
  specific trained character).
- Equal — default, usually fine.
- Higher `strength_clip`, lower `strength_model` — rare; mostly used
  for trigger-word-only LoRAs.

## Trigger words

Many LoRAs are trained with specific trigger tokens. The model card
will list them. Without the trigger, the LoRA's effect is diluted.

Typical triggers:

- Style LoRA: a tag like `<style-90s-anime>` or a specific word like
  `madhousestyle`.
- Character LoRA: the character's training name, often with an
  underscored ID like `mary_ltv`.

**Put triggers at the front of the positive prompt.** They get more
weight there. Comma-separate them.

## Training data bias — the invisible trap

Example: a "90s anime" style LoRA trained mostly on nighttime urban
scenes will produce... nighttime urban scenes, regardless of your
prompt saying "sunlit beach". You can fight it, but you are fighting
the training distribution.

How to notice:

- Generate 10 outputs with the LoRA and a *neutral* prompt (like
  "landscape"). What do you get? That is the LoRA's center of mass.
- Generate 10 outputs *without* the LoRA using your project prompt.
  Compare.
- If the LoRA's center of mass is far from your project, either the
  LoRA is wrong for the project or you need to prompt explicitly
  against its bias (and accept weaker effect).

## LoRA quality checklist

Before committing to a LoRA for a full project:

- [ ] Is its base model correct (LTX-2.3 for video, Flux/SDXL for the T2I
      stage)?
- [ ] Does the model card include sample images/clips that look like
      what you want?
- [ ] Does it have documented trigger words?
- [ ] Do you get something usable at strength 0.8 with triggers, on a
      prompt matching a sample from its card?
- [ ] Can you still prompt around it (not all outputs identical)?

If any of these fail, move on. There are more LoRAs than time.

## Sourcing LoRAs

Main public sources:

- **Civitai** — largest collection, variable quality, explicit
  model-compatibility labels. Filter by base model.
- **Hugging Face** — fewer but often higher-quality research-grade
  releases. Lightricks posts official LoRAs here.
- **Individual artist Patreons / Discords** — best-quality custom
  work, paid.
- **Roll your own** — feasible for character LoRAs with 20–50 curated
  stills. Out of scope for this course.

## LoRA internals in plain language

You don't need to read the paper to work effectively with LoRAs, but a
few concepts underneath show up as knobs in the UI and behaviour in
the output. Skip this section on a first pass if you want; come back
when something weird happens.

### The low-rank trick

A diffusion model's attention and feedforward layers are big matrices
(hundreds of millions of numbers). Fine-tuning the full model means
editing all of them. A LoRA instead learns two much smaller matrices
`A` and `B` such that the update to the big weight `W` is
approximately `B @ A`. The *rank* is the inner dimension — how many
"directions" in weight-space the LoRA is allowed to push along.

Practical consequences:

- **Low-rank LoRAs (rank 4–16)** are tiny (10–50 MB), fast to load,
  cheap to train. They capture simple biases (a style, a small
  concept) well. They struggle with detailed characters or complex
  multi-concept adaptations.
- **Mid-rank (rank 32–64)** — a common sweet spot. 100–300 MB. Handle
  detailed characters, nuanced styles, small collections of concepts.
- **High-rank (rank 128+)** — fewer constraints, more capacity,
  starts to approach a small fine-tune. Bigger files, harder to
  train without overfitting.

You do not set rank at inference; it's fixed at training time. But
when you see a LoRA's file size, you can guess its ambition: a 20 MB
LoRA will not reliably hold a character's face in ten different
outfits.

### Alpha and effective strength

LoRAs ship with an `alpha` value set at training time. Inference
scaling is effectively `alpha / rank * strength_you_set`. Most tools
handle the alpha/rank ratio automatically, so the `strength` slider
you see is what matters. But two LoRAs with different alpha/rank
ratios can have very different "feel" at the same nominal strength —
which is why "strength 0.8" is a rule of thumb, not a law.

When a LoRA seems to do nothing at 1.0, try 1.5 and 2.0 before giving
up. When a LoRA blows up the output at 0.8, try 0.4 and 0.2. Trust
the output, not the slider label.

### Why LoRAs affect style *and* composition

A style LoRA trained only on paintings of lone figures against sky
will bias not just the brushwork but the composition — you will get
more lone figures against sky. This is not a bug; the LoRA learned
co-occurrences. It is why the "training data bias" section above
matters: the LoRA imports the aesthetic *gestalt* of its training
set, not a clean isolated style signal.

Mitigation: combine the LoRA with prompts that push against its
bias (different setting, different composition) and accept that the
effect will weaken. Or, for very biased LoRAs, use them only at low
strength as an accent.

## Reading a model card like a professional

When evaluating a LoRA on Civitai / HF / elsewhere, read the card
like a used-car listing. The format varies; the information you need
is consistent:

| Field | Why it matters | Red flag |
|---|---|---|
| Base model | Must match your stage exactly (LTX-2.3 2.3, SDXL 1.0, Flux.1 dev, SD 1.5) | Vague ("works with most models") — usually means SD1.5 and nothing else |
| Rank | Sanity-checks ambition vs. file size | Huge file, tiny scope described (overtrained) |
| Trigger words | Without them the LoRA is 30% itself | None listed — you're guessing |
| Training images (number, description) | Predicts bias and coverage | "10 images, all one character pose" → will only do that pose |
| Recommended strength | Author knows the LoRA; honour their starting point | Wildly high (1.5+) suggested → author is compensating for weak training |
| Sample outputs | The truth teller | No samples, or samples using other LoRAs / heavy inpainting |
| Version history | v1 vs v3 usually differ | Only one version, released yesterday, no feedback yet |
| Licence | Matters for any publishing | "No commercial" / "No AI output use" → avoid if you plan to share work |

### Sample-image forensics

When you look at sample outputs, ask:

1. Is the resolution natural for the base model, or suspiciously large?
   (Heavy post-upscale hides weakness.)
2. Are the samples all the same pose? (LoRA is pose-collapsed.)
3. Is the style consistent across samples, or does it fluctuate
   between "strong" and "barely visible"? (Unstable training.)
4. Are hands and faces handled in-frame, or always cropped out?
   (Author is hiding weaknesses.)
5. Are samples specifically animation-friendly if you need video?
   Still-image samples don't tell you about temporal stability.

## File formats and provenance

Common formats you will encounter:

- **.safetensors** — preferred. No arbitrary code execution risk on
  load.
- **.ckpt / .pt** — older PyTorch pickle format. Avoid unless you
  fully trust the source. Malicious pickles are a real attack vector.
- **.bin** — Hugging Face-style weight files. Usually safe but check
  context.

For animation LoRAs specifically, watch out for packaged zips that
contain multiple files (the LoRA, a "LoRA helper", sample workflows).
Unzip into a scratch folder, audit what's there, then move only the
`.safetensors` into your `models/loras/` tree. Don't execute bundled
scripts.

### Compatibility archaeology

You downloaded a LoRA and it does nothing or crashes the sampler.
Checklist:

1. **Base model mismatch.** The #1 cause. Open the LoRA in a text-
   capable tool (Kohya's utilities, ComfyUI's metadata viewer, or a
   small Python script with `safetensors.safe_open`). Look at the
   metadata dictionary. Fields like `ss_sd_model_name`,
   `ss_base_model_version`, `ss_v2` tell you what it was trained
   against.
2. **Missing text encoder conditioning.** Some LoRAs only modify the
   UNet/DiT side. A LoRA loader that forces CLIP strength > 0 on such
   a LoRA can error or no-op. Try `strength_clip = 0`.
3. **Different precision.** A fp16 LoRA merging into a fp8 model
   sometimes has numeric instabilities. Convert or pin precision.
4. **The file is corrupt.** Download was incomplete. Re-download,
   check size against the source page.
5. **Node pack outdated.** Your LoraLoader variant may not yet
   support a newer LoRA format (e.g. LoHA, LyCORIS). Update the
   custom node pack.

### Family tree of LoRA-adjacent formats

You will see these names. All are "LoRA-like" — same mental model,
different math:

- **LoRA** — the baseline. What 95% of your downloads will be.
- **LyCORIS** — umbrella for LoHA, LoKR, etc. More expressive forms
  of low-rank adaptation. Load with LyCORIS-compatible nodes.
- **IA3** — even smaller adapters that scale activations. Rare for
  style/character work.
- **DoRA** — magnitude-decomposed LoRA. Recent; usually loads via
  the standard LoRA loader on updated node packs.
- **Hypernetworks** — SD 1.5-era, largely superseded. Ignore.
- **Embeddings / Textual Inversion** — not a LoRA at all; they add
  new tokens to the text encoder. Listed here because they get
  conflated in model zoos. Complementary, not competing.

## Training your own character LoRA — bird's-eye

You will hit a wall where no existing LoRA matches your needed
character. Rolling your own is reasonable once you have the pipeline.
This is a sketch, not a tutorial — actual training deserves its own
course.

### Data preparation

- 20–50 images of the character is enough for a decent v1. Quality
  over quantity.
- **Variety on axes you want generalisation**: pose, angle, lighting,
  expression. Keep the character features constant across variety.
- **Consistency on identity axes**: if your character has "silver
  streak in black hair", every training image must show hair (no
  hats cropping it out) and the streak must always be in the same
  place.
- 512×512, 768×768, or 1024×1024 depending on target base model.
  Crop to subject; minimal background clutter.

### Captions

Each image gets a text caption. Two schools:

1. **Single trigger + describe everything else.** Example:
   `mychar_v1, 1girl, black hair with silver streak, red jacket,
   outdoor, sunny, smiling`. This teaches the LoRA that the trigger =
   the fixed part that's *not* in the captions.
2. **Trigger + minimal captions.** Example: `mychar_v1, smiling,
   outdoor`. Simpler, but the LoRA bakes in more (hair, jacket
   become "part of the character" even if they shouldn't be).

For flexible characters (should work in any outfit), use school 1.
For iconic characters (always same outfit), school 2.

### Training parameters starter

For SDXL character LoRAs (concrete numbers, adjust empirically):

- Kohya_ss GUI or sd-scripts as the trainer.
- Network dim (rank) 32, network alpha 16.
- Learning rate 1e-4 (UNet), 5e-5 (text encoder) — or text encoder
  off if you only want visual changes.
- 10–20 repeats per image, 10–20 epochs. Total step count ~1500–4000.
- Optimiser: AdamW8bit.
- Resolution matches data.
- Save every epoch. You will pick the middle ones, not the last one
  — overtraining is a bigger enemy than undertraining.

For LTX-2.3 LoRAs specifically, training is newer and the tooling is
less settled. Check the Lightricks repo's training scripts for the
current best practice; community consensus moves quarterly.

### Picking the "right" epoch

Generate the same prompt with every saved epoch. You will see:

- Early epochs: character barely visible, style unstable.
- Middle epochs: character reads, still flexible with prompt.
- Late epochs: character very strong, output "copies" training poses,
  prompt flexibility collapses.

The right epoch is the last one where prompt flexibility survives.

### Common mistakes

- **Training on 5 nearly identical images** — LoRA memorises those 5,
  does nothing else.
- **Heavy watermarks / signatures in data** — LoRA learns to generate
  watermarks.
- **Mixed styles in training data** (some photos, some anime) — LoRA
  averages into mush.
- **Text in training captions that also appears in other LoRAs you'll
  stack with** — trigger collisions.

## Trigger words, deeper

A trigger is a text token (or short phrase) that the LoRA was
conditioned on during training. Behaviour notes:

- **Placement matters.** Tokens near the front of the prompt have
  more weight in most text encoders. Put your primary trigger in the
  first 8 tokens.
- **Comma-separate.** `triggername, rest of prompt` is cleaner than
  `triggername rest of prompt`, which can merge tokens.
- **Multiple triggers per LoRA.** Some LoRAs have a primary trigger
  plus outfit/variant triggers. `charname, outfit_casual, smiling` —
  consult the card.
- **Weighting triggers.** You can weight a trigger with prompt
  syntax, e.g. `(triggername:1.2)`. Use this when the trigger seems
  weak but you don't want to raise the whole LoRA strength.
- **Trigger collisions.** Two LoRAs with overlapping trigger words
  will interfere. Rename via prompt-level workarounds: only use
  Style LoRA A's trigger in positive, suppress LoRA B's in negative.
  (Hacky; better to avoid LoRAs with colliding triggers.)

### When a LoRA has no documented trigger

Sometimes the card forgets or the author trained without a unique
trigger. Recovery:

1. Try the character/subject name.
2. Try the style name (e.g. "madhousestyle").
3. Read the training caption convention if disclosed.
4. Try the LoRA at strength 1.2–1.5; trigger-less LoRAs need more
   push.
5. Move on. A trigger-less LoRA is usually an amateurish release.

## Exercise — Audit one style LoRA

Pick one style LoRA relevant to your track (90s anime, 90s Saturday
morning cartoon, etc.). For the T2I stage (not LTX-2.3):

1. Generate 6 outputs at strength 0.0, 0.4, 0.6, 0.8, 1.0, 1.2 with
   the same prompt, seed, and everything else.
2. Lay them side by side. Where does the style arrive? Where does it
   overcook?
3. Pick your operating strength for this LoRA and note it.
4. Now change the prompt (different subject, same style tag). Does the
   style survive? That tells you how robust the LoRA is.

You will do this audit many times over a career. Speed comes with
reps.

Continue to [08-multi-lora-workflows.md](08-multi-lora-workflows.md).
