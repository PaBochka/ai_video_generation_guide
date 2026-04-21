# 02 — Installation & Setup

This module gets you to a working ComfyUI with LTX-2.3 running a sample
workflow end-to-end. Do not skip the smoke test at the end.

> **Remember the encoder.** LTX-2.3 uses a **Gemma-based multimodal
> encoder** (text + audio), not T5 / CLIP. This means three separate
> large weight files need to live in the right folders before anything
> loads: the LTX-2.3 checkpoint, the Gemma encoder, and the LTX-2.3
> VAE (video + audio). If you are following an old LTXV 0.9.x tutorial,
> the "CLIPLoader / T5" step it shows does not apply here.

## Hardware reality check

LTX-2.3 is heavier than LTXV 0.9.x because the Gemma encoder is large
and audio co-generation adds latent channels. Treat the table below as
rough rather than definitive — check the official LTX-2.3 model card
for current numbers.

| VRAM | What is realistic with LTX-2.3 |
|------|-------------------|
| 8 GB | Very tight. Expect OOM without aggressive offloading (fp8 + CPU-offload Gemma). Not recommended for learning. |
| 12 GB | Low-resolution, short clips, with fp8 weights and Gemma offloaded. Usable for learning. |
| 16 GB | Mid resolution, 4–5 s clips with audio, fp8. Acceptable. |
| 24 GB (3090/4090) | Native resolution, 5–10 s clips with audio, multi-LoRA stacks. Sweet spot. |
| 48 GB+ | Full resolution, longer clips, heavy LoRA stacking, batching, high-precision Gemma. |

CPU and RAM matter more than with LTXV 0.9.x — Gemma offloading goes
to system RAM, so 32 GB+ RAM is a real benefit. NVMe SSD helps a lot
because model weights are large and the Gemma weights in particular
get swapped during multi-LoRA workflows.

## Install ComfyUI

Follow the official install path for your OS. The canonical source is the
`comfyanonymous/ComfyUI` GitHub repo — read its README, then come back.
A minimal outline:

```
git clone https://github.com/comfyanonymous/ComfyUI
cd ComfyUI
python -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate
pip install -r requirements.txt
# Install PyTorch with the CUDA version matching your driver — follow the
# pytorch.org selector, do not blindly copy a version from a tutorial.
python main.py
```

Open `http://127.0.0.1:8188` in a browser. You should see an empty node
canvas with a default workflow.

## Install ComfyUI-Manager

ComfyUI-Manager is not optional for this course. It handles installing
custom nodes, updating them, and resolving missing-node errors when you
load someone else's workflow.

```
cd ComfyUI/custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager
```

Restart ComfyUI. You will see a "Manager" button in the top menu.

## Install LTX-2.3 custom nodes

LTX-2.3 ships its own ComfyUI node pack. Use Manager:

1. Open Manager -> Install Custom Nodes.
2. Search for "LTX-2" or "LTX Video" (the official pack is from
   Lightricks). Pick the LTX-2.3 pack, not the older LTXV 0.9.x pack
   if both are listed.
3. Install, then restart ComfyUI.

