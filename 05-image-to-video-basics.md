# 05 — Image-to-Video Basics

T2V is fun. I2V is where you get *control*. For a stylised short, almost
every shot will start from an image you designed — either drawn, or
generated with a text-to-image model and curated.

## Why I2V

- **Identity lock.** A named character looks the same shot-to-shot only
  if you pin it with an image. Prompts alone drift.
- **Composition lock.** You storyboarded the shot; I2V lets the
  storyboard frame actually be the first frame.
- **Style lock.** Image models (Flux, SDXL, SD1.5) have a mature style
  LoRA ecosystem. Generate the key image with your style LoRA, then
  animate with LTX-2.3 I2V.

## The minimal I2V graph (LTX-2.3)

```text
   LTX-2.3 Checkpoint Loader
        |
        +-- diffusion model --------------------------------------+
                                                                  |
   Gemma Encoder Loader (text + audio)                            |
        |                                                         |
   Gemma Encode (positive)  Gemma Encode (negative)               |
        |                         |                               |
        +--- positive --+-- negative -----------------------------+
                        |                                         |
   LoadImage (start frame) -> LTX-2.3 Video VAE Encode -> image latent
                                                                  |
                        +---- image latent -----------------------+
                        |                                         |
                        v                                         |
                   LTX-2.3 Sampler (I2V mode)  <-- empty video + audio latent
                        |
                        +---- split ----+
                        |               |
                   Video VAE Decode  Audio VAE Decode
                        |               |
                        +--- VHS_VideoCombine (video + synced audio)
                                        |
                                  output.mp4
```

Key LTX-2.3 differences from an LTXV-0.9 I2V graph:

- Text conditioning is **Gemma Encode**, not `CLIPTextEncode` / T5.
- The start image is encoded by the **LTX-2.3 video VAE** (the same
  VAE used for output decode). Mixing in an unrelated VAE will
  misalign the latent and your first frame will drift immediately.
- The sampler still emits a **joint video+audio latent**. Audio is
  generated from scratch in I2V (there's no audio input from the
  image); the model invents a soundscape that matches the prompt +
  image semantics.

The key node is the LTX-2.3 sampler / conditioning node configured
for image conditioning. In the official LTX-2.3 node pack this is
typically a variant sampler or an extra conditioning node you wire
the encoded image latent into. Refer to the **current LTX-2.3 example
`image_to_video` workflow** for exact node and pin names.

### Starting settings

- Input image resolution: match your target video resolution **exactly**.
  Pre-resize in an image editor, or add an `ImageScale` / `ImageResize`
  node before the VAE encode. Mismatched resolutions cause the first
  frame to be re-encoded and drift.
- Frames, sampler, steps, CFG: use the official example defaults.
  Course-level starting points: CFG ~3.0–3.5, 20–30 steps, ~4 s clips
  at native fps.
- **Denoise / strength of conditioning:** exposed by the LTX-2.3 I2V
  sampler under a name like "conditioning strength", "image influence",
  or implicit in the sampler. Start near the default (often 1.0 for
  strict I2V) and lower it only if the first frame looks frozen for
  too long before motion begins.
- **Audio CFG (if exposed):** a separate guidance scale for the audio
  channel. Keep near the default (3.0–4.0 typical). Raise if audio
  feels generic, lower if it feels over-dramatic or unrelated to the
  picture.

## Exercise 1 — Your first I2V

1. In any image tool, create a 768×512 image of a character you designed
   (use Flux/SDXL with a style of your choice). Save as `char.png`.
2. Load it into the I2V workflow. Prompt:
   ```
   [same character description as the image], slowly turns their head
   to look at the camera, eyes focus, subtle breathing motion, soft
   natural lighting, cinematic.
   ```
3. Generate. Expected: first frame matches your image closely, then
   subtle head turn and eye focus over ~4 seconds.

If the face morphs identity across the clip, either:

- The prompt is underspecified about the character. Add concrete
  details (hair colour, eye colour, clothing).
- The conditioning is too weak. Check the sampler's I2V-strength input.
- The character is not in the model's prior. You need a **character LoRA**
  (module 08).

