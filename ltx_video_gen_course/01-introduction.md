# 01 — Introduction

## What is LTX-2.3?

**LTX-2.3** is Lightricks' 2025 flagship open video model. It is a
**Diffusion Transformer (DiT)** operating in a compressed video latent
space, with two important differences from the earlier LTXV 0.9.x line:

- **Gemma-based multimodal encoder.** Conditioning is produced by a
  Gemma (Google's open LLM) variant that handles **text and audio
  jointly** — not a T5 encoder like older LTXV. In practice this means
  the official ComfyUI workflow has a Gemma-style encoder node where
  older tutorials show `CLIPLoader` + T5.
- **Native synchronised audio.** LTX-2.3 generates audio *alongside*
  the video in a single pass. A generation produces an mp4 with
  picture and sound co-generated, not a silent clip that you score
  later. You can still mute the audio output, or replace it in post,
  but the model treats audio conditioning as first-class.

Compared to earlier open video models (AnimateDiff, SVD, LTXV 0.9.x),
LTX-2.3:

- Treats temporal coherence as a first-class property of the
  architecture.
- Responds meaningfully to prompts about *motion* (camera moves,
  subject action, pacing), not just frame-1 composition.
- Treats sound as part of the output, so prompt structure has an
  audio dimension (diegetic sounds, music mood, dialogue tone).
- Is fast enough that iteration on a single workstation is realistic,
  though the Gemma encoder + audio path raises VRAM requirements over
  LTXV 0.9.x at the same resolution.

> **Heads-up on older material.** Most online LTX tutorials predate
> LTX-2 and describe the T5-based LTXV 0.9.x stack. Those workflows
> **will not load LTX-2.3** — the encoder node is different, the
> conditioning pins differ, and the sampler may expect an audio
> conditioning slot. Always start from the **official LTX-2.3 example
> workflow** shipped by Lightricks in their ComfyUI node pack; adapt
> the *ideas* from this course into that graph rather than rebuilding
> from LTXV-0.9 screenshots.

## Why ComfyUI?

ComfyUI is a node-graph front-end for diffusion models. The trade-off vs.
one-click tools (A1111, Forge, etc.) is:

- **Cost:** steeper learning curve, uglier UI.
- **Payoff:** you can see and modify every step of the pipeline. For video,
  where a "workflow" is the product, this is essential. You will end up
  making dozens of small, specialised graphs.

Everything in this course assumes ComfyUI. If you are coming from A1111,
the mental flip is: instead of one big settings panel, each setting is a
node you wire up.

## A mental model for video diffusion

A good way to think about an LTX-2.3 generation:

```text
   [Text prompt]   [Optional audio prompt/ref]   [Optional image(s)]
        |                    |                           |
        v                    v                           v
  Gemma multimodal encoder (text + audio, jointly)   VAE encode
        |                    |                           |
        +--------------------+---------------------------+
                             v
                  LTX-2.3 Diffusion Transformer
                  - takes noisy video+audio latent
                  - iterates N denoising steps
                  - conditioned on Gemma tokens + image(s)
                             |
                             v
                 VAE decode (video latent -> frames)
                 Audio decode (audio latent -> waveform)
                             |
                             v
                 Video frames  +  synced audio track
                             |
                             v
                 (Optional) frame interpolation,
                 upscaling, colour grading, audio
                 replacement / foley
```

The three boxes to pay attention to as a beginner:

1. **Conditioning in** (text via Gemma + optional audio + optional
   images + optional control signals).
2. **Denoising** (steps, sampler, CFG/guidance — same concepts as
   image diffusion but tuned for video).
3. **Dual decode** — LTX-2.3 emits video *and* audio. You can ignore
   the audio for animation tracks that will be scored later, but it
   exists and sometimes produces surprisingly useful reference
   soundtracks.

The VAE stages are mostly "just work" once set up. Post-processing
(interpolation, upscaling) is swappable and we cover it in module 09.

## Keyframes in diffusion video

"Keyframe" means two slightly different things, and you need to keep them
straight:

- **Traditional animation keyframe:** a hand-drawn pose that defines a
  beat; in-betweens are drawn between keyframes.
- **Diffusion video keyframe:** an image (or latent) that pins the output
  at a specific time index — typically the first frame, sometimes the last
  frame, occasionally middle anchors.

In LTX-2.3, the diffusion-video sense dominates: you provide an image
as the start frame (I2V), and optionally an end frame for
interpolation-style generation. We will use this to reproduce the
*effect* of traditional keyframing — define the pose beats, let the
model generate the in-betweens.

## What you will build

By the end of this course you will have:

- A library of 4–6 reusable ComfyUI workflow graphs.
- A prompt/LoRA cheat sheet for your chosen style track.
- A 10–20 second animated short.
- Enough fluency to read a new LTX-2.3 example workflow and understand
  what every node is doing within a few minutes.

Continue to [02-installation-setup.md](02-installation-setup.md).