Alternatively, via git (check the current canonical repo — Lightricks
publishes LTX-2 nodes either in the continuation of `ComfyUI-LTXVideo`
or a separate LTX-2 repo):

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/Lightricks/ComfyUI-LTXVideo
```

Verify: after restart, double-click the canvas and search for `LTX` —
you should see LTX-2.3-specific loader, encoder, sampler, and VAE
decode nodes. Exact node names depend on the pack version; use the
*names you see* in your install when rebuilding the graphs in this
course. The tell that you have LTX-2.3 (not 0.9.x) is a **Gemma
encoder node** present in the official example workflow, and an
**audio output** pin on the sampler or decoder.

## Download the model weights

You need at least **four** groups of files for LTX-2.3 (vs. three for
older LTXV). Put them in the paths shown (relative to `ComfyUI/`):

| What | Where | Notes |
|------|-------|-------|
| LTX-2.3 main checkpoint | `models/checkpoints/` (or an LTX-specific subfolder the node pack expects) | The video+audio DiT weights. Precision options: fp16 / bf16 / fp8. fp8 on 12–16 GB. |
| LTX-2.3 Gemma encoder | `models/text_encoders/` (sometimes a dedicated subfolder like `models/LLM/` or `models/gemma/` — follow the node pack) | The multimodal encoder. Large (several GB). **Do not substitute T5** — LTX-2.3 will not load. |
| LTX-2.3 VAE | `models/vae/` | Video VAE. Often bundled with audio VAE or as a separate file. |
| LTX-2.3 audio VAE (if separate) | `models/vae/` or `models/audio_vae/` | Decodes audio latents to waveform. Some releases bundle it into the main VAE file. |

Source: the Lightricks Hugging Face repo for LTX-2.3. Read the model
card on the day you download — it is the source of truth for which
files are bundled vs. separate in the release you are pulling.

**Precision picking:**

- **bf16** or **fp16** on 24 GB+. Highest quality.
- **fp8** on 12–16 GB. Small quality hit, big VRAM saving.
- Gemma encoder can usually be loaded in lower precision (int8 / fp8)
  independently of the main DiT. If your node pack exposes this, use
  it — Gemma is large and precision-tolerant as an encoder.

## Recommended auxiliary custom nodes

Install these via Manager. You will use all of them in later modules:

- **ComfyUI-VideoHelperSuite** — `VHS_VideoCombine`, `VHS_LoadVideo`, etc.
  Essential for saving outputs as mp4/webm and loading reference clips.
- **ComfyUI-Frame-Interpolation** — RIFE / FILM nodes for doubling frame
  rate in post.
- **ComfyUI-Custom-Scripts** (pythongosssss) — quality-of-life: node auto-
  complete, prompt history.
- **rgthree-comfy** — power tools: muters, bypassers, context nodes. Makes
  complex graphs manageable.
- **ComfyUI-Impact-Pack** — conditional sampling, image filters. Useful
  for I2V preprocessing.

## Folder layout you will use

Exact filenames come from the LTX-2.3 release you download — copy the
names your files actually have. The shape is the important part:

```text
ComfyUI/
  models/
    checkpoints/
      ltx-2.3-fp8.safetensors          # or ltx-2.3-bf16.safetensors
    text_encoders/                     # or models/LLM/, follow node pack
      ltx-2.3-gemma-encoder.safetensors
    vae/
      ltx-2.3-vae.safetensors          # video VAE (+ audio if bundled)
      ltx-2.3-audio-vae.safetensors    # only if distributed separately
    loras/
      style/                           # organise by purpose
        90s-spiderman-v2.safetensors
        anime-madhouse-90s.safetensors
      character/
      motion/
  user/default/workflows/
    01-t2v-basic.json
    02-i2v-basic.json
    ...
  output/
    2026-04-20-spiderman-test-01/
      0001.mp4                         # has synced audio by default
      prompt.txt
```

The per-output dated subfolder discipline is important. Video iteration
generates garbage fast; you need to know which mp4 went with which prompt
two weeks later. With LTX-2.3 you are also saving audio-bearing mp4s —
keep the audio on for review; strip it at delivery time if you prefer
to score in post.

## Smoke test

Before moving on, run the **official LTX-2.3 example workflow**. This
is the single most important step of setup — it's your ground-truth
reference for node names, pin wiring, and encoder configuration. Every
workflow later in this course is a modification of this example.

1. In ComfyUI: Workflow menu -> Browse Templates -> find LTX-2 / LTX-2.3
   examples (installed with the node pack). Or load the example `.json`
   from the node pack's `example_workflows/` folder.
2. Use the default prompt. Do not change anything else.
3. Queue the prompt. Watch the console for errors.
4. Expected: a 2–5 second mp4 saved in `output/`, **with synchronised
   audio**. If your speakers are off, play it back with sound on — the
   audio is a feature, not a bug.

If this works, you are ready. If it does not, 95% of failures fall into:

- **Missing model file** — the loader node has a dropdown empty or red.
  Fix: actually download the file to the right folder, then click the
  refresh button at the top right of ComfyUI.
- **Missing nodes (red nodes)** — a custom node pack is not installed.
  Fix: Manager -> Install Missing Custom Nodes.
- **OOM** — reduce resolution, reduce frame count, switch to fp8 weights,
  enable CPU-offload for Gemma, enable `--lowvram` on startup.
- **Gemma encoder not found** — you either didn't download it, put it
  in the wrong subfolder (`text_encoders/` vs. `LLM/` vs. `gemma/`
  varies by pack), or the file is truncated. Re-check the node pack's
  README for the exact folder.
- **"T5 expected" or `CLIPLoader` node red** — you are accidentally
  loading an **older LTXV 0.9.x example workflow** instead of the
  LTX-2.3 one. Open the correct LTX-2.3 example and discard the old one.
- **Audio missing from the output mp4** — the audio VAE wasn't loaded,
  or your `VHS_VideoCombine` node has no audio input pin wired.
  Inspect the example workflow's audio path and replicate it.

When the smoke test produces a playable mp4 with audio, continue to
[03-ltx-model-fundamentals.md](03-ltx-model-fundamentals.md).
