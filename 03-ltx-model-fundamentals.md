# 03 — LTX Model Fundamentals

You can produce decent output without understanding the model. You cannot
*debug* bad output without it. This module is deliberately conceptual;
there are no workflows to build. Read it once, return to it when things
break.

## The DiT in one paragraph

A Diffusion Transformer predicts noise-to-remove from a noisy latent,
conditioned on text (and optionally image/audio/control signals).
"Latent" means a compressed representation produced by a VAE — for
video, the compression is both spatial (e.g. 8× per side) and temporal
(several frames of video become one latent frame). LTX-2.3 processes a
**joint video+audio latent** as a sequence of space-time (and time-
only, for audio) patches; the transformer can therefore reason about
motion *and* sound coherently, not just per-frame.

Practical implications:

- **The latent space is small** compared to the pixel space. This is
  why LTX-2.3 fits in consumer VRAM at all.
- **Temporal patching means prompt changes steer motion globally**.
  Prompting "the camera slowly zooms in" actually works. Prompting
  "frame 1: wide shot, frame 30: close-up" does not — that is not how
  the model ingests prompts.
- **Audio and video co-denoise.** Because the model processes joint
  latents, the audio reflects the visual action (footsteps when the
  character walks, a whoosh on a whip-pan) and vice versa. You can
  exploit this: "rain on the window" in the prompt produces both the
  rain picture and the rain sound.
- **Latents are lossy**. The VAE decoder can introduce subtle flicker,
  colour drift, or soft detail. A bad VAE destroys output. This is
  why LTX-2.3 ships its own tuned video + audio VAE.

## The Gemma encoder in one paragraph

Text prompts (and, where supported, audio references) pass through a
**Gemma-based multimodal encoder** that emits the conditioning tokens
fed to the DiT cross-attention. Gemma understands long prompts well;
you can write more naturally descriptive paragraphs than with T5 and
the model tracks them. Audio reference inputs (a short wav clip) can
condition tone/mood — used rarely, but available. Practical notes:

- **Prompt length.** Gemma tolerates longer prompts than T5. You can
  write 2–4 sentences of natural prose. Don't abuse this — dense long
  prompts still collapse past a point — but you are not limited to
  75 tokens.
- **Instruction-style phrasing works.** You can write
  "Show a woman in a red coat walking slowly through rain. The camera
   stays at eye level and tracks her from the side." Gemma parses the
  instructional structure; the model follows it.
- **Audio prompt input (if exposed in your node pack)** takes a short
  wav or mp3. Keep it under 10 seconds and thematic (e.g. the kind of
  music you want) — it's a nudge, not a precise driver.

## Resolution, frame count, length

LTX-2.3 has a native operating range. Going outside it degrades quality
fast — not gradually. The safe envelope (verify with the current
LTX-2.3 model card on the day you download):

- **Resolution:** must be a multiple the model expects — typically
  multiples of 32, ideally 64. Common good choices: 768×512, 1024×576,
  512×768 for portrait. LTX-2.3 can reach higher resolutions than the
  0.9.x line, but VRAM cost rises steeply. Extreme aspect ratios
  produce artifacts.
- **Frame count:** the model has a sweet spot; exact valid lengths
  depend on the temporal compression factor of the LTX-2.3 VAE.
  Follow the lengths used in the official example workflow (often a
  `8k+1` or `4k+1` pattern) rather than picking arbitrary numbers.
- **Frame rate:** LTX-2.3 has a native frame rate (commonly 24 or 25
  fps — check the model card). The *output file* can be retimed; the
  *generation* is tied to the model's native rate.
- **Audio sample rate:** the audio VAE operates at a fixed sample rate
  (commonly 44.1 kHz or 48 kHz). You don't pick this; it's baked in.

The `length = frame_count / fps` relationship matters: generating at
24 fps, a clip around 4–5 seconds is a comfortable length for a single
shot.

## CFG / guidance

LTX-2.3 uses classifier-free guidance (CFG) like most diffusion models.
Quirks specific to video:

- **Sweet spot is lower than for images.** CFG 3–5 is typical. CFG 7+
  tends to produce over-saturated, over-moving, flicker-prone output.
- **Negative prompt has real effect.** LTX-2.3 responds to negatives
  via the Gemma encoder; use them sparingly and specifically.
  Boilerplate "ugly, deformed, blurry" lists are low-value.
