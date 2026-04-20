# 04 — Text-to-Video Basics

Goal of this module: build the simplest possible working T2V workflow
for **LTX-2.3**, then learn how to write prompts that steer it —
including its audio dimension.

## The minimal T2V graph (LTX-2.3)

```text
   LTX-2.3 Checkpoint Loader
        |
        +-- diffusion model ------------------------+
                                                    |
   Gemma Encoder Loader (text + audio)              |
        |                                           |
        +-- gemma_model ----+                       |
                            |                       |
   Gemma Encode (positive) Gemma Encode (negative)  |
        |                   |                       |
        +---- positive -----+---- negative ---------+
                                    |
   Empty Video Latent (w, h, frames) + Empty Audio Latent
                                    |
                            LTX-2.3 Sampler
                                    |
                            +-------+-------+
                            |               |
                     LTX-2.3 Video VAE   LTX-2.3 Audio VAE
                            |               |
                       Video frames      Waveform
                            |               |
                            +-------+-------+
                                    |
                         VHS_VideoCombine (video + audio)
                                    |
                                  output.mp4
```

Key differences from an LTXV-0.9 graph:

- **No `CLIPLoader` / T5**. The text side goes through a **Gemma
  encoder** node.
- **Two VAE decode paths** — video and audio — joining at the final
  `VideoCombine` node.
- Sampler outputs a **joint latent** that splits into video + audio
  downstream.

Node names in your actual install will differ — use the official
**LTX-2.3 example workflow** shipped with the node pack as ground
truth, and match the *wiring pattern* above. If you are looking at an
LTXV-0.9 tutorial and see `CLIPLoader` + T5, that graph will not run
on LTX-2.3; use it only for conceptual reference.

### Suggested starting settings

Use whatever the official LTX-2.3 example workflow ships as defaults —
they are tuned for the model. If you want course-specific starting
points to play with:

- Resolution: **768×512** (or whatever the example uses — LTX-2.3 is
  generally happier at higher resolutions than 0.9.x)
- Frames: follow the example's valid length — round to model-aligned
  lengths rather than arbitrary numbers. ~4 s at native fps is the
  default clip duration for exercises here.
- Frame rate: use the model's native fps (typically 24 or 25; confirm
  in the example workflow).
- Sampler + steps + CFG: use the example defaults. Starting place is
  typically CFG ~3.5 and 20–30 steps. Do not chase sampler micro-tuning
  before you've spent 20 prompts at the defaults.
- Audio: leave it on while learning. You'll listen for prompt/picture
  mismatch as a diagnostic.
- Seed: fix it to something memorable while iterating (e.g. `42`), then
  randomise later.

Save this as `01-t2v-basic.json`. It is the skeleton you will clone for
every other workflow in this course.

## Prompt anatomy for LTX-2.3

Because LTX-2.3 uses a **Gemma** encoder, you can write more natural
prose than you could with T5-era models. Full sentences, instructional
phrasing, even multi-paragraph descriptions parse well. The trade-off:
more room to contradict yourself. Clarity > brevity, but coherence >
length.

A good LTX-2.3 T2V prompt has five parts. The fifth — audio — is new
and optional, but worth thinking about.

1. **Subject & action.** *Who* is doing *what*.
   "A young woman in a red raincoat walks through a rain-soaked neon
   alley, one hand holding a folded paper umbrella."
2. **Camera.** Shot size, angle, movement.
   "Medium tracking shot, camera follows from the side at eye level,
   slight handheld sway."
3. **Style / medium.** Artistic register.
   "90s anime cel animation, thick black ink outlines, limited palette,
   hand-painted backgrounds with visible brushwork."
4. **Atmosphere.** Light, weather, mood, time of day.
   "Night, heavy rain, puddles catching neon signs, steam rising from
   grates, distant traffic out of frame."
5. **Audio (optional).** Diegetic sounds, ambient bed, music mood.
   "Ambient rain patter on cloth umbrella, distant traffic hum, quiet
   footsteps on wet pavement, no music."

Glue them together as natural sentences or comma-separated clauses.
Gemma parses both. Aim for one tight paragraph — the model tracks
structure but loses focus past a dense 150 words.

### The audio dimension in practice

You do not need audio words in every prompt; silence/ambient is a
reasonable default. But when audio matters:

- **Name the source.** "Footsteps on gravel", "wind through pine",
  "crackling fire" beats "nature sounds".
- **Set loudness/distance.** "Distant thunder", "close breath",
  "muffled crowd behind glass" steer the model's mix.
- **Say 'no music' explicitly** when you want diegetic audio only.
  Otherwise LTX-2.3 sometimes invents a score.
- **Music mood tags, not genre labels.** "Tense low strings",
  "uplifting piano motif", "lone acoustic guitar" work better than
  "cinematic music" or a genre name, which the model reads too
  literally.

### Motion verbs matter

The difference between a static postcard and an animation is in the verbs:

- **Strong motion verbs:** walks, runs, spins, swings, crashes, flares,
  drips, erupts, unfurls.
