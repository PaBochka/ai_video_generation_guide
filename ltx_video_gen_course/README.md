# ComfyUI + LTX-2.3 — Animation Course

A practical, project-oriented course that takes you from installing ComfyUI
to producing a stylised short animation (90s SpiderMan, classic anime, or a
mixed-media piece in the spirit of the reference video you linked).

> **Model context.** This course targets **LTX-2.3** (Lightricks, 2025).
> LTX-2.3 is a different generation from the older **LTXV 0.9.x / "LTX
> Video"** line: it uses a **Gemma-based multimodal encoder (text +
> audio)** instead of T5, and produces **synchronised video + audio
> natively**. Older LTXV-0.9 workflows will not load LTX-2.3, and vice
> versa. If you find a pre-2025 LTXV tutorial online, adapt the *ideas*
> but expect the *nodes, encoder, and conditioning graph* to be
> different.

The course assumes you have a GPU with at least 12 GB VRAM (24 GB is more
comfortable for LTX-2.3 at higher resolutions) and some tolerance for
reading node graphs. No prior ComfyUI experience is required, but familiarity
with Stable Diffusion concepts (prompts, samplers, CFG) will help.

## How to use this course

1. Work through the modules in order — each one builds on the previous.
2. After each module, open ComfyUI and reproduce the workflow before moving on.
   Nothing in generative video sticks unless you touch the nodes.
3. Save your workflows as `.json` inside the `workflows/` folder you create in
   your own ComfyUI install. Version them.
4. Keep a "prompt journal" — a plain text file where you paste prompts,
   seeds, sampler settings, and a one-line note on what you liked/disliked.
   This is the single most underrated habit in video diffusion work.

## Course map

| # | Module | What you learn |
|---|---|---|
| 01 | [Introduction](01-introduction.md) | What LTX-2.3 is, why ComfyUI, mental model for video+audio diffusion |
| 02 | [Installation & Setup](02-installation-setup.md) | ComfyUI, ComfyUI-Manager, LTX-2.3 weights, Gemma encoder, custom nodes |
| 03 | [LTX Model Fundamentals](03-ltx-model-fundamentals.md) | DiT architecture, latent video, Gemma encoder, synced audio, timing |
| 04 | [Text-to-Video Basics](04-text-to-video-basics.md) | Minimal T2V graph, prompt anatomy, motion descriptors |
| 05 | [Image-to-Video Basics](05-image-to-video-basics.md) | I2V conditioning, how to preserve identity, first-frame tricks |
| 06 | [Keyframes & Temporal Control](06-keyframes-techniques.md) | Start/end frames, interpolation, scene beats, pacing |
| 07 | [LoRA Fundamentals for Video](07-lora-fundamentals.md) | What a LoRA actually is, training data bias, strengths |
| 08 | [Multi-LoRA Workflows](08-multi-lora-workflows.md) | Stacking, blending, region/time-scoped LoRAs, conflict resolution |
| 09 | [Advanced Workflow Patterns](09-advanced-workflow-patterns.md) | ControlNet-style conditioning, upscaling, frame interpolation, audio sync |
| 10 | [Style Guides: 90s SpiderMan & Anime](10-style-guides.md) | Concrete LoRA stacks, prompt libraries, shot lists for each style |
| 11 | [Final Project](11-final-project.md) | 10–20 second animated short, from storyboard to delivery |

## Reference project

Your capstone (module 11) is a 10–20 second stylised short. Pick one track:

- **Track A — 90s SpiderMan**: cel-shaded, rim light, ink outlines, crash-zooms,
  speed lines. Aim for the feel of the 1994 animated series opening.
- **Track B — Anime**: choose a sub-style (90s Madhouse, modern Kyoto Animation,
  Ghibli pastoral). Emphasis on limited animation with expressive key poses.
- **Track C — Mixed-media**: in the spirit of the reference video you linked,
  combine live-action plates with animated overlays, driven by I2V + LoRA.

Whichever track you pick, you will end up with a folder containing:
storyboard, per-shot prompts, workflow `.json`, raw outputs, and a final
edited cut.

## Conventions used in this course

- Node graphs are described in plain text as pipelines: `NodeA -> NodeB -> NodeC`.
  Where layout matters, an ASCII diagram is provided.
- All sampler numbers, step counts, and CFG values are **starting points**,
  not recipes. Video models are more sensitive to prompt wording than to
  sampler micro-tuning.
- LoRA strengths are given as `lora_name @ model:0.8 / clip:0.8`. Model and
  CLIP weights are tuned independently.

Start with [01-introduction.md](01-introduction.md).