- The sampler may expose separate **image CFG** and **video CFG** when
  mixing I2V conditioning, and separate **audio CFG** as well. Keep
  them close unless you know why one needs to be different.

## Samplers and steps

- **Euler / DPM++** variants typically work. Use whatever the official
  LTX-2.3 example workflow uses as your default; it's tuned for the
  model.
- **Steps:** 20–30 is typical. LTX-2.3 converges relatively fast. Past
  40 steps you pay compute for marginal quality.
- **Scheduler:** most of the gain comes from matching the scheduler
  the model was trained with (usually exposed as a specific option in
  the LTX-2.3 sampler node). Do not swap schedulers unless you know
  which one is expected.

## Seeds and reproducibility

- The same prompt + seed + all settings will reproduce bit-for-bit on the
  same GPU. Change GPU and you may get small differences due to CUDA
  nondeterminism.
- **Save seeds you like.** Put them in your prompt journal. Seeds for
  good compositions are reusable across related prompts.
- For scene continuity across shots, see the keyframe module — seed
  locking alone is not enough, you need visual conditioning.

## Audio in LTX-2.3 — first-class, not an afterthought

Because audio is co-generated, a few things are true that weren't with
LTXV 0.9.x:

- **The prompt has an audio dimension.** Words like "footsteps echoing
  on wet asphalt", "rain on glass", "distant traffic", "a low hum of
  fluorescent lights" create both picture and sound cues. Cheap way to
  get foley.
- **Mismatched sound is a diagnostic.** If your visual prompt describes
  a silent environment but the audio comes out crashing and chaotic,
  something in the prompt is mis-weighted. Listen; don't just watch.
- **You can ignore the audio.** For a stylised animated short you
  usually score separately. Mute the output; keep the picture.
- **You can keep the audio as temp.** The generated audio is often a
  surprisingly good temp track — use it to cut picture against, then
  replace at the mix stage.
- **Audio guidance at generation.** Some LTX-2.3 sampler variants let
  you attach a reference audio clip (mood, genre, ambience). Useful
  for consistent acoustic tone across shots; keep the clip short.

## Common failure modes and what they mean

| Symptom | Likely cause |
|---------|-------------|
| Subject morphs identity mid-clip | Weak I2V conditioning, prompt too abstract, LoRA strength too low |
| Flicker/strobing on textures | VAE decode issue, resolution not aligned, too few steps |
| Static output despite motion prompt | CFG too low, frame count below model minimum, prompt lacks motion verbs |
| Over-saturated, "burned" look | CFG too high, LoRA weights too high, wrong scheduler |
| First frame perfect, last frame garbage | Resolution/frame-count outside native envelope, or length > model's reliable span |
| Slow-motion look | Frame rate mismatch: you generated at 24 fps and encoded as 30 |
| Hands/faces deform only near the edge | Resolution not a multiple of 64, or upscaled past native |
| Output mp4 has no audio | Audio VAE not loaded, or audio pin not wired in video-combine node |
| Audio is chaotic / doesn't match picture | Mismatched audio/visual prompting, or audio CFG too high/low |
| "CLIPLoader" or "T5" error on queue | You loaded a pre-LTX-2 workflow by mistake — switch to the LTX-2.3 example |

You will see all of these. Use the table instead of random-walk
tweaking.

## Precision (fp16 vs fp8 vs bf16)

- **bf16** — similar memory to fp16 on modern cards; numerically more
  stable. Baseline for 24 GB+ on LTX-2.3.
- **fp16** — equivalent quality to bf16 on most samples; use whichever
  the node pack prefers.
- **fp8** — meaningful VRAM saving, small quality hit on fine detail.
  Worth it on 12–16 GB cards. Recommended default for learning.

Gemma encoder precision is independent of the DiT precision. Gemma
tolerates int8 / fp8 well with minor impact on prompt adherence; load
it low if VRAM is tight.

## What not to worry about yet

- Training your own LoRA. Later.
- Custom schedulers, noise injection tricks, exotic samplers. Later.
- Ultra-long (>10s) single-shot generations. The model will produce them
  but quality drops; for a short film you stitch shots, you do not try
  to generate the whole film in one pass.

Continue to [04-text-to-video-basics.md](04-text-to-video-basics.md).