## Preparing input images well

Good I2V inputs share a few traits:

- **Native resolution.** See above.
- **Clean background separation.** Busy backgrounds on the first frame
  sometimes bleed into subject motion. Not a rule, but be aware.
- **The character is doing the starting pose of the action.** If the
  prompt says "the warrior draws her sword", the input image should
  show the hand on the hilt, not already mid-swing — otherwise the
  model has no room to animate the draw.
- **Lighting is plausibly temporal.** Very flat "album cover" lighting
  with no direction makes motion look unphysical. Directional light is
  easier for the model to propagate.

## Ending frames (where supported)

LTX-2.3 supports end-frame conditioning in several workflow patterns:
you provide a first and last image, and the model interpolates the
motion between them. This is powerful for deliberate beat-to-beat
animation.

The graph is the I2V graph plus a second `LoadImage` wired into the
sampler's end-frame input:

```text
   start_img -> LTX-2.3 Video VAE Encode -+-> LTX-2.3 Sampler
                                          |
   end_img   -> LTX-2.3 Video VAE Encode -+
```

Exact node name varies — look for an LTX-2.3 sampler variant labelled
"keyframe", "first/last", or with both `latent_start` and `latent_end`
inputs.

Things to know:

- **Both images must share style** or the interpolation will look like a
  morph between two different paintings. Generate both with the same
  style LoRA and similar prompts.
- **Both images must share rough composition** (subject in roughly the
  same part of frame, similar framing). The model interpolates, it does
  not magically cut between unrelated shots.
- This is covered in depth in module 06.

## I2V failure modes

| Symptom | Fix |
|---------|-----|
| First frame is your image, then it "wakes up" and drifts | Raise conditioning strength, add identity details to prompt |
| First frame is slightly off (colour shifted, detail lost) | Resolution mismatch, or wrong VAE — must be the LTX-2.3 video VAE, not a foreign one |
| Motion is too subtle, looks like a still with wobble | Prompt has no strong motion verbs, CFG too low, or length too short |
| Background stays, subject teleports | You asked for motion the input pose cannot start — redesign the pose |
| Audio is present but bizarre (laughter on a silent landscape, music during a tense moment) | Prompt has no audio clause — Gemma invented one. Add an explicit audio description or "no music, only ambient wind" |
| Output has no audio at all | `VideoCombine` node not receiving the audio pin; or audio VAE missing. Compare to the LTX-2.3 example I2V workflow |

## Exercise 2 — Shot pair

Pick a beat: e.g. character looks down, then looks up.

1. Generate two images with the same seed and style LoRA, prompting only
   the pose difference (one looking down, one looking up).
2. Use them as start and end in the first/last frame workflow.
3. Prompt the motion ("slowly raises her head, hair shifts, eyes meet
   camera").
4. Compare to a T2V version of the same motion (no input images).

The I2V version will be on-model and repeatable. The T2V version will
be prettier sometimes but not a reliable shot in a sequence.

## Exercise 3 — I2V with audio intent

Reuse the Exercise 1 character image. Generate three takes of the
same subtle head-turn motion, changing only the audio clause:

1. `..., audio: soft ambient room tone, distant clock ticking, no
   music.`
2. `..., audio: gentle piano motif fading in, warm low register,
   no sound effects.`
3. `..., audio: quiet intake of breath, faint cloth rustle, a single
   muted heartbeat.`

The picture should look nearly identical across all three. The
emotional register changes completely. This is a tool you have now
and did not have with LTXV 0.9.x — use it in the final project to
tint the mood of otherwise repeatable shots.

## What to save before moving on

- `02-i2v-basic.json` — first-frame only
- `03-i2v-firstlast.json` — first + last frame
- A small gallery of 4–6 I2V outputs with the source images next to
  them. You will reuse the good ones as storyboard beats.
- For each gallery entry: the prompt (with audio clause), seed, and a
  one-sentence note on how the audio shaped the reading.

Continue to [06-keyframes-techniques.md](06-keyframes-techniques.md).