- **Camera verbs:** pans, tracks, dolly in, pulls back, tilts up,
  crash-zoom, whip-pan.
- **Weak / ambiguous:** is, stands, exists, beautiful, amazing.

If your output is static, count the verbs in your prompt. If there are
zero motion verbs, the model has nothing to animate.

### Negative prompt — keep it specific

Video-appropriate negative examples:

- `static, still, frozen, slideshow`  — fights "motionless" outputs
- `flicker, strobing, jitter`         — fights temporal artifacts
- `watermark, text, subtitles, logo`  — fights training-data leakage
- `deformed hands, extra fingers`     — general anatomy guard

Do not dump 50 negative tokens. Five to ten, specific, is better than
forty generic.

## Exercise 1 — Hello, video

Prompt:

```
A golden retriever runs across a grassy meadow toward the camera,
tail wagging, afternoon sunlight, shallow depth of field, photoreal,
handheld camera tracking the dog, 4k cinematic.
```

Negative:

```
static, still, slideshow, watermark, text, deformed
```

Settings as above. Generate, save, take notes. Expected: ~4s clip of a
dog running, with believable fur motion and a camera that actually moves.

If the dog is static, re-read the "motion verbs" section and rewrite.

## Exercise 2 — Same shot, different styles

Keep the subject/action/camera. Change only the style+atmosphere. Run
three:

1. `...90s anime cel animation, thick ink outlines, hand-painted
   backgrounds, limited frame animation, saturated colours.`
2. `...1990s american tv animation, cel-shaded, bold black outlines,
   flat shading, comic-book panel style, dynamic pose.`
3. `...stop-motion felt puppet, visible fabric texture, slight jitter
   on 2s, warm tungsten practical lights.`

Compare. The point is to feel how much the style tag shifts output *and*
how much it doesn't — style LoRAs (module 07) are for when prompt words
alone are not enough.

## Exercise 3 — Camera drill

Same subject, run 5 variations changing only camera:

- `static locked-off wide shot`
- `slow dolly in, subject in centre, lens flares`
- `whip pan from left to right, subject enters mid-pan`
- `overhead top-down, subject crosses frame diagonally`
- `crash zoom to extreme close-up on the subject's eyes`

Crash zooms and whip pans are your 90s SpiderMan vocabulary. Practice
invoking them deliberately.

## Exercise 4 — Audio drill (LTX-2.3 only)

Same subject (dog in meadow). Hold the visual prompt constant. Vary
only the audio clause at the end. Run four:

1. `..., ambient audio: no music, only wind through tall grass and
   distant birdsong.`
2. `..., audio: uplifting orchestral swell rising over the clip,
   warm strings and light brass, no sound effects.`
3. `..., audio: close panting breath and heavy paws on wet grass,
   quiet distant traffic, no music.`
4. `..., audio: tense low cello drone, subtle wind, a single distant
   bell toll near the end of the shot.`

Play each back with headphones. You will feel how much the audio
prompt changes the **emotional reading** of an identical image. This
is the single biggest leverage LTX-2.3 gives you over the 0.9.x
generation.

## Prompting tips specific to LTX-2.3

- **Natural sentences are fine** — preferred, even. Gemma parses
  "A woman walks down the street, pauses, and looks up at a neon
  sign flickering above her" cleanly. You don't need to comma-chain
  token fragments.
- **Concrete, grounded descriptions beat poetic mood.** "Red
  raincoat, left hand holding a paper umbrella" outperforms
  "mysterious figure in the rain".
- **One subject is easier than two.** Two named characters in a single
  shot is a hard problem; four is usually a disaster. If you need
  multiple characters, stage them sequentially across shots.
- **Physical materials help.** "Wet asphalt reflecting red neon",
  "dust motes in a shaft of sunlight", "frost on a windshield" all
  produce stronger atmosphere than abstract mood words — and
  automatically produce matching audio ("rain on glass", "distant
  traffic hum").
- **Audio descriptions should match the picture.** If the shot is an
  intimate close-up, audio should be intimate too (breath, cloth
  rustle). If the shot is a wide exterior, audio should be distant
  and ambient. Mismatch produces uncanny output.
- **"No music" is a valid instruction.** LTX-2.3 will otherwise score
  your clip by default, often tastefully, but not always what you want.
- **Dialogue.** LTX-2.3 can produce non-verbal vocal sounds (gasps,
  laughs, cries) and sometimes short lines of audible dialogue, but
  don't count on clean lip-sync in 2025. For dialogue-driven pieces,
  generate picture silent and record dialogue separately.
- **Name the medium.** "35mm film", "Super 8", "cel animation on 2s",
  "CRT scanlines" — all bias the output meaningfully.

## What to save before moving on

- `01-t2v-basic.json` — the skeleton workflow.
- A prompt journal with at least 5 entries covering the drills above.
- A mental catalogue: which prompt words actually moved your outputs,
  which were decorative noise. This catalogue is personal and informs
  every later module.

Continue to [05-image-to-video-basics.md](05-image-to-video-basics.md).
