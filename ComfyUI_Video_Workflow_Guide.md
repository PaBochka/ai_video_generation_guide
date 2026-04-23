# Building Text/Image-to-Video Workflows in ComfyUI
### A Comprehensive, Model-Agnostic Guide

---

## 1. Understanding the Pipeline

Every text-to-video or image-to-video workflow in ComfyUI follows a similar high-level pipeline, regardless of the specific model (Wan, HunyuanVideo, CogVideoX, AnimateDiff, LTX-Video, Mochi, etc.). The core idea mirrors image generation but adds a **temporal (time/frame) dimension**.

**The universal pipeline looks like this:**

```
Load Model → Encode Conditioning → Configure Latent Space → Sample (Denoise) → Decode → Export
```

Each of these stages maps to one or more node groups in ComfyUI.

---

## 2. Core Workflow Components

### 2.1 Model Loading Nodes

These nodes load the pretrained weights into memory. Video models are typically large (10–30+ GB), so VRAM management matters here.

**Common node types:**

- **Checkpoint Loader** — Loads a single combined file (`.safetensors` / `.ckpt`) containing the UNet/DiT, VAE, and text encoder(s). Some video models ship as a single checkpoint.
- **Diffusion Model Loader (UNet/DiT Loader)** — Loads only the denoising backbone (UNet or DiT transformer). Used when the model components are distributed across separate files.
- **VAE Loader** — Loads the Variational Autoencoder separately. Video VAEs are 3D-aware (they encode/decode along the temporal axis too).
- **CLIP / Text Encoder Loader** — Loads the text encoder(s). Many modern video models use dual encoders (e.g., CLIP + T5-XXL, or CLIP-L + CLIP-G).

**Key considerations:**

- **Precision / dtype**: Most video models support `fp16`, `bf16`, or `fp8`. Lower precision reduces VRAM at slight quality cost. Nodes often have a `dtype` or `weight_dtype` parameter.
- **Device offloading**: Some loader nodes support CPU offloading or sequential loading to fit large models on consumer GPUs (16–24 GB VRAM).
- **Quantization**: Some workflows use GGUF-quantized models (Q4, Q8, etc.) for dramatically lower VRAM usage. These require special loader nodes (e.g., GGUF-based loaders).

---

### 2.2 Text Conditioning (Prompting)

Text conditioning translates your text prompt into embeddings that guide the denoising process.

**Typical nodes:**

- **CLIP Text Encode** — The standard node. Takes a text string and a CLIP model, outputs a `CONDITIONING` object. For video, you usually need both a positive (what you want) and a negative (what to avoid) conditioning.
- **Dual CLIP Encode** — For models that use two text encoders (common in modern architectures). Encodes through both and merges the outputs.
- **T5 Text Encode** — Some video models use T5-XXL instead of or alongside CLIP. Separate node for T5-based encoding.

**Tips for video prompting:**

- Describe **motion and temporal changes**, not just a static scene. E.g., "A woman walks through a forest, camera slowly panning left, leaves falling."
- Many video models respond well to **cinematic/camera language**: "dolly shot," "tracking shot," "slow motion," "timelapse."
- Negative prompts commonly include: "blurry, distorted, static, low quality, watermark, text, morphing, flickering."
- Some models support **per-frame or temporal prompting** via special conditioning nodes that let you assign different prompts to different time regions.

---

### 2.3 Image Conditioning (Image-to-Video)

For image-to-video workflows, you provide one or more reference images that the model uses as the starting frame(s) or style reference.

**Typical nodes and approaches:**

- **Load Image** — Loads your input image from disk.
- **Image Encode (VAE Encode)** — Encodes the image into latent space using the video VAE. This latent is then injected into the denoising process, typically as the first frame.
- **Image-to-Video Conditioning Node** — Model-specific nodes that combine the encoded image with text conditioning to create a unified guidance signal. The model "continues" the video from the input image.
- **CLIP Vision Encode** — Some models use a CLIP Vision encoder to extract semantic features from the reference image (style/content guidance rather than pixel-perfect first-frame matching).

**Common patterns:**

- **First-frame conditioning**: The input image becomes frame 0; the model generates subsequent frames.
- **Last-frame conditioning**: Some models support specifying the end frame, generating frames that transition from start to end.
- **Start + End frame**: A few models can interpolate between two reference images.
- **Style/IP-Adapter conditioning**: Uses image features for stylistic guidance without locking in the exact first frame.

**Image preparation:**

- Resize your input image to the model's expected resolution before encoding. Common video resolutions: 512×512, 576×320, 832×480, 848×480, 1280×720.
- Aspect ratio matters — most models are trained on specific aspect ratios.

---

### 2.4 Latent Space Configuration

Before sampling, you need to define the "canvas" in latent space — including the number of frames.

**Key nodes:**

- **Empty Latent Video** — Creates a blank latent tensor with dimensions: `[batch, channels, frames, height, width]`. You set the resolution and frame count here.
- **Empty Latent Image (with batch)** — Some older setups use a standard empty latent with the batch dimension repurposed for frames.

**Parameters you control:**

- **Width & Height** — The spatial resolution of the output. Must match the model's training resolution or be a supported aspect ratio. Common: 512×512, 832×480, 1280×720.
- **Frame Count / Length** — Number of frames to generate. Models have maximum supported lengths (often 16, 24, 49, 81, 97, or 121 frames). Going beyond the trained maximum usually produces artifacts.
- **Batch Size** — Usually 1 for video workflows (the temporal dimension is separate from batch).

**Important notes:**

- Resolution and frame count directly impact VRAM usage. Doubling any dimension roughly quadruples memory.
- Some models require frame counts that follow specific formulas (e.g., `4k+1` or `8k+1` where k is an integer).

---

### 2.5 Sampling (The KSampler)

The sampler is the engine of the workflow — it iteratively denoises the latent tensor, guided by your conditioning.

**Core nodes:**

- **KSampler** — The standard ComfyUI sampler. Works with video models that plug into the standard ComfyUI pipeline.
- **KSampler Advanced** — Offers more control (start/end steps, noise injection, etc.).
- **Model-Specific Samplers** — Some video models ship with custom sampler nodes optimized for their architecture (e.g., flow-matching samplers for rectified flow models).

**Key parameters:**

| Parameter | What It Does | Typical Values |
|-----------|-------------|----------------|
| **Steps** | Number of denoising steps | 20–50 (more = higher quality, slower) |
| **CFG Scale** | How strongly the model follows your prompt | 1.0–15.0 (video models often use lower CFG, e.g., 1.0–7.0) |
| **Sampler** | The algorithm used for stepping | `euler`, `euler_ancestral`, `dpmpp_2m`, `uni_pc`, `deis` |
| **Scheduler** | Controls the noise schedule | `normal`, `karras`, `sgm_uniform`, `beta`, `linear_quadratic` |
| **Seed** | Random seed for reproducibility | Any integer; same seed = same output |
| **Denoise** | How much noise to add/remove | 1.0 for full generation; <1.0 for img2vid or refinement |

**Video-specific sampling notes:**

- Many video models use **flow matching** instead of traditional diffusion, which means they have their own scheduler logic and may not use CFG in the traditional sense (some use "guidance scale" embedded in the model rather than classifier-free guidance).
- **Denoise strength** is critical for image-to-video: lower values (0.5–0.85) preserve more of the input image; higher values give the model more creative freedom.
- **Temporal consistency** is partly controlled by the sampler — some samplers produce smoother motion than others. Euler and DPM++ 2M are generally safe defaults.

---

### 2.6 VAE Decoding

After sampling, the denoised latent must be decoded back into pixel space.

**Nodes:**

- **VAE Decode** — Decodes the latent tensor into a sequence of images (frames). For video models, this uses the video-aware VAE that understands the temporal dimension.
- **Tiled VAE Decode** — Decodes in spatial tiles to reduce VRAM usage. Essential for high-resolution outputs on limited hardware. Some nodes also tile along the temporal axis.

**Notes:**

- Video VAE decoding is memory-intensive. If you run out of VRAM at this stage, switch to tiled decoding.
- The output of VAE decode is a batch of images (one per frame), which you then assemble into video.

---

### 2.7 Video Output / Export

The final stage: combining decoded frames into a playable video file.

**Common nodes:**

- **Video Combine** (from ComfyUI-VideoHelperSuite) — The most popular node for video export. Combines frames into MP4, GIF, or WEBM.
- **Save Image Sequence** — Saves individual frames as numbered PNG/JPG files for external editing.
- **Save Animated WebP / APNG** — For short clips or web-friendly formats.

**Parameters:**

- **Frame Rate (FPS)** — Typically 8, 16, 24, or 30 fps. Match the model's training FPS for natural-looking motion.
- **Codec** — H.264 (MP4) is the most compatible. H.265 for better compression. VP9 for WEBM.
- **Quality / CRF** — Controls compression. Lower CRF = higher quality, larger file.
- **Loop Count** — For GIFs: how many times to loop (0 = infinite).

---

## 3. Optional but Common Enhancements

### 3.1 ControlNet / Guidance

ControlNets add structural guidance to generation — depth maps, edge detection, pose estimation, etc.

- **Apply ControlNet** — Attaches a ControlNet model to your conditioning. The ControlNet receives a guidance image/sequence and steers the spatial structure of each frame.
- **Common control types**: Depth, Canny edges, OpenPose, Segmentation maps, Line art.
- For video, you can provide a **sequence** of control images (one per frame) to guide motion.

### 3.2 LoRA Loading

LoRAs (Low-Rank Adaptations) fine-tune the model's behavior without replacing the full weights.

- **Load LoRA** — Loads a LoRA file and applies it to the model and/or CLIP encoder.
- Stack multiple LoRAs for combined effects (e.g., a style LoRA + a motion LoRA).
- **Strength** parameter controls how much the LoRA influences output (0.0–1.0+).

### 3.3 Upscaling / Super-Resolution

Generated video is often at lower resolution. Upscaling improves final quality.

- **Latent upscale + second pass** — Upscale the latent, then re-denoise at partial strength for added detail.
- **Pixel-space upscaling** — Use models like RealESRGAN or similar to upscale decoded frames.
- **Frame-by-frame vs. temporal upscaling** — Frame-by-frame is simpler but may introduce flicker; temporal-aware upscalers maintain consistency.

### 3.4 Frame Interpolation

Increases frame count for smoother motion without regenerating.

- **RIFE / FILM nodes** — Take existing frames and generate in-between frames.
- Useful for going from 8 fps model output to 24/30 fps for smooth playback.

### 3.5 IP-Adapter / Style Transfer

IP-Adapters allow image-based style guidance without using the image as a direct first frame.

- Load an IP-Adapter model alongside the main model.
- Feed reference images to guide style, composition, or subject appearance.

### 3.6 Attention / Memory Optimization

Critical for running large video models on consumer hardware.

- **Tiled sampling / attention** — Processes the video in spatial or temporal tiles.
- **SAGE Attention / Flash Attention** — Optimized attention implementations that reduce VRAM and increase speed.
- **CPU offloading** — Moves unused model components to system RAM.
- **Tea Cache / Temporal Caching** — Caches intermediate computations across frames to reduce redundant work.

---

## 4. Typical Workflow Layout

Here is how the nodes are typically wired together in a standard text-to-video workflow:

```
┌─────────────────┐     ┌──────────────────┐
│  Load Checkpoint │────▶│  CLIP Text Encode │──── positive conditioning ────┐
│  (or separate    │     │  (Positive)       │                               │
│   loaders)       │     └──────────────────┘                               │
│                  │     ┌──────────────────┐                               │
│                  │────▶│  CLIP Text Encode │──── negative conditioning ────┤
│                  │     │  (Negative)       │                               │
│                  │     └──────────────────┘                               │
│                  │                                                         │
│                  │──── model ──────────────────────────────────────────────┤
│                  │                                                         │
│                  │──── vae ───────────────────────┐                       │
└─────────────────┘                                 │                       │
                                                     │                       │
┌─────────────────┐                                 │     ┌─────────────┐  │
│ Empty Latent     │──── latent ────────────────────────▶│  KSampler   │◀──┘
│ Video            │                                 │     │             │
│ (set resolution  │                                 │     └──────┬──────┘
│  & frame count)  │                                 │            │
└─────────────────┘                                 │         latent
                                                     │            │
                                                     │     ┌──────▼──────┐
                                                     └────▶│  VAE Decode  │
                                                           └──────┬──────┘
                                                                  │
                                                               images
                                                                  │
                                                           ┌──────▼──────┐
                                                           │ Video Combine│
                                                           │ (export MP4) │
                                                           └─────────────┘
```

**For image-to-video**, add these before the sampler:

```
┌─────────────┐     ┌──────────────┐     ┌──────────────────────┐
│ Load Image   │────▶│ VAE Encode   │────▶│ Image-to-Video       │
│              │     │              │     │ Conditioning Node    │
└─────────────┘     └──────────────┘     │ (merges with text    │
                                          │  conditioning)       │
                                          └──────────┬───────────┘
                                                     │
                                              conditioning ──▶ KSampler
```

---

## 5. VRAM and Performance Reference

| Resolution | Frames | Approx. VRAM (fp16) | Notes |
|-----------|--------|---------------------|-------|
| 512×512 | 16 | 8–12 GB | Feasible on most GPUs |
| 512×512 | 49 | 12–18 GB | Mid-range GPUs |
| 832×480 | 81 | 16–24 GB | High-end consumer |
| 1280×720 | 97+ | 24–48+ GB | Requires offloading or quantization |

**Tips to reduce VRAM:**

- Use `fp8` or GGUF-quantized models.
- Enable tiled VAE decoding.
- Use SAGE/Flash attention if supported.
- Lower resolution and interpolate/upscale afterward.
- Reduce frame count and use frame interpolation (RIFE).
- Enable sequential CPU offloading in loader nodes.

---

## 6. Essential Custom Node Packs

ComfyUI's video capabilities rely heavily on community custom nodes. Here are the most commonly needed:

| Node Pack | Purpose |
|-----------|---------|
| **ComfyUI-VideoHelperSuite** | Video export (Video Combine), frame loading, GIF/MP4 handling |
| **ComfyUI-Manager** | Install/manage custom nodes and models from within the UI |
| **ComfyUI-GGUF** | Load GGUF-quantized models for lower VRAM |
| **ComfyUI-KJNodes** | Utility nodes (batch operations, conditioning helpers) |
| **ComfyUI-Frame-Interpolation** | RIFE/FILM frame interpolation |
| **ComfyUI-Advanced-ControlNet** | ControlNet support with temporal batching |
| **ComfyUI-Impact-Pack** | Utility pack (face detection, regional prompting, etc.) |

Model-specific packs (e.g., for Wan, HunyuanVideo, CogVideoX) provide the loader and conditioning nodes tailored to each architecture.

---

## 7. Debugging Common Issues

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Black or noisy output | Wrong VAE, mismatched model components, CFG too high | Verify all components are from the same model family; lower CFG |
| Static / frozen video | Denoise too low, prompt lacks motion, wrong frame count | Increase denoise; add motion words to prompt; check frame count formula |
| Flickering frames | Sampler instability, no temporal attention | Try `euler` or `dpmpp_2m`; ensure model has temporal layers loaded |
| Out of memory (OOM) | Resolution or frame count too high | Reduce resolution/frames; enable tiling; use quantized models |
| Color shift / banding | VAE precision issues | Use `fp32` for VAE decoding; enable tiled VAE |
| Distorted motion | CFG too high for the model | Many video models work best at CFG 1.0–6.0; experiment |
| First frame doesn't match input | Denoise too high in img2vid | Lower denoise to 0.6–0.8; check image encoding |
| Blurry output | Not enough steps, resolution mismatch | Increase steps; match model's native resolution |

---

## 8. General Best Practices

1. **Start small**: Begin with low resolution and few frames to test your prompt and settings, then scale up.
2. **Match model specs**: Always check what resolution, frame count, aspect ratio, and FPS the model was trained on. Fighting against training data produces bad results.
3. **Seed locking**: Keep the seed fixed while tweaking parameters so you can compare changes fairly.
4. **Save workflow**: Use ComfyUI's built-in "Save" to preserve working configurations. Video workflows are complex and easy to break.
5. **Check the model card**: Every model has specific requirements for conditioning, scheduling, and configuration. Read the documentation before building your workflow.
6. **Batch test prompts**: Use ComfyUI's queue system to batch multiple generations with different prompts/seeds.
7. **Post-process externally**: For professional results, export frame sequences and use dedicated video editors (DaVinci Resolve, FFmpeg) for final cuts, color grading, and audio.

---

## 9. Model Spotlight: LTX-2 / LTX-2.3

LTX-2 (Lightricks) is a joint **audio + video** DiT. Because it generates both modalities in a single denoising pass, the latent entering the sampler is a *concatenated AV latent*, not a plain video latent. That's why the generic pipeline in §4 fails on LTX-2 with shape-mismatch errors at the sampler — the model expects an extra modality dimension. **LTX-2.3 supersedes LTX-2** (better detail, 9:16 portrait, lower-noise audio, stronger I2V, refreshed distilled weights, formal two-stage base+upscale pipeline). All new workflows should target LTX-2.3.

### 9.1 Required custom node pack

**ComfyUI-LTXVideo** by Lightricks — install via Manager (search "LTXVideo") or:
```
git clone https://github.com/Lightricks/ComfyUI-LTXVideo.git
pip install -r requirements.txt
```
Reference workflow JSONs ship in `custom_nodes/ComfyUI-LTXVideo/example_workflows/2.3/` (Single_Stage, Two_Stage_Distilled, ICLoRA_Motion_Track, FLF2V).

### 9.2 Model files and placement

| Folder | File | Notes |
|--------|------|-------|
| `models/checkpoints/` | `ltx-2.3-22b-dev.safetensors` (46 GB BF16) | 48 GB+ cards, max quality |
| `models/checkpoints/` | `ltx-2.3-22b-dev-fp8.safetensors` (~29 GB) | 24–32 GB cards, recommended default |
| `models/checkpoints/` | `ltx-2.3-22b-dev-nvfp4.safetensors` (~21 GB) | Blackwell, ~16 GB cards |
| `models/checkpoints/` | GGUF Q2_K..Q8_0 (8–25 GB) | Load with `UnetLoaderGGUF` |
| `models/checkpoints/` | `ltx-2.3-22b-distilled-1.1.safetensors` | Fewer steps, lower CFG |
| `models/loras/` | `ltx-2.3-22b-distilled-lora-384.safetensors` (7.6 GB) | Apply on dev at strength 0.6–0.8 → fast sampling |
| `models/loras/` | IC-LoRAs (Union depth+edges, motion, pose, Canny, camera dolly/jib, detailer) | Optional structural control |
| `models/text_encoders/` | `gemma_3_12B_it_fp8_scaled.safetensors` (13 GB) | Recommended Gemma variant |
| `models/text_encoders/` | `gemma_3_12B_it_fp4_mixed.safetensors` (9.5 GB) | Lower VRAM |
| `models/latent_upscale_models/` | `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` | For stage-2 latent upscale |
| `models/latent_upscale_models/` | `ltx-2.3-temporal-upscaler-x2-1.0.safetensors` | Optional temporal upscale |

**Direct download URLs (verified):**

| File | URL |
|------|-----|
| `gemma_3_12B_it_fp8_scaled.safetensors` (13.2 GB) | `huggingface.co/Comfy-Org/ltx-2/blob/main/split_files/text_encoders/gemma_3_12B_it_fp8_scaled.safetensors` |
| `gemma_3_12B_it_fp4_mixed.safetensors` (9.5 GB) | same repo, same folder |
| `ltx-2.3-22b-dev-fp8.safetensors` (29.1 GB) | `huggingface.co/Lightricks/LTX-2.3-fp8/blob/main/ltx-2.3-22b-dev-fp8.safetensors` |
| `ltx-2.3-22b-distilled-fp8.safetensors` (29.5 GB) | `huggingface.co/Lightricks/LTX-2.3-fp8` (same repo) |
| `ltx-2.3-22b-distilled-lora-384-1.1.safetensors` (7.6 GB) | `huggingface.co/Lightricks/LTX-2.3` |
| `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` (996 MB) | `huggingface.co/Lightricks/LTX-2.3` |
| `ltx-2.3-temporal-upscaler-x2-1.0.safetensors` (262 MB) | `huggingface.co/Lightricks/LTX-2.3` |
| Camera-control LoRAs (LTX-2 19B; load on 2.3 too) | `huggingface.co/collections/Lightricks/ltx-2` |

Note: FP8 checkpoints live in the **separate** `Lightricks/LTX-2.3-fp8` repo (not the BF16 `Lightricks/LTX-2.3` repo).

### 9.3 Text encoder: Gemma 3 12B Instruct (NOT generic CLIP)

Gemma 3 is a 12B decoder LLM with its own tokenizer, projection, and hidden-state dims — **the standard `CLIPLoader` / `DualCLIPLoader` will silently fail or produce shape errors at the DiT**. Symptoms include console warnings like `clip missing: ['gemma3_12b.logit_scale', ...]`, gibberish output, or empty conditioning.

**Correct loader:** `LTXAVTextEncoderLoader` (or `DualCLIPLoaderGGUF` with `type="ltxv"` for GGUF). Place a single consolidated `.safetensors` directly in `models/text_encoders/` — **do not** `git clone` from HuggingFace; LFS pointer files leave tokenizer assets missing and "silently break performance" (LTX-2 issue #106).

**Optimization for Gemma's VRAM cost:**
- `LTXVSaveConditioning` writes encoded conditioning to `models/embeddings/` as `.safetensors`; `LTXVLoadConditioning` reloads it instantly. Encode Gemma once, reuse across many generations.
- `GemmaAPITextEncode` offloads encoding to Lightricks' free API — zero local Gemma VRAM.

### 9.4 Text-to-video node chain (canonical two-stage)

```
CheckpointLoaderSimple (ltx-2.3-22b-dev-fp8)
   ├── MODEL ──▶ (optional) LoraLoader (distilled-lora-384, str 0.5 distilled / 0.2 dev)
   └── VAE   ──▶ to decode stage

LTXAVTextEncoderLoader (gemma_3_12B_it_fp8_scaled)
   └── CLIP ──▶ CLIPTextEncode (positive)
             └▶ CLIPTextEncode (negative)
                  └─▶ LTXVConditioning (frame_rate=25)
                         └── pos / neg conditioning

EmptyLTXVLatentVideo (W, H, length=97, batch=1) ─┐
LTXVEmptyLatentAudio (length ≥ video length)    ─┴─▶ LTXVConcatAVLatent ─▶ AV latent

LTXVScheduler (steps=20, max_shift=2.05, base_shift=0.95, stretch=true, terminal=0.1) ─▶ SIGMAS
KSamplerSelect (euler / euler_ancestral)                                                ─▶ SAMPLER
RandomNoise (seed)                                                                      ─▶ NOISE
MultimodalGuider (model, pos, neg, scale=28, skip_blocks=[...])                         ─▶ GUIDER
   (or CFGGuider with cfg=1.0 when running distilled sigmas)

SamplerCustomAdvanced (NOISE, GUIDER, SAMPLER, SIGMAS, AV latent)
   └── output ──▶ LTXVSeparateAVLatent
                     ├── video latent ──▶ LTXVTiledVAEDecode  ──▶ frames
                     └── audio latent ──▶ LTXVAudioVAEDecode  ──▶ AUDIO

frames + AUDIO + fps ──▶ VHS_VideoCombine  (or CreateVideo)
```

**Stage 2 (recommended for full-quality 2× output):** take the stage-1 video latent → `LatentUpscaleModelLoader` (loads `ltx-2.3-spatial-upscaler-x2-1.1`) → `LTXVLatentUpsampler` → second `SamplerCustomAdvanced` (steps=3–4, CFG=1.0, distilled LoRA active) → `LTXVTiledVAEDecode`. The upscaler ingests the **non-fully-denoised** latent (that's why §9.7's `LTXVScheduler.terminal=0.1` matters — it leaves headroom for the second pass).

**`CFGGuider` vs `MultimodalGuider`:**
- `CFGGuider` — single CFG scalar, video-only (or distilled where CFG is off). Used in two-stage distilled detail passes (typically `cfg=1`).
- `MultimodalGuider` — required when generating audio+video together. Exposes per-modality scales, **STG (Spatio-Temporal Guidance)** with `skip_block_indices` / `stg_scale` / `rescale`, and cross-modal sync. Published 2.3 single-stage workflows use `scale=28` on this guider; STG defaults are not officially documented (start with the values from the shipped JSON template).

Helpful wrappers:
- **`LTXVCropGuides`** — trims IC-LoRA / conditioning guides back to output frame count after upsampling.
- **`LTXVNormalizingSampler`** — wraps the sampler to prevent latent overbake at high CFG. Use on stage 1 only; skip for inpaint/extend.

**Example prompts (verbatim from `LTX-2.3_T2V_I2V_Single_Stage_Distilled_Full.json`):**

> **Positive (T2V + audio):** *"A traditional Japanese tea ceremony takes place in a tatami room as a host carefully prepares matcha. Soft traditional koto music plays in the background, adding to the serene atmosphere. The bamboo whisk taps rhythmically against the ceramic bowl while water simmers in an iron kettle. Guests kneel in formal seiza position, watching in respectful silence. The host bows and presents the tea bowl, turning it precisely before offering it to the first guest with soft-spoken words."*
>
> **Negative:** *"pc game, console game, video game, cartoon, childish, ugly"*

Note the Lightricks negative-prompt convention: short, **concept-level** tokens (media categories + quality adjectives), **not** the long SD-style "low quality, blurry, distorted, extra fingers..." list. The shipped templates literally use only those six tokens.

### 9.5 Image-to-video chain

Same as §9.4, but before `LTXVConcatAVLatent`:

```
LoadImage ──▶ LTXVPreprocess ──▶ LTXVImgToVideoInplace (strength=1.0)
                                       └── injects image as frame 1 of EmptyLTXVLatentVideo
```

**Critical:**
- `LTXVPreprocess` degrades the input to match training compression. **Skipping it produces near-static / ghostly output.**
- Resize input to **half** target resolution for stage 1, full for stage 2.
- Prompt should describe **motion**, not the scene (the model already sees the scene).
- Multi-clip extension: `GetImageRangeFromBatch` grabs last N frames of a prior clip → use as connector, strip overlap before concat.
- FLF2V (first / first-middle-last) workflows live in `example_workflows/2.3/`.

### 9.6 Resolution and frame constraints

- **32-stride rule**: `width % 32 == 0` and `height % 32 == 0`. Full-res ceiling: 1280×720 up to 2560×1440.
- **Frame formula**: `frames = 8n + 1` → valid: 9, 17, 25, 33, 41, 49, 57, 65, 73, 81, 97, 105, 121, 129, 161, 257 (max ~257).
- **FPS**: trained for 24 / 25 / 30. 50 fps works but doubles cost. Keep `LTXVConditioning.frame_rate` and `VideoCombine.fps` identical.
- **Audio latent length must be ≥ video latent length** even for silent output, or sampling fails. For a driving audio track, `TrimAudioDuration` to exactly match video length before encoding.

**Practical (W, H, frames) presets:**

| Preset | Stage-1 size | Stage-2 size (after x2) | Frames | Duration @ 25fps | Aspect |
|--------|--------------|-------------------------|--------|------------------|--------|
| Square preview | 640×640 | 1280×1280 | 49 | 2.0s | 1:1 |
| Landscape short | 768×512 | 1536×1024 | 97 | 3.9s | 3:2 |
| Cinematic 16:9 | 832×480 | 1664×960 | 121 | 4.8s | ~16:9 |
| Portrait 9:16 (2.3 only) | 480×832 | 960×1664 | 121 | 4.8s | 9:16 |
| Long single-shot | 768×512 | 1536×1024 | 257 | 10.3s | 3:2 |

For longer than ~10s use `LTXVLoopingSampler` (§10.5) — VRAM grows roughly linearly with frame count.

### 9.7 Sampler / scheduler presets

**Terminology first.** "Distilled" is a **sampling mode** — few steps + `CFGGuider cfg=1.0` + distilled LoRA active — **not** a scheduler. It appears in two shipped templates, which behave differently:

- **`Single_Stage_Distilled_Full.json`** — distilled mode for the whole render. Uses `ManualSigmas` (8 hard-coded sigmas), which **replaces** `LTXVScheduler`.
- **`Two_Stage_Distilled.json`** — `LTXVScheduler` in **both** stages. Stage 1 is the full 20-step dev pass ending at `terminal=0.1`; stage 2 runs the distilled mode (3–4 steps, CFG=1, distilled LoRA on) on the 2×-upscaled latent to finish the remaining ~10% of denoise. `ManualSigmas` is **not** used here — `LTXVScheduler` just runs for fewer steps.

So the right mental model is: distilled mode is a *fast refinement primitive*. Run it standalone (single stage, `ManualSigmas`) for speed, or as stage 2 (`LTXVScheduler`, 3–4 steps) after a full-quality stage 1.

| Stage | Steps | CFG | Sampler | Notes |
|-------|-------|-----|---------|-------|
| Dev, stage 1 | 20 (up to 25–35) | 4.0 (range 2–5) on `MultimodalGuider` (or `scale=28` per template) | `euler` / `euler_ancestral` | `LTXVScheduler` + `LTXVNormalizingSampler` |
| Dev, stage 2 (upsample, distilled mode) | 3–4 | 1.0 (`CFGGuider`) | `euler` | `LTXVScheduler` (short run), distilled LoRA active |
| Single-stage distilled | 8 | 1.0 (`CFGGuider`) | `euler` | `ManualSigmas` (see below), replaces `LTXVScheduler` |

**`LTXVScheduler` parameter meaning:**

| Param | Default | Effect when raised |
|-------|---------|---------------------|
| `max_shift` | 2.05 | Pushes schedule toward noisier sigmas at start → stronger global motion, looser structure |
| `base_shift` | 0.95 | Tilts the whole curve toward higher noise; lower it for a more linear flow-matching curve |
| `stretch` | True | If True, rescales sigmas so the schedule terminates at `terminal` instead of ~0 |
| `terminal` | 0.1 | Cutoff sigma; denoising stops here, leaving headroom for stage-2 refinement. Lower = cleaner stage-1; higher = more detail headroom |

**The single-stage distilled workflow uses `ManualSigmas` instead of `LTXVScheduler`** (two-stage distilled keeps `LTXVScheduler` in both stages — see the terminology note at the start of this section). The shipped 2.3 single-stage distilled JSON hard-codes:

```
sigmas = "1.0, 0.99375, 0.9875, 0.98125, 0.975, 0.909375, 0.725, 0.421875, 0.0"
```

(8 effective steps + terminal 0.) Swap the node if you want different distilled behavior; the schedule is **not** baked into the model.

### 9.8 VRAM tiers (LTX-2.3 22B)

| VRAM | Recommended setup |
|------|-------------------|
| 48+ GB | BF16 dev + Gemma BF16; full two-stage; CFG up to 5 |
| 24–32 GB | FP8 dev + Gemma fp8_scaled; two-stage; `--reserve-vram 5` |
| 16 GB | NVFP4 or GGUF Q4_K_M + Gemma fp4_mixed; distilled LoRA; tiled VAE; cache conditioning |
| ≤12 GB | GGUF Q2/Q3 + `GemmaAPITextEncode`; distilled LoRA; single stage only |

Universal: `VAEDecodeTiled` / `LTXVTiledVAEDecode`, batch=1, low-VRAM loader variants from the pack, launch with `--reserve-vram 5`.

**VAE tile parameters:**

- Generic `VAEDecodeTiled` widgets `(tile_size, overlap, temporal_size, temporal_overlap)` — pixel-space spatial tile, spatial overlap, frames per temporal tile, frame overlap. Default for LTX-2: `(512, 64, 4096, 8)`. **Safe 12 GB values:** `(256, 32, 32, 4)`.
- LTX-specific `LTXVTiledVAEDecode` widgets `(tile_height, tile_width, overlap, force_input_latent, decode_method, upscale_method)` — tile sizes are in **latent tiles** (not pixels). Two-stage distilled JSON uses `(2, 2, 6, false, "auto", "auto")`. **Safe 12 GB:** `(1, 1, 4, false, "auto", "auto")`.

### 9.9 LTX-2-specific error table

| Symptom | Cause | Fix |
|---------|-------|-----|
| Sampler shape mismatch (e.g., "4 vs 3") | Plain video latent fed to AV-aware DiT | Use `EmptyLTXVLatentVideo` + `LTXVEmptyLatentAudio` + `LTXVConcatAVLatent` |
| Sampler shape mismatch (other axes) | W/H not 32-stride or frames ≠ `8n+1`; stage-1↔stage-2 res mismatch | Snap to valid sizes |
| Gibberish / silent CLIP failure | Generic `CLIPLoader` on Gemma, or HF Git-LFS pointers | Use `LTXAVTextEncoderLoader` + single-file safetensors |
| OOM during text encode | Gemma footprint | fp4_mixed / fp8_scaled; CPU-offload Gemma; `GemmaAPITextEncode`; cache via `LTXVSaveConditioning` |
| OOM during VAE decode | High res / many frames | `VAEDecodeTiled (512, 64, 4096, 8)`; FP8 checkpoint |
| Static / ghostly I2V | Missing preprocessing | Insert `LTXVPreprocess` before `LTXVImgToVideoInplace` |
| Audio desync / silent track mismatch | Audio latent shorter than video, or wrong fps | Match audio latent length; align `LTXVConditioning.frame_rate` with `VideoCombine.fps` |
| Oversaturation at high CFG | Latent overbake | Wrap stage-1 sampler with `LTXVNormalizingSampler` |
| "Missing node" errors | Outdated `ComfyUI-LTXVideo` | Update via Manager, restart |

### 9.10 Quick-start checklist

1. Install **ComfyUI-LTXVideo** + **ComfyUI-VideoHelperSuite** + (optional) **ComfyUI-GGUF**.
2. Drop `ltx-2.3-22b-dev-fp8.safetensors` in `models/checkpoints/`.
3. Drop `gemma_3_12B_it_fp8_scaled.safetensors` in `models/text_encoders/` (single-file, no HF clone).
4. Drop `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` in `models/latent_upscale_models/`.
5. Open `example_workflows/2.3/Two_Stage_Distilled.json` from the node pack.
6. Verify W/H are 32-multiples and frames = `8n+1` before queueing.
7. First run: cache the encoded conditioning with `LTXVSaveConditioning` to skip Gemma on subsequent generations.

### 9.11 Audio prompting

LTX-2.3 uses **a single shared text prompt** for both video and audio — there is no separate audio conditioning field, and **no documented bracket-tag syntax** (`[sound: ...]` etc.). Sound cues are written as natural language inside the same positive prompt.

**What works (from shipped templates):**
- Name the source: *"Soft traditional koto music plays in the background."*
- Describe rhythm/texture: *"The bamboo whisk taps rhythmically against the ceramic bowl while water simmers in an iron kettle."*
- Speech: *"...with soft-spoken words"* or *"a child's laughter echoes briefly"* — generates speech-like audio (no lip-sync to specific words; treat dialogue as ambience, not script).
- Layering: combine ambient bed + foreground sound + occasional accent in one sentence.

**What to avoid:**
- Negative prompts targeting audio do not have an established convention. Stick with the visual-concept negatives (*"pc game, console game, video game, cartoon, childish, ugly"*).
- Don't try `[silent]` or `[no audio]` tags — they're not parsed. For silent output, just omit sound mentions and let the audio latent generate ambient quiet.
- Don't expect lyrics or specific musical keys — the audio model is ambient/cinematic, not generative-music in the Suno sense.

**Per-modality balance** is controlled at the sampler via `MultimodalGuider`'s STG / cross-modal scales, **not** at the prompt layer.

### 9.12 Worked example: 4-second cinematic T2V with audio (24 GB card)

Concrete settings to drop into a fresh `Two_Stage_Distilled.json`:

| Component | Value |
|-----------|-------|
| Checkpoint | `ltx-2.3-22b-dev-fp8.safetensors` |
| Text encoder | `gemma_3_12B_it_fp8_scaled.safetensors` via `LTXAVTextEncoderLoader` |
| Speed LoRA | `ltx-2.3-22b-distilled-lora-384-1.1.safetensors` @ str=0.2 |
| Resolution (stage 1) | 768 × 512 |
| Frames | 97 (= 8·12 + 1) |
| Audio latent length | ≥ 97 |
| FPS | 25 (`LTXVConditioning.frame_rate=25`, `VideoCombine.fps=25`) |
| Stage-1 sampler | `euler`, steps=20, `LTXVScheduler` defaults, `MultimodalGuider scale=28` |
| Spatial upscaler | `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` → 1536×1024 |
| Stage-2 sampler | `euler`, steps=4, `CFGGuider cfg=1.0` |
| VAE decode | `LTXVTiledVAEDecode (2, 2, 6, false, auto, auto)` |
| Positive prompt | (your scene + sound description, see §9.11) |
| Negative prompt | `pc game, console game, video game, cartoon, childish, ugly` |

Expected stage-1 + stage-2 generation time on a 4090: ~80–120s. Peak VRAM: ~20 GB.

---

## 10. LTX-2.3 Advanced: LoRA Stacking, Camera Control, Multi-Keyframe Anchors

This section covers what to wire on top of the base §9 pipeline when you want **directed camera motion**, **structural control** (depth/pose/Canny), or **multiple anchor frames** (first+last, first+middle+last, arbitrary keyframes).

### 10.1 LoRA loaders — two distinct nodes, not interchangeable

| Loader | Use for | Why |
|--------|---------|-----|
| `LoraLoaderModelOnly` | Distilled-speed LoRA, style LoRAs, **camera-control LoRAs** | Standard ComfyUI node. "Model-only" because LTX-2.3 has no CLIP branch on the LoRA side (Gemma sits outside) |
| `LTXICLoRALoaderModelOnly` | **IC-LoRAs** (Union, Motion-Track, Detailer, Pose, Canny) | LTX-specific. Extracts `ref_downscale_factor` (the `ref0.5` in the filename) and exposes it as a second output that must feed `LTXAddVideoICLoRAGuide` |

**Stacking is linear** — thread MODEL through each loader in sequence. Tested practical limit: ~3 LoRAs simultaneously.

```
CheckpointLoaderSimple
   └── MODEL ──▶ LoraLoaderModelOnly (distilled-lora-384-1.1, str=0.5)
                    └── MODEL ──▶ LoraLoaderModelOnly (camera or style LoRA, str=0.5–0.8)
                                     └── MODEL ──▶ LTXICLoRALoaderModelOnly (ic-lora-union, str=1.0)
                                                      ├── MODEL ──▶ CFGGuider / SamplerCustomAdvanced
                                                      └── downscale_factor ──▶ LTXAddVideoICLoRAGuide
```

Strength guidance:
- Distilled-speed LoRA: **0.5** with the distilled checkpoint, **0.2** with the full dev checkpoint.
- Camera LoRA: **0.5–0.8**. Above 0.8 → motion accumulation artifacts.
- Style LoRA: **≤0.6** when combined with an IC-LoRA, so geometry stays with the IC branch.

### 10.2 Camera-control LoRAs

All camera-motion LoRAs are **LTX-2 (19B) trained**, but they load on LTX-2.3 via `LoraLoaderModelOnly`. No 2.3-native camera LoRAs exist yet. Place them in `models/loras/ltxv/ltx2/`:

| File | Effect |
|------|--------|
| `ltx-2-19b-lora-camera-control-dolly-in.safetensors` | Camera moves toward subject |
| `ltx-2-19b-lora-camera-control-dolly-out.safetensors` | Camera pulls back |
| `ltx-2-19b-lora-camera-control-dolly-left.safetensors` | Lateral track left |
| `ltx-2-19b-lora-camera-control-dolly-right.safetensors` | Lateral track right |
| `ltx-2-19b-lora-camera-control-jib-up.safetensors` | Camera rises (crane up) |
| `ltx-2-19b-lora-camera-control-jib-down.safetensors` | Camera descends |
| `ltx-2-19b-lora-camera-control-static.safetensors` | Locks the camera (suppresses drift) |

Pan / tilt LoRAs are **not published** as of writing.

**Mid-clip camera switching is not supported natively.** There is no `LoraScheduler`, `LoraTimeMask`, or temporal-mask input on `LoraLoaderModelOnly`. The convention is:

1. Render clip A with camera LoRA #1 (e.g., dolly-in).
2. Render clip B starting from clip A's last frames, with camera LoRA #2 (e.g., jib-up). See §10.5.
3. Stitch in `VideoCombine` or an external editor.

**Reinforce the LoRA in the prompt.** No special trigger token exists for the camera LoRAs — Lightricks does not publish trigger phrases. Convention is to describe the move in plain English so the LoRA's bias and the prompt point the same direction:

| LoRA | Suggested prompt phrase |
|------|--------------------------|
| dolly-in | *"the camera slowly dollies in toward the subject"* |
| dolly-out | *"the camera pulls back, revealing more of the scene"* |
| dolly-left / right | *"camera tracks left/right alongside the subject"* |
| jib-up | *"the camera rises smoothly on a crane, revealing the landscape"* |
| jib-down | *"the camera descends from above"* |
| static | *"static locked-off camera, no pan or tilt, no camera movement"* |

If the prompt and LoRA disagree, the result is mush — the dolly-in LoRA + a prompt saying "the camera holds still" is the most common foot-gun.

### 10.3 IC-LoRAs and the guide pipeline (depth / pose / Canny / motion track)

IC-LoRAs ("in-context" LoRAs) take a **reference video** of structural cues — depth maps, edges, pose skeletons, or sparse motion tracks — and use it to constrain generation.

Available IC-LoRAs (place in `models/loras/ltxv/ltx2/`):

| File | Cues |
|------|------|
| `ltx-2.3-22b-ic-lora-union-control-ref0.5.safetensors` | Canny + Depth + Pose merged in one LoRA |
| `ltx-2.3-22b-ic-lora-motion-track-control-ref0.5.safetensors` | Sparse motion vectors (drawn tracks) |
| `ltx-2-19b-ic-lora-detailer.safetensors` | V2V refiner / detail enhancer |
| `ltx-2-19b-ic-lora-pose-control.safetensors` | Pose-only (LTX-2 era; superseded by Union for 2.3) |
| `ltx-2-19b-ic-lora-canny-control.safetensors` | Canny-only (superseded by Union) |

**Wiring (Union-Control example):**

```
LoadVideo ──▶ GetVideoComponents ──▶ ResizeImageMaskNode ─┬─▶ VideoDepthAnythingProcess ──┐
                                                          ├─▶ CannyEdgePreprocessor      ─┤
                                                          └─▶ DWPreprocessor              ─┤
                                                                                           ▼
                                                                            LTXAddVideoICLoRAGuide
                                                                            (downscale_factor input
                                                                             from LTXICLoRALoaderModelOnly)
                                                                                  │
                                  (after stage-2 spatial upsample only:)          ▼
                                                                            LTXVCropGuides
                                                                                  │
                                                                                  ▼
                                                                          positive conditioning
                                                                          ──▶ CFGGuider ──▶ Sampler
```

Notes:
- The depth/Canny/pose preprocessors are the **generic ControlNet-aux nodes** — there are no `LTXVPrepareDepth` / `LTXVPrepareCanny` wrappers. For depth use `VideoDepthAnythingProcess` (with `LoadVideoDepthAnythingModel`) → `VideoDepthAnythingOutput`.
- For motion-track IC-LoRA, replace the preprocessors with **`LTXVDrawTracks` + `LTXVSparseTrackEditor`** (you literally draw motion vectors in the editor).
- **`LTXVCropGuides`** is needed only after `LTXVLatentUpsampler` (stage 2). It trims the guide latent to match the upsampled spatial size — IC-LoRAs were trained at `ref_downscale=0.5`.
- **`LTX Add Video IC-LoRA Guide Advanced`** adds `attention_strength` and `attention_mask` for spatially localized control (e.g., apply depth only inside a region mask).

Reference workflows (in `custom_nodes/ComfyUI-LTXVideo/example_workflows/2.3/`):
- `LTX-2.3_ICLoRA_Union_Control_Distilled.json`
- `LTX-2.3_ICLoRA_Motion_Track_Distilled.json`

### 10.4 Multi-keyframe / anchor-frame I2V

Three nodes do the heavy lifting:

| Node | Role | Best for |
|------|------|----------|
| `LTXVImgToVideoConditionOnly` | Single first-frame seed via conditioning | The default I2V template |
| `LTXVImgToVideoInplace` | VAE-encodes image and **blends** into latent frames at given strength (rigid) | Hard first/last anchors that must match exactly |
| `LTXVAddGuide` | **Soft** anchor — model negotiates around it. Accepts `frame_idx` (`-1` = last frame) | Multiple keyframes, non-boundary anchors |

**Critical:** every anchor image needs `LTXVPreprocess` first (matches training compression — skip it and output goes static/ghostly), and must be sized to match the latent (use `ResizeImageMaskNode`).

#### FLF2V — First + Last Frame to Video

There is **no dedicated `LTXVFirstLastFrame` node** and no shipped `FLF2V.json` in 2.3 examples. Build it from two `LTXVAddGuide` calls:

```
LoadImage(first) ─▶ LTXVPreprocess ─▶ LTXVAddGuide(frame_idx=0,  strength=1.0) ─┐
LoadImage(last ) ─▶ LTXVPreprocess ─▶ LTXVAddGuide(frame_idx=-1, strength=1.0) ─┴─▶ positive conditioning
```

#### FMLF2V — First + Middle + Last

Add a third `LTXVAddGuide` call at the midpoint:

```
LTXVAddGuide(frame_idx = total_frames // 2, strength = 0.5)   ◀── middle anchor (lower strength)
```

Recommended starting strength for the middle anchor is **~0.5** (per Kijai). Tune up if the middle drifts; tune down if it creates a visible "snap."

#### Arbitrary anchor frames at any frame index

Two routes:

1. **Official:** chain N copies of `LTXVAddGuide`, one per anchor, each with its own `frame_idx` and `strength`.
2. **Kijai fork:** `LTXVAddGuideMulti` accepts a list of `(frame_index, image, strength)` tuples in one node. Lives in `Kijai/LTX2.3_comfy` (HuggingFace), not the official Lightricks repo.

There is **no per-anchor denoise schedule** — `strength` per call is the only per-anchor lever. The sampler sigmas (`ManualSigmas` / `LTXVScheduler`) are global.

### 10.5 Video extension, continuation, and looping

For long videos, generate in chunks and let the model overlap each chunk with the tail of the previous one.

**Recommended (official):** use `LTXVLoopingSampler` instead of `SamplerCustomAdvanced`. **Note:** it requires an `STGGuiderAdvanced`, not `CFGGuider` or `MultimodalGuider`.

Full parameter table (from node source):

| Param | Default | Range | Meaning |
|-------|---------|-------|---------|
| `temporal_tile_size` | 80 | 24–1000 | Frames generated per tile |
| `temporal_overlap` | 24 | 16–80 | Frames reused from the previous tile (each tile advances by `tile_size − overlap`) |
| `temporal_overlap_cond_strength` | 0.5 | 0–1 | How strongly the previous tile's tail constrains the new tile (raise to 0.7 for stronger continuity, lower to 0.3 if seams look frozen) |
| `guiding_strength` | 1.0 | 0–1 | Strength of IC-LoRA / guiding latents |
| `cond_image_strength` | 1.0 | 0–1 | Strength of the conditioning image (I2V) |
| `horizontal_tiles` / `vertical_tiles` | 1 | 1–6 | Spatial tiling (for very high res) |
| `spatial_overlap` | 1 | 1–8 | Spatial tile overlap |

**Concrete recipe — 250 frames @ 25 fps (10 seconds):**

```
temporal_tile_size = 80
temporal_overlap   = 24
temporal_overlap_cond_strength = 0.5
```

→ `ceil((250 − 24) / (80 − 24)) ≈ 5 tiles`. Each tile generates 80 frames; 24 overlap with the previous tile. Peak VRAM is roughly `temporal_tile_size / total_frames` of single-shot peak (≈ 32% here), at the cost of per-tile overhead and minor seam softness near boundary frames.

**Manual stitch fallback (no looping sampler):** decode clip A → take its last N frames as a batch → feed into a fresh run with `LTXVAddGuide` at `frame_idx=0..N-1`. Strip the overlapping frames before concatenating in `VideoCombine`.

**Looping (first frame == last frame):** no first-class feature. Set the same image as both the first and last anchor (`LTXVAddGuide` at `frame_idx=0` and `frame_idx=-1`, both strength 1.0) and the sampler converges toward a loop. Quality varies — easier scenes (slow camera, repetitive motion) loop more cleanly than complex action.

### 10.6 Combining camera LoRAs with keyframe anchors

Mechanically compatible — `LoraLoaderModelOnly` modifies MODEL before `LTXVAddGuide` operates on conditioning, so there's no node-level conflict. Behavioral caveats:

- **Anchor frames win at boundaries.** A "dolly-in" LoRA + a last-frame anchor that contradicts the zoom direction will fight; the result looks broken.
- **IC-LoRA + camera LoRA**: prefer the IC-LoRA for trajectory and either drop the camera LoRA or hold it at **≤0.5**. Both want to control motion; the IC-LoRA is more specific.
- **FLF2V + camera LoRA**: documented as working. **FMLF2V + camera LoRA**: not documented either way — test on short clips first.

### 10.7 What's NOT supported (don't waste time looking)

- Mid-clip LoRA switching (no `LoraScheduler` or temporal-mask LoRA loader for LTX).
- Pan / tilt camera LoRAs (not released).
- Per-anchor denoise schedule.
- A native loop / first==last template.
- Pan/tilt or roll inferred from the static LoRA — `static` only suppresses motion.

For unsupported behaviors, the standard workaround is **render in segments and stitch** (§10.5).

### 10.8 Worked example: FLF2V with intermediate anchor at frame 49

Goal: generate a 97-frame clip where frame 0 = portrait A, frame 48 = portrait B (mid-pose), frame 96 = portrait C, with a slow dolly-in throughout.

**Settings:**

| Component | Value |
|-----------|-------|
| Total frames | 97 |
| Anchor 1 | `LoadImage(A)` → `LTXVPreprocess` → `LTXVAddGuide(frame_idx=0,  strength=1.0)` |
| Anchor 2 | `LoadImage(B)` → `LTXVPreprocess` → `LTXVAddGuide(frame_idx=48, strength=0.5)` |
| Anchor 3 | `LoadImage(C)` → `LTXVPreprocess` → `LTXVAddGuide(frame_idx=-1, strength=1.0)` |
| Camera LoRA | `dolly-in` @ str=0.5 (lower than usual to avoid fighting the anchors) |
| Speed LoRA | `distilled-lora-384-1.1` @ str=0.5 |
| Resolution | 768×512 (stage 1) → 1536×1024 (stage 2) |
| Sampler | distilled `ManualSigmas` (8 steps, see §9.7) + `CFGGuider cfg=1.0` |
| Prompt | *"the camera slowly dollies in toward the subject; subtle ambient room tone"* |

**Wiring sketch:**

```
CheckpointLoaderSimple ─▶ LoraLoaderModelOnly(distilled, 0.5) ─▶ LoraLoaderModelOnly(dolly-in, 0.5) ─▶ MODEL ─┐
                                                                                                              │
LTXAVTextEncoderLoader ─▶ CLIPTextEncode(pos)+(neg) ─▶ LTXVConditioning(fps=25) ─┐                            │
                                                                                  │                           │
LoadImage(A) ─▶ LTXVPreprocess ─▶ LTXVAddGuide(idx=0,  str=1.0) ──┐               │                           │
LoadImage(B) ─▶ LTXVPreprocess ─▶ LTXVAddGuide(idx=48, str=0.5) ──┼─▶ positive ───┤                           │
LoadImage(C) ─▶ LTXVPreprocess ─▶ LTXVAddGuide(idx=-1, str=1.0) ──┘               │                           │
                                                                                   │                          │
EmptyLTXVLatentVideo(768, 512, 97) ─┐                                              │                          │
LTXVEmptyLatentAudio(≥97)           ─┴─▶ LTXVConcatAVLatent ─▶ AV latent ──────────┴───▶ CFGGuider ◀──────────┘
                                                                                              │
                                                              ManualSigmas + KSamplerSelect ──▶ SamplerCustomAdvanced ─▶ ...
```

**Tuning if it looks wrong:**
- Middle anchor pops/snaps → lower `strength` from 0.5 to 0.3.
- Middle anchor drifts → raise to 0.7.
- Camera doesn't move → raise camera-LoRA strength to 0.7, or rephrase prompt to be more explicit about the move ("a clear, smooth dolly-in over the full duration").
- Camera fights the end anchor → drop camera LoRA to 0.3 or remove it; let the prompt + anchor positions imply the move.

### 10.9 Compatibility quick-reference

| Combination | Works? | Notes |
|-------------|--------|-------|
| Distilled LoRA + camera LoRA | Yes | Both via `LoraLoaderModelOnly`, chain in series |
| Camera LoRA + IC-LoRA | Caveat | Keep camera LoRA ≤0.5; IC-LoRA wants the trajectory |
| FLF2V + camera LoRA | Yes | Camera LoRA shapes interior trajectory; anchors win at boundaries |
| FMLF2V + camera LoRA | Untested | Test on short clips first |
| Multiple IC-LoRAs simultaneously | Untested | One IC-LoRA per chain in shipped templates; Union already merges depth+canny+pose |
| Camera LoRA + audio | Yes | Audio is independent of model-side LoRAs |
| `LTXVLoopingSampler` + camera LoRA | Yes | LoRA modifies MODEL, looping sampler uses MODEL — no conflict |
| `LTXVLoopingSampler` + IC-LoRA | Yes | `guiding_strength` knob controls IC-LoRA weight per tile |

---

## 11. Prompting Gemma for LTX-2.3 (vs. CLIP)

LTX-2.3's text encoder is **Gemma 3 12B Instruct**, not CLIP. Most online prompting advice (SD, SDXL, Flux, Midjourney) assumes CLIP and does not transfer. This section captures the differences.

### 11.1 What changes vs. CLIP conventions

| SD / CLIP convention | Gemma / LTX-2.3 reality |
|----------------------|--------------------------|
| Comma-separated tokens (`red dress, long hair, forest, night`) | Works, but **inferior** to prose. Gemma understands full sentences. |
| `(word:1.4)` emphasis syntax | **Not parsed as weighting.** Parentheses are read as punctuation. Emphasize via repetition + specificity + front-loading. |
| 77-token context limit | Not a concern. Paragraph-length prompts are fine. |
| Short prompts (`anime girl, blue eyes, sword`) | Underspecified — Gemma is designed for long context. |
| Long SD-style negative laundry list | Lightricks convention: ≤8 concept-level tokens (see §9.4). |
| Word order barely matters | Word order **matters**. First sentence anchors style; later sentences layer specifics. |

**Bottom line:** write like a cinematographer describing a shot to a crew, not like an SD power-user tagging an image.

### 11.2 The five-paragraph shot prompt

Gemma reads blank lines as semantic breaks. This structure outperforms a single wall of text with identical content:

```
¶1  STYLE + MEDIUM          (copy-paste across the entire project)
¶2  SUBJECT + FRAMING       (who/what, in what shot type)
¶3  ACTION + CAMERA         (what happens + how the camera behaves)
¶4  LIGHTING + ATMOSPHERE   (look of the scene)
¶5  SOUND                   (audio layers — see §9.11)
```

Paragraph 1 is your **style preamble**. Copy-paste it verbatim across every shot in a project. Rewriting it per shot is the single biggest cause of cross-shot style drift.

### 11.3 Emphasis without weights

Since `(word:1.4)` does nothing, the working equivalents are:

1. **Repetition across paragraphs.** Mention a critical visual element in two different paragraphs.
2. **Specificity.** `"a fitted cherry-red costume with black spider-web pattern stitched across the chest"` beats `(red costume:1.4)`.
3. **Front-loading.** Critical elements go in ¶2, not ¶5.
4. **Adjective density.** `"a tall, gaunt, deeply shadowed figure"` emphasizes more than `"a tall figure"`.

### 11.4 Motion verbs — the #1 fix for static output

Weak verbs produce ghostly/frozen output. Strong verbs produce motion. Every sentence in ¶3 should contain a concrete motion verb.

| Weak | Strong |
|------|--------|
| "The character is in the forest" | "The character walks through the forest" |
| "She looks sad" | "She lowers her gaze and exhales slowly" |
| "Rain falls" | "Heavy rain streaks diagonally across the frame" |
| "The car moves" | "The car accelerates hard, tires spinning, toward camera-right" |

Pacing verbs (`slowly`, `suddenly`, `gradually`, `smoothly`, `abruptly`) affect the tempo of generated motion.

### 11.5 Named colors beat hex codes

Gemma knows `"cherry red"`, `"crimson"`, `"oxblood"` — each with different visual associations. It does not know `#B22222`.

| Generic | Specific |
|---------|----------|
| "blue" | navy, cobalt, electric blue, periwinkle, teal, sky blue, denim |
| "red" | cherry red, crimson, scarlet, brick, oxblood, coral, rose |
| "green" | emerald, forest, sage, chartreuse, jade, olive |
| "yellow" | golden, mustard, cream, lemon, amber |

### 11.6 Gemma-specific capabilities worth knowing

Gemma is a full LLM; features CLIP lacks:

- **Negation in the positive prompt works**: `"no camera movement"`, `"without any text on screen"` — CLIP often inverts; Gemma respects.
- **Sequences**: `"first she stands still, then she turns toward the camera"` — a shot with two beats.
- **Counting**: `"three figures in the background"` — reliable up to ~5; degrades above.
- **Comparatives**: `"taller than the figure behind her"`, `"quieter than the previous shot"`.
- **Causality**: `"because of the heavy rain, the streets are empty"` — adds motivation the model can express visually.

### 11.7 Known weaknesses

- **Text rendering on screen** — fails; add text in post.
- **Precise counting above ~5** — `"seven candles"` often renders as 4 or 8.
- **Hands performing specific finger actions in CU**.
- **Brand-name items** (specific logos, specific car models) — drift badly.

### 11.8 When to extend the negative prompt

Start with Lightricks' default (`pc game, console game, video game, cartoon, childish, ugly`). Add concept-level terms **only** when you see a specific problem:

| Problem | Add |
|---------|-----|
| Too anime when you want American animation | `anime, manga, japanese animation` |
| Too 3D-rendered | `3D rendered, CGI, pixar, video game` |
| Too modern for a retro target | `modern, HDR, digital sharpness` |
| Too realistic for a stylized target | `photorealistic, photography, live action` |

**Do not** add SD-era tokens like `blurry, low quality, jpeg artifacts, extra fingers, bad hands, deformed`. Lightricks' training did not weight these, and they displace useful negative tokens. Cap total at ~8 concepts.

### 11.9 The iteration protocol — change one thing

1. **Lock the seed** in `RandomNoise`.
2. **Identify ONE thing that's wrong** (specific visual element, not a vibe).
3. **Edit ONE paragraph** (the one that owns that thing).
4. **Rerun at low res** (512×384, 49 frames → ~25s on a 4090).
5. **Compare to previous version.** Did the one thing change? Did others stay the same?
6. **Keep or revert.**

Changing prompt and seed simultaneously during iteration loses all signal.

---

## 12. Node Reference

This section is a compact lookup for the nodes that appear in §§2–10. Each table row gives the node's purpose, its main inputs/outputs, and the widgets you'll actually touch. For exhaustive per-node pages (especially edge-case widgets), follow the links at the end.

**How to read this section:**
- Widget names are lowercase (`cfg`, `sampler_name`) — they match the ComfyUI UI labels.
- "↑" means "raising this value →"; "↓" the opposite. Directional notes describe *typical* effects; a few nodes invert these near extreme values.
- Default / recommended values assume LTX-2.3 video work unless otherwise noted.

---

### 12.1 Core loader nodes

#### `CheckpointLoaderSimple`

**What it does.** Loads a combined checkpoint (`MODEL` + `CLIP` + `VAE`) from `models/checkpoints/` into VRAM.
**Why it's needed.** Most SD-family and packaged video models ship all three components in one `.safetensors` file — this node is how those weights enter the graph. Without it (or the separate-file alternatives below), there is no model to denoise with.
**Effect on video.** The *choice* of checkpoint is the single biggest factor in the output: it determines visual style, motion vocabulary, trained resolution, aspect-ratio sweet spot, and maximum frame count. The node itself has no output-shaping widgets — precision is baked into the file.
**Widgets.** `ckpt_name` — file to load.

#### `UNETLoader` / `DiffusionModelLoader`

**What it does.** Loads only the denoising backbone (UNet or DiT). VAE and text encoder must be loaded separately.
**Why it's needed.** Most modern video checkpoints (LTX-2.3 dev/fp8/nvfp4, HunyuanVideo, Wan) ship the backbone, VAE, and text encoder as *separate* files so you can mix precisions (e.g., FP8 backbone + BF16 VAE). A `CheckpointLoaderSimple` cannot read these split files.
**Effect on video.** `weight_dtype` trades VRAM against fidelity: `fp8_e4m3fn` / `fp8_e5m2` roughly halves VRAM vs. `default` (BF16), at the cost of subtle color shifts, mild banding, and occasional detail loss — usually imperceptible on short clips, more visible on long renders with smooth gradients.
**Widgets.** `unet_name`; `weight_dtype` — `default` / `fp8_e4m3fn` / `fp8_e5m2`.

#### `VAELoader`

**What it does.** Loads a standalone Variational Autoencoder — the component that translates between pixels and latent space.
**Why it's needed.** Video VAEs are **3D-aware** (they encode/decode along the temporal axis too). If you load an image-only VAE by mistake, or mix a VAE from a different model family, decoding produces black frames, colored noise, or violent flicker. Split-file checkpoints need this node because the VAE lives in its own `.safetensors`.
**Effect on video.** VAE choice controls final color fidelity, how cleanly fine detail decodes, and how much temporal flicker is present. Mismatched VAEs are a common cause of "the output looks broken but the sampler succeeded".
**Widgets.** `vae_name`.

#### `CLIPLoader` / `DualCLIPLoader`

**What it does.** Loads one or two CLIP / T5-XXL text encoders and exposes them as a `CLIP` input.
**Why it's needed.** SD-family, Wan, HunyuanVideo, CogVideoX, and Flux all use CLIP or CLIP+T5 to turn prompts into conditioning tensors. Without a matching encoder the prompt never reaches the model. **Do not use for LTX-2.3** — LTX's Gemma 3 encoder has a different tokenizer and hidden-state size, and `CLIPLoader` silently produces garbage conditioning (see §9.3 and `LTXAVTextEncoderLoader` in §12.5).
**Effect on video.** Tokenizer/encoder mismatch → gibberish output or wrong subject entirely. Matched correctly, the encoder choice mostly affects prompt adherence fidelity, not raw visual quality.
**Widgets.** `clip_name(_1/_2)`; `type` — must match the model family (`sdxl`, `flux`, `wan`, etc.).

#### `UnetLoaderGGUF` (ComfyUI-GGUF)

**What it does.** Loads GGUF-quantized backbones (Q2_K…Q8_0) — extreme-compression formats borrowed from llama.cpp.
**Why it's needed.** LTX-2.3 22B at BF16 is 46 GB — impossible on a 12–16 GB card. GGUF quantization shrinks that to 8–25 GB with manageable quality loss, putting the model on consumer hardware that otherwise couldn't run it at all.
**Effect on video.** ↓ quant level (Q2_K, Q3_K) → drastically lower VRAM, visibly softer/smudged detail and mild color shifts; Q8_0 is nearly lossless vs. BF16 but still ~60% the size. For character-consistency-critical shots, use Q6_K or higher.
**Widgets.** `unet_name` — the `.gguf` file. Quant level is baked into the file.

#### `DualCLIPLoaderGGUF` (ComfyUI-GGUF)

**What it does.** Loads GGUF-quantized text encoders, including Gemma 3 for LTX-2.3 via `type="ltxv"`.
**Why it's needed.** Gemma 3 12B FP8 is 13 GB; GGUF-quantized it drops to 4–8 GB, which matters when running LTX-2.3 at ≤16 GB total VRAM. This node is the GGUF equivalent of `LTXAVTextEncoderLoader` — the generic GGUF CLIP loader will not load Gemma correctly without `type="ltxv"`.
**Effect on video.** Low-bit Gemma (Q3/Q4) → prompt adherence degrades slightly (subtle elements of long prompts get dropped); visual output is otherwise unchanged because the backbone still denoises at full precision.
**Widgets.** `clip_name_1/2`; `type` — must be `ltxv` for LTX-2.3.

#### `LoraLoader`

**What it does.** Applies a LoRA (Low-Rank Adaptation) to both `MODEL` and `CLIP`, injecting style / subject / motion biases without replacing the base weights.
**Why it's needed.** LoRAs are the standard way to specialize a model — style, specific characters, motion patterns, camera behavior — without retraining or swapping checkpoints. Stacking lets you combine a style LoRA with a motion LoRA in the same render.
**Effect on video.** `strength_model ↑` → stronger stylistic pull on the image/motion; past ~1.2, you get overfitting artifacts (warped anatomy, color saturation, repeating textures). `strength_clip` controls how strongly the LoRA influences prompt interpretation — usually match `strength_model`, but drop `strength_clip` to 0 if the LoRA distorts prompts.
**Widgets.** `lora_name`; `strength_model` (0–1+); `strength_clip` (0–1+).

#### `LoraLoaderModelOnly`

**What it does.** Applies a LoRA only to `MODEL` (not CLIP). This is the LoRA loader for **LTX-2.3** and other models where the text encoder sits outside the LoRA graph.
**Why it's needed.** LTX-2.3's Gemma encoder is not part of the LoRA-patchable weights — the LoRAs were trained against the DiT backbone only. Feeding a Gemma-less LoRA through `LoraLoader` (which also patches CLIP) silently breaks prompt behavior. All LTX LoRAs — distilled-speed, camera-control, style — must go through this node.
**Effect on video.** Identical strength behavior to `LoraLoader` but applies only to motion / visual / style biases. See §10.1 for per-category strength recommendations (distilled: 0.2–0.5; camera: 0.5–0.8; style: ≤0.6).
**Widgets.** `lora_name`; `strength_model`.

---

### 12.2 Text / image conditioning nodes

#### `CLIPTextEncode`

**What it does.** Encodes a prompt string into a `CONDITIONING` tensor the sampler can consume.
**Why it's needed.** The model denoises in response to a conditioning tensor, not raw text — this node is how your prompt actually reaches the sampler. You use it twice per workflow: once for the positive prompt (what you want) and once for the negative (what to avoid).
**Effect on video.** Prompt quality is the main driver of output content, subject, composition, and motion. On CLIP, emphasis syntax `(word:1.4)` boosts a token's weight; on Gemma (LTX-2.3) this syntax is ignored (§11.1) — emphasize by repetition, specificity, and paragraph order instead.
**Widgets.** `text` — the prompt string.

#### `LoadImage`

**What it does.** Loads an image from `input/` into the graph as an `IMAGE` tensor (and optionally a `MASK`).
**Why it's needed.** Every image-to-video, keyframe, or IC-LoRA workflow starts with this node — it's the entry point for reference stills. For FLF2V you'll have two or three `LoadImage` nodes, one per anchor.
**Effect on video.** The input image's resolution, aspect ratio, and content anchor what the model generates. Feeding an image whose aspect ratio doesn't match the latent target → distortion at encode time. Always resize to the model's trained resolution before encoding (see `ResizeImageMaskNode` in §12.8).
**Widgets.** `image` — file picker + `upload` button. Optional `mask` output.

#### `VAEEncode`

**What it does.** Encodes a pixel-space image into a latent tensor using the VAE — the inverse of `VAEDecode`.
**Why it's needed.** Samplers operate in latent space, not pixel space. Any reference image that needs to be *blended into the latent* (hard anchors, first-frame injection, inpainting) must first be encoded here. I2V workflows that only pass the image as *conditioning* (`LTXVImgToVideoConditionOnly`) skip this.
**Effect on video.** Encoding must use the **matching** VAE — a VAE from a different model family silently produces wrong latents and the sampler outputs noise or black. No shaping widgets.
**Widgets.** `pixels`, `vae` — inputs only.

#### `VAEDecode`

**What it does.** Decodes a denoised latent back into pixel-space frames — the last step before you have viewable images.
**Why it's needed.** The sampler's output is a compressed latent; you can't play or export it directly. This is the bridge from latent to RGB frames.
**Effect on video.** Decoding is where VAE mismatches reveal themselves (flicker, color banding, black frames). For long or HD video, this node's VRAM spike can OOM even when sampling succeeded — switch to `VAEDecodeTiled` / `LTXVTiledVAEDecode` in that case.
**Widgets.** `samples`, `vae` — inputs only.

#### `VAEDecodeTiled`

**What it does.** Same as `VAEDecode` but processes the latent in **spatial and temporal tiles**, keeping only one tile in VRAM at a time.
**Why it's needed.** Video VAE decoding is the most memory-intensive stage of the pipeline. A 1280×720×97-frame latent can blow past 24 GB on plain `VAEDecode`. Tiled decoding trades time for VRAM — essential on 12–16 GB cards.
**Effect on video.** Tiled decode can leave faint **seam artifacts** at tile boundaries. `overlap ↑` → softer seams, more VRAM/time. `tile_size ↓` → lower VRAM, slower, more visible seams. `temporal_size` controls how many frames decode per temporal tile — match to a multiple of the VAE's temporal stride to avoid flicker at tile edges. See §9.8 for LTX-safe values.
**Widgets.** `tile_size` (px, spatial tile), `overlap` (px, spatial overlap), `temporal_size` (frames per tile), `temporal_overlap` (frames).

#### `CLIPVisionEncode`

**What it does.** Passes an image through a CLIP Vision encoder to extract a **semantic embedding** — a high-level description of what the image contains — rather than a pixel-level latent.
**Why it's needed.** For IP-Adapter / style-transfer workflows where you want the model to pick up *content and style* from a reference without locking the exact pixels in as frame 0. Think "make the video in the same mood as this photo" rather than "start the video with this photo".
**Effect on video.** Output is a conditioning embedding, not a latent. It biases subject appearance, palette, and composition across all frames — but does not force any specific first frame.
**Widgets.** `clip_vision`, `image` — no widgets.

---

### 12.3 Latent and sampling nodes

#### `EmptyLatentVideo`

**What it does.** Creates an empty latent "canvas" shaped `[batch, channels, frames, height, width]` — the blank tensor the sampler will fill.
**Why it's needed.** The sampler needs a latent to denoise; this defines its dimensions. Used by Wan, HunyuanVideo, CogVideoX. **Not** for LTX-2.3 — use `EmptyLTXVLatentVideo` + audio pair (§12.5) because LTX expects an AV-concatenated latent.
**Effect on video.** `width`/`height` = output resolution; `length` = frame count. Each model has stride rules (multiples of 8, 16, 32) and a max trained frame count — breaking them causes shape errors or artifacts. Doubling any dim roughly quadruples sampling VRAM.
**Widgets.** `width`, `height`, `length` (frames), `batch_size`.

#### `KSampler`

**What it does.** The standard ComfyUI denoising loop — takes a latent full of noise and iteratively cleans it toward the conditioning target.
**Why it's needed.** This is the engine that actually generates video. Works with any model that plugs into ComfyUI's standard interface (most SD-family and older video models).
**Effect on video.** `steps ↑` → more refinement, higher quality, linearly slower; diminishing returns past ~30 for most video models. `cfg ↑` → stronger prompt adherence, risk of oversaturation/warping past model's sweet spot (video models usually 1–7). `denoise < 1.0` keeps some of the input latent's signal — this is how I2V and refine passes work; at 1.0 the sampler ignores the input latent entirely. `sampler_name` changes stepping algorithm; `scheduler` changes the sigma curve (Euler + normal is a safe default).
**Widgets.** `seed`, `steps`, `cfg`, `sampler_name`, `scheduler`, `denoise`.

#### `KSamplerAdvanced`

**What it does.** `KSampler` with extra step-range controls — start and end the denoise at arbitrary steps, optionally leave leftover noise for a second pass.
**Why it's needed.** Two-pass refine (low-res → upscale latent → finish denoising), hi-res fix, noise injection mid-denoise — all require starting/stopping the sampler at specific steps and handing the partial latent to another sampler.
**Effect on video.** `start_at_step` / `end_at_step` split the denoise curve into stages — useful for stage-1/stage-2 pipelines without switching to `SamplerCustomAdvanced`. `return_with_leftover_noise=True` leaves the latent "unfinished" for the next pass to continue.
**Widgets.** All `KSampler` widgets plus `start_at_step`, `end_at_step`, `add_noise`, `return_with_leftover_noise`.

#### `SamplerCustomAdvanced`

**What it does.** A **modular** sampler — instead of bundled widgets, you wire in the pieces separately: `NOISE` source, `GUIDER` (model+conditioning), `SAMPLER` (algorithm), `SIGMAS` (schedule), and the `LATENT`.
**Why it's needed.** LTX-2.3 needs non-standard combinations (custom guiders, `ManualSigmas`, AV-concatenated latents) that don't fit inside the monolithic `KSampler`. All shipped LTX-2.3 workflows use this.
**Effect on video.** None from the node itself — everything is delegated to the inputs. The advantage is that each piece can be swapped independently: try `ManualSigmas` vs `LTXVScheduler` without touching anything else.
**Widgets.** None — wiring is the configuration.

#### `RandomNoise`

**What it does.** Emits a noise tensor seeded by `noise_seed`. Feeds the `NOISE` input of `SamplerCustomAdvanced`.
**Why it's needed.** Denoising starts from pure noise; this is where that noise comes from. The seed makes the render **reproducible** — same seed + same graph = same output.
**Effect on video.** Changing the seed changes everything that isn't pinned by the prompt or anchors. **Lock the seed** during prompt iteration (§11.9); changing prompt and seed simultaneously gives no signal about which change caused the difference.
**Widgets.** `noise_seed` — integer.

#### `KSamplerSelect`

**What it does.** Picks the stepping algorithm and outputs it as a `SAMPLER` object for `SamplerCustomAdvanced`.
**Why it's needed.** The algorithm determines how the sampler walks from noise to clean latent. Different algorithms converge differently — some are stable but plain, others creative but flickery.
**Effect on video.** For video, temporal consistency (no flicker) matters as much as image quality. `euler` and `dpmpp_2m` are the safest defaults — smooth motion, predictable. `euler_ancestral` injects extra noise per step → more creative, more flicker. `uni_pc` / `deis` are fast but sometimes unstable on video. LTX-2.3 templates all ship with `euler`.
**Widgets.** `sampler_name`.

#### `BasicScheduler`

**What it does.** Generates a sigma schedule — the noise-level curve the sampler walks down.
**Why it's needed.** Many models expect a specific schedule shape. Wrong schedule → over- or under-denoised output.
**Effect on video.** `karras` → front-loads detail (sharper). `sgm_uniform` → cleaner convergence for SD-family. `linear_quadratic` → matches flow-matching models more closely than `normal`. For LTX-2.3 use `LTXVScheduler` instead — it's tuned for the model's shifted flow-matching schedule.
**Widgets.** `scheduler`, `steps`, `denoise`.

#### `CFGGuider`

**What it does.** Wraps model + positive + negative conditioning into a `GUIDER` with a single `cfg` scalar — classifier-free guidance strength.
**Why it's needed.** `SamplerCustomAdvanced` needs a `GUIDER` input; `CFGGuider` is the simplest one. It applies standard CFG: bigger `cfg` → sampler pushes harder toward positive conditioning and away from negative.
**Effect on video.** `cfg = 1.0` → no classifier-free guidance (no difference between positive and negative) — this is what **distilled** LTX-2.3 uses, because the distilled LoRA was trained to work without CFG. `cfg > 1.0` → stronger prompt adherence, risk of oversaturation / contrast crushing (especially on flow-matching models). For LTX-2.3 AV generation use `MultimodalGuider` instead (§12.6).
**Widgets.** `cfg`.

---

### 12.4 Output / video assembly nodes

#### `SaveImage`

**What it does.** Writes individual frames to `output/` as numbered PNGs.
**Why it's needed.** For external editing workflows — export frames, bring into DaVinci / After Effects / FFmpeg, finish there. Also useful for debugging a single frame from a long render.
**Effect on video.** None on content; affects only what reaches disk. Frames are saved lossless (PNG).
**Widgets.** `filename_prefix`.

#### `VHS_VideoCombine` (VideoHelperSuite)

**What it does.** Assembles a batch of frames (+ optional audio) into a playable video file — MP4, WEBM, GIF, or WebP.
**Why it's needed.** The model outputs a batch of still images; this is how you turn them into a video your OS / browser / player actually plays. Most-used export node in the ecosystem.
**Effect on video.** `frame_rate` must match `LTXVConditioning.frame_rate` — mismatched fps makes motion look sped-up or slowed-down relative to what the model generated. `format` chooses container/codec (H.264 MP4 = most compatible; H.265 = smaller file, less support; GIF = quick previews, limited colors). `pix_fmt` — use `yuv420p` for maximum compatibility (some players reject 10-bit / 4:4:4). `crf ↓` → higher quality, larger file; 17–23 is a good range. `audio` input is where LTX-2.3's `AUDIO` output connects.
**Widgets.** `frame_rate`, `format`, `pix_fmt`, `crf`, `save_metadata`, `audio` (optional).

#### `CreateVideo`

**What it does.** ComfyUI's newer built-in alternative to `VHS_VideoCombine` — same purpose, fewer widgets.
**Why it's needed.** Works without installing VideoHelperSuite; fine for simple MP4 exports. VHS is still more flexible (more codec/format options, richer audio handling).
**Effect on video.** Same parameters govern playback speed and file size. Less granular control over codec/container specifics.
**Widgets.** Simpler subset of `VHS_VideoCombine`.

#### `VHS_LoadVideo` (VideoHelperSuite)

**What it does.** Reads a video file and outputs an `IMAGE` batch + `AUDIO`. The inverse of `VHS_VideoCombine`.
**Why it's needed.** Video-to-video workflows, IC-LoRA guide videos (depth/Canny/pose references), extension workflows that need to pull the last frames of a prior clip.
**Effect on video.** `force_rate` resamples to a target fps during load — use to match your generation fps. `force_size` resizes frames. `frame_load_cap` limits how many frames load (memory saver). `skip_first_frames` + `select_every_nth` let you sample a long video sparsely — useful when feeding guide videos to IC-LoRAs.
**Widgets.** `video`, `force_rate`, `force_size`, `frame_load_cap`, `skip_first_frames`, `select_every_nth`.

#### `GetImageRangeFromBatch` (KJNodes)

**What it does.** Slices a contiguous range of frames out of an `IMAGE` batch.
**Why it's needed.** Video extension (§10.5): grab the last N frames of clip A to feed as anchors for clip B. Also for discarding intro/outro frames before export.
**Effect on video.** No quality impact; controls which frames survive. `start_index = -N` counts N frames from the end — the canonical way to grab a "tail" for stitching.
**Widgets.** `start_index`, `num_frames`.

#### `GetVideoComponents`

**What it does.** Splits a loaded video into separate `IMAGE` batch, `AUDIO` stream, and `fps` number.
**Why it's needed.** Downstream nodes (IC-LoRA preprocessors, audio passthrough) usually want these separately. Reading them from `VHS_LoadVideo` and then splitting via this node is the standard pattern.
**Effect on video.** None — pure re-routing.
**Widgets.** None.

---

### 12.5 LTX-2.3 loaders and conditioning

All of these require `ComfyUI-LTXVideo`.

> **Authoritative reference for §§12.5–12.8.** For the full, up-to-date widget list on every LTX-2.3 node (including widgets omitted here for brevity), see the official repo README and the reference workflows:
> - [`github.com/Lightricks/ComfyUI-LTXVideo`](https://github.com/Lightricks/ComfyUI-LTXVideo) — node pack README.
> - `custom_nodes/ComfyUI-LTXVideo/example_workflows/2.3/` — shipped JSONs (`Two_Stage_Distilled`, `Single_Stage_Distilled_Full`, `ICLoRA_Union_Control`, `ICLoRA_Motion_Track`). These are the ground truth for which widgets are wired vs. left at defaults.
> - [`huggingface.co/Lightricks/LTX-2.3`](https://huggingface.co/Lightricks/LTX-2.3) — model card with conditioning / scheduler notes.

#### `LTXAVTextEncoderLoader`

**What it does.** Loads Gemma 3 12B Instruct — LTX-2.3's AV-aware text encoder — and exposes it as a `CLIP` input.
**Why it's needed.** Gemma 3 is a full 12B decoder LLM with its own tokenizer, projection, and hidden-state dimensions. The generic `CLIPLoader` / `DualCLIPLoader` can't correctly load it — it silently fails or produces shape errors at the DiT. **Required** instead of `CLIPLoader` for LTX-2.3. Always load a single consolidated safetensors; never Git-LFS pointer files (tokenizer assets end up missing — §9.3).
**Effect on video.** Wrong loader → gibberish video or empty output. Correct loader → prompt adherence is excellent even for paragraph-length prompts (Gemma handles long context well).
**Widgets.** `text_encoder_name` — single consolidated safetensors from `models/text_encoders/`.

#### `GemmaAPITextEncode`

**What it does.** Sends the prompt to Lightricks' API, returns the encoded conditioning — no local Gemma needed.
**Why it's needed.** Gemma 3 12B occupies 9.5–13 GB of VRAM locally. On ≤12 GB cards, encoding Gemma locally crowds out the backbone; this node moves that cost to Lightricks' servers.
**Effect on video.** Identical conditioning quality to local Gemma. Downsides: requires network connectivity, is not reproducible offline, and adds ~1–3s latency per prompt.
**Widgets.** `api_key` (if required), `text`.

#### `LTXVSaveConditioning`

**What it does.** Serializes an encoded `CONDITIONING` tensor to `models/embeddings/` as a `.safetensors` file.
**Why it's needed.** Gemma encoding is expensive (multi-second, gigabytes of VRAM). If you iterate on the **same prompt** across many seeds / resolutions / LoRA strengths, you encode Gemma once and reuse the cached result — saving minutes of compute per iteration session.
**Effect on video.** None on quality. Massive speedup on iterative workflows.
**Widgets.** `filename_prefix`.

#### `LTXVLoadConditioning`

**What it does.** Reloads a previously-saved conditioning tensor — the inverse of `LTXVSaveConditioning`.
**Why it's needed.** Lets you skip `LTXAVTextEncoderLoader` + `CLIPTextEncode` entirely on reruns. Drops Gemma from VRAM on the second+ render, freeing memory for higher resolution / more frames.
**Effect on video.** None on quality; frees Gemma's VRAM cost.
**Widgets.** `conditioning_name`.

#### `EmptyLTXVLatentVideo`

**What it does.** Creates an empty video latent **shaped for LTX-2.3's AV-aware DiT** — not the same shape as `EmptyLatentVideo`.
**Why it's needed.** LTX-2.3 denoises video and audio together, so the video latent must have the exact stride and channel layout the DiT expects. Feeding a generic `EmptyLatentVideo` here → shape-mismatch error at the sampler ("4 vs 3", etc.).
**Effect on video.** Defines the output resolution and frame count. Must obey LTX's constraints: `width % 32 == 0`, `height % 32 == 0`, `length = 8n + 1`. Breaking these → sampler error. Higher resolution and more frames scale VRAM and time roughly linearly per dimension.
**Widgets.** `width`, `height`, `length`, `batch_size` (keep at 1).

#### `LTXVEmptyLatentAudio`

**What it does.** Creates an empty audio latent that pairs with the video latent.
**Why it's needed.** LTX-2.3 always samples video + audio together — even for silent output. If audio latent is missing or **shorter** than the video latent, sampling fails.
**Effect on video.** For silent output, use defaults and leave the prompt without sound cues — the model will generate ambient quiet. For driven audio (e.g., when you want audio to match a specific duration), `length` must be **≥** video latent length.
**Widgets.** `length`.

#### `LTXVConcatAVLatent`

**What it does.** Concatenates the video and audio latents along the modality axis into the single tensor the DiT ingests.
**Why it's needed.** The DiT has one latent input with both modalities stacked. This node builds that tensor. Without it → wrong input shape → sampler error.
**Effect on video.** None on content; pure plumbing. Must be present in every LTX-2.3 sampling graph.
**Widgets.** None.

#### `LTXVSeparateAVLatent`

**What it does.** Splits the sampler's output latent back into separate video and audio latents — inverse of `LTXVConcatAVLatent`.
**Why it's needed.** The decoders are modality-specific: `LTXVTiledVAEDecode` wants the video latent, `LTXVAudioVAEDecode` wants the audio latent. You can't feed the concatenated tensor to either.
**Effect on video.** None on content; routes the two streams to their respective decoders.
**Widgets.** None.

#### `LTXVConditioning`

**What it does.** Stamps frame-rate metadata onto the conditioning tensor so the model knows at what fps to generate motion.
**Why it's needed.** LTX-2.3 was trained with per-clip fps information — it generates different motion pacing at 24 vs 25 vs 30 fps. Without this node, the model defaults to something that may not match your export fps.
**Effect on video.** `frame_rate` controls the **tempo** of motion the model generates. **Must match** `VHS_VideoCombine.fps` — a mismatch makes motion look too fast or too slow on playback. 25 is the shipped default; 24 for cinematic, 30 for smoother motion.
**Widgets.** `frame_rate`.

---

### 12.6 LTX-2.3 schedulers, guiders, samplers

#### `LTXVScheduler`

**What it does.** Emits the sigma schedule (noise-level curve the sampler walks down) tuned specifically for LTX-2.3's flow-matching DiT.
**Why it's needed.** LTX-2.3 is trained on a *shifted* flow-matching schedule — not the DDPM-style schedules `BasicScheduler` produces. Feeding a generic schedule → model denoises at the wrong rate → blurry, undercooked, oversaturated, or broken frames. Used in both stages of the two-stage workflow (§9.7).
**Effect on video.** `max_shift ↑` → curve pushes toward noisier start → stronger global motion, looser spatial structure. `base_shift ↑` → curve tilts to noisier overall (more chaotic detail). `stretch=True` lets the schedule terminate at `terminal` instead of ~0 — this is what leaves room for stage 2. `terminal ↑` → more "undone" noise left for a stage-2 refine pass; `↓` → cleaner standalone render but no stage-2 headroom.
**Widgets.** `steps` (20 stage-1, 3–4 stage-2), `max_shift` (2.05), `base_shift` (0.95), `stretch` (True), `terminal` (0.1). Full table in §9.7.

#### `ManualSigmas`

**What it does.** Hard-codes a fixed list of sigmas — a literal noise schedule — instead of generating one algorithmically.
**Why it's needed.** The **single-stage distilled** LTX-2.3 workflow was trained against a *specific* 8-point sigma schedule. The distilled LoRA expects exactly those values — using `LTXVScheduler` instead produces the wrong noise trajectory, and the distilled LoRA fights the sampler, giving muddy / broken output. `ManualSigmas` pins the schedule to the exact values the LoRA was trained with. **Only used in single-stage distilled mode;** the two-stage distilled workflow keeps `LTXVScheduler` in both stages.
**Effect on video.** With the shipped values (`1.0, 0.99375, 0.9875, 0.98125, 0.975, 0.909375, 0.725, 0.421875, 0.0`), you get the canonical 8-step distilled render — fast (~8s on a 4090) and clean. Shorter sigma lists → even faster but coarser motion and softer detail. Longer lists → diminishing returns; the distilled LoRA saturates above ~12 steps.
**Widgets.** `sigmas` — comma-separated floats.

#### `MultimodalGuider`

**What it does.** Wraps model + positive + negative conditioning into a `GUIDER` with **separate guidance scales** for video and audio, plus STG (Spatio-Temporal Guidance).
**Why it's needed.** LTX-2.3 denoises both modalities in one pass, but the optimal guidance differs per modality. A plain `CFGGuider` applies one scalar to both — either under-guiding audio or over-cooking video. **Required** for any AV generation in LTX-2.3.
**Effect on video.** `scale` is the main "CFG-equivalent" knob — the shipped template uses 28; ↑ past ~32 → oversaturation, contrast crushing, warping. `skip_block_indices` selects which DiT layers STG perturbs (STG works by slightly corrupting internal layers to force the model to denoise more coherently). `stg_scale ↑` → stronger temporal coherence (less flicker) but less motion variety. `rescale` counteracts latent energy drift at high scales.
**Widgets.** `scale`, `skip_block_indices`, `stg_scale`, `rescale`.

#### `STGGuiderAdvanced`

**What it does.** An extended `MultimodalGuider` with finer STG control, **required** by `LTXVLoopingSampler`.
**Why it's needed.** The looping sampler walks the latent in temporal tiles and needs STG parameters exposed at a more granular level than the plain guider. Feeding a plain `MultimodalGuider` → type/interface errors.
**Effect on video.** Same knobs as `MultimodalGuider` plus advanced STG levers — these primarily affect **seam quality between temporal tiles** (how well neighboring tiles stitch together without visible flicker).
**Widgets.** Superset of `MultimodalGuider`.

#### `LTXVNormalizingSampler`

**What it does.** A sampler *wrapper* — takes another `SAMPLER` as input and normalizes the latent between denoise steps to prevent intensity overbake.
**Why it's needed.** At high CFG / `scale` values, LTX-2.3's latent accumulates magnitude across denoise steps, and the final output looks crushed — oversaturated, halos around highlights, black-point clipping. Normalization bleeds that accumulated intensity back on each step, keeping colors and contrast natural.
**Effect on video.** On stage 1 → more natural color and contrast, especially at `scale ≥ 28`. **Do not use** on inpaint / extend workflows — the normalization interferes with the continuity constraint between adjacent clips and creates visible seams.
**Widgets.** `sampler` (input).

#### `LTXVLoopingSampler`

**What it does.** Generates long video by sampling it in **temporal tiles** — chunks of frames that overlap and stitch together — instead of denoising the whole clip at once.
**Why it's needed.** VRAM scales roughly linearly with frame count. Past ~10s at 25 fps (~250 frames) single-shot denoising OOMs on consumer GPUs. Temporal tiling keeps peak VRAM near a single tile's worth, enabling 10–30 second clips on 24 GB cards.
**Effect on video.** `temporal_tile_size ↑` → fewer seams, higher VRAM and per-tile cost. `temporal_overlap ↑` → smoother transitions between tiles, slower (more redundant denoising). `temporal_overlap_cond_strength ↑ (→ 0.7)` → stronger continuity (reduces motion drift across tiles); `↓ (→ 0.3)` → more variation tile-to-tile (useful for shots that legitimately evolve). `guiding_strength` → how strongly IC-LoRA guides each tile. `horizontal_tiles` / `vertical_tiles > 1` enable spatial tiling for very high resolution.
**Widgets.** `temporal_tile_size` (80), `temporal_overlap` (24), `temporal_overlap_cond_strength` (0.5), `guiding_strength` (1.0), `cond_image_strength` (1.0), `horizontal_tiles` / `vertical_tiles` (1), `spatial_overlap` (1).

---

### 12.7 LTX-2.3 image / keyframe conditioning

#### `LTXVPreprocess`

**What it does.** Applies a deliberate degradation (compression + subtle noise) to the input image to match the compression statistics of LTX-2.3's training data.
**Why it's needed.** LTX-2.3 was trained on *compressed* frames (from real video codecs). Feeding a pristine sharp image confuses the model — it sees a distribution it never trained on. **Skipping this node produces near-static or ghostly I2V output** — one of the most common LTX-2.3 bugs.
**Effect on video.** With preprocessing → the model happily animates the input image. Without it → the input sits frozen on screen for most of the clip, or the video becomes ghostly/blurry. No knobs to tune.
**Widgets.** None.

#### `LTXVImgToVideoConditionOnly`

**What it does.** Injects a preprocessed image into the conditioning as a **soft first-frame anchor**. The model is told "start here" but is free to negotiate the exact first frame.
**Why it's needed.** The default I2V mode — simpler than `LTXVImgToVideoInplace`, faster to set up, more forgiving when the input image doesn't exactly match target resolution.
**Effect on video.** First frame closely resembles the input but isn't pixel-identical. Motion is fluid because the model isn't locked in. Best for "animate this photo" use cases.
**Widgets.** `conditioning`, `image`, `vae` — no scalar widgets.

#### `LTXVImgToVideoInplace`

**What it does.** VAE-encodes the image and **blends it directly into the first frame of the latent** — a hard anchor. The sampler is forced to reconstruct exactly that input.
**Why it's needed.** When you need the first frame to match pixel-for-pixel (e.g., matching a previous clip's last frame for extension, or locking in a specific portrait).
**Effect on video.** `strength = 1.0` → first frame matches the input exactly; motion fluidity may suffer because the model has less freedom. `strength ↓` → more drift allowed → smoother motion but first frame may subtly differ from input. For extension workflows (stitching clips), use 1.0.
**Widgets.** `strength`.

#### `LTXVAddGuide`

**What it does.** Places a **soft anchor image** at an arbitrary frame index in the clip. The building block for multi-keyframe I2V (FLF2V, FMLF2V, arbitrary keyframes).
**Why it's needed.** There is no dedicated first-last-frame node in LTX-2.3; you compose it from two `LTXVAddGuide` calls (`frame_idx=0` + `frame_idx=-1`). For three or more anchors, chain additional calls.
**Effect on video.** `frame_idx` picks where the anchor lands (`0` = first, `-1` = last, any integer = specific frame). `strength = 1.0` → anchor is rigid (use for boundary frames); `strength ≈ 0.5` → soft anchor (use for **interior** frames, which need wiggle room to avoid visible snaps). Too-high interior strength → "snap" pop at that frame. Too-low → middle anchor drifts / is ignored. Start interior at 0.5 (§10.4).
**Widgets.** `frame_idx`, `strength`.

#### `LTXVAddGuideMulti` (Kijai fork)

**What it does.** Same behavior as `LTXVAddGuide` but accepts a **list of anchors** `(frame_index, image, strength)` in one node instead of chaining N copies.
**Why it's needed.** Cleaner graphs when you have 3+ anchor frames. Functionally identical to chaining `LTXVAddGuide` — just less spaghetti.
**Effect on video.** Identical to chained `LTXVAddGuide`. Lives in Kijai's fork, not the official Lightricks repo.
**Widgets.** One row per anchor.

#### `LTXVLatentUpsampler`

**What it does.** Upsamples the stage-1 video latent by 2× using a trained spatial upscaler model (e.g., `ltx-2.3-spatial-upscaler-x2-1.1`).
**Why it's needed.** Two-stage rendering — generate at half resolution for speed/VRAM, then upsample and refine at full resolution. Runs in **latent space**, not pixels, so it understands video structure rather than just interpolating pixels.
**Effect on video.** Doubles spatial resolution (768×512 → 1536×1024). Because the stage-1 latent wasn't fully denoised (`LTXVScheduler.terminal=0.1`), the upsampled latent has headroom for a stage-2 refinement pass to add true detail — **not** just blurry upscaling.
**Widgets.** `upscaler` (model input), `scale` (2×).

#### `LatentUpscaleModelLoader`

**What it does.** Loads the spatial or temporal latent upscaler checkpoint from `models/latent_upscale_models/`.
**Why it's needed.** `LTXVLatentUpsampler` needs a loaded upscaler model as input. Spatial upscaler doubles resolution; temporal upscaler doubles frame count (optional, for very smooth output).
**Effect on video.** Model choice determines upscale quality. Use the matching version (LTX-2.3 upscalers with LTX-2.3 backbone).
**Widgets.** `model_name`.

#### `LTXVCropGuides`

**What it does.** After `LTXVLatentUpsampler`, trims the IC-LoRA / conditioning guide latents back to match the upsampled spatial/frame size.
**Why it's needed.** IC-LoRAs are trained at `ref_downscale=0.5` — the guide latent is at half the resolution of the output. After stage-2 upsampling, guides and the main latent disagree on size, which breaks the attention masks. This node auto-aligns them.
**Effect on video.** Essential if you use IC-LoRAs with two-stage rendering — without it, structural guidance (depth/Canny/pose) stops tracking correctly in stage 2. No effect (and safe to omit) when not using IC-LoRAs.
**Widgets.** None (reads shapes automatically).

#### `LTXVTiledVAEDecode`

**What it does.** LTX-specific tiled VAE decode — like the generic `VAEDecodeTiled` but tuned to LTX-2.3's VAE. Widgets are in **latent tiles** (not pixels).
**Why it's needed.** LTX-2.3's VAE decode is VRAM-hungry at full resolution. This node tiles along latent space — fewer pixel-level artifacts than naive pixel-tiled decode, and aware of LTX's temporal stride.
**Effect on video.** Smaller `tile_height` / `tile_width` / `overlap` → lower VRAM, slower, more visible seams if overlap is too low. `decode_method="auto"` picks the best strategy for the loaded VAE. See §9.8 for safe 12 GB values and the shipped two-stage defaults.
**Widgets.** `tile_height`, `tile_width`, `overlap`, `force_input_latent`, `decode_method` (`auto`), `upscale_method` (`auto`).

#### `LTXVAudioVAEDecode`

**What it does.** Decodes the audio latent into a playable `AUDIO` stream.
**Why it's needed.** LTX-2.3's audio is stored as latent features alongside the video latent; this node reconstructs waveform audio. Feeds `VHS_VideoCombine`'s `audio` input.
**Effect on video.** None on video; produces the audio track. Audio quality is model-fixed (ambient/cinematic; no lyrics or musical keys — §9.11).
**Widgets.** None.

---

### 12.8 LTX-2.3 IC-LoRA and structural control

#### `LTXICLoRALoaderModelOnly`

**What it does.** Loads an IC-LoRA ("in-context" LoRA) and exposes its `ref_downscale_factor` as a second output alongside the patched `MODEL`.
**Why it's needed.** IC-LoRAs were trained with a specific reference downscale factor (encoded in the filename as `ref0.5`). The `LTXAddVideoICLoRAGuide` node needs that factor to resize the guide video correctly. Using a plain `LoraLoaderModelOnly` → the factor isn't extracted → guide mismatch → weak or broken structural control.
**Effect on video.** `strength_model ↑` → stronger structural lock (depth/pose/edges constrain generation more rigidly); `↓` → more creative freedom but weaker adherence to the guide. Typical: 1.0 for Union-Control, tune down to 0.7–0.8 when combining with another style LoRA.
**Widgets.** `lora_name`, `strength_model`.

#### `LTXAddVideoICLoRAGuide`

**What it does.** Attaches a **preprocessed guide video** (depth maps / Canny edges / pose skeletons / motion-track lines) to the conditioning tensor. Every frame of the guide video constrains the corresponding generated frame.
**Why it's needed.** This is what turns an IC-LoRA into actual motion / structure control. Without it, the IC-LoRA is loaded but has no reference to guide against — it produces essentially normal output.
**Effect on video.** The guide video determines *where things go* — depth controls spatial layout per frame, Canny controls edge structure, pose controls body skeleton, motion-track controls camera trajectory. The model fills in texture, style, and detail from the text prompt, but composition and motion track the guide.
**Widgets.** `positive` / `negative` (conditioning), `vae`, `guide_video`, `downscale_factor`.

#### `LTX Add Video IC-LoRA Guide Advanced`

**What it does.** Same behavior as the basic guide node, plus **attention strength** and an optional **attention mask** for spatially-localized control.
**Why it's needed.** Sometimes you only want IC-LoRA control in part of the frame — e.g., depth constraint only on the foreground subject, not the sky. The mask lets you restrict where the guide applies.
**Effect on video.** `attention_strength ↑` → stronger guide influence within the mask; `attention_mask` (image) → white regions apply the guide, black regions ignore it. Without a mask, behavior matches the basic node.
**Widgets.** All from basic node plus `attention_strength`, `attention_mask`.

#### `LTXVDrawTracks`

**What it does.** Interactive canvas where you **draw sparse motion vectors** by hand — arrows indicating where specific points should move over the clip.
**Why it's needed.** For motion-track IC-LoRA, the "guide" isn't a depth/Canny preprocessor — it's hand-drawn motion intent. This node is how you author that intent without needing a reference video.
**Effect on video.** Arrows you draw become motion paths in the generated video. Sparse tracks (a few arrows) → broad motion guidance; dense tracks → precise choreography. Best for enforcing specific camera paths or character movement.
**Widgets.** Interactive canvas.

#### `LTXVSparseTrackEditor`

**What it does.** Companion editor to `LTXVDrawTracks` — refines, deletes, and repositions individual tracks after initial drawing.
**Why it's needed.** Drawing perfect tracks in one pass is rare; this lets you iterate without redrawing everything.
**Effect on video.** Same as `LTXVDrawTracks` — the edited tracks become the motion guide.
**Widgets.** Interactive editor.

**Preprocessors (generic, from `comfyui_controlnet_aux` / `ComfyUI-VideoDepthAnything`):**

#### `VideoDepthAnythingProcess` + `LoadVideoDepthAnythingModel`

**What it does.** Runs a video through Depth Anything to produce a per-frame depth map video (grayscale, near = bright, far = dark).
**Why it's needed.** Depth is one of the three cue types used by the Union IC-LoRA. You feed a reference video of real motion, Depth Anything extracts depth per frame, and the IC-LoRA uses that depth stack to constrain spatial layout in the generated video.
**Effect on video.** Output preserves the depth/spatial layout of the reference but allows the model to paint any surface texture / style on top. Model variant: `vits` (fastest, roughest), `vitb` (balanced), `vitl` (slowest, most accurate). Higher input resolution → sharper depth edges, higher VRAM.
**Widgets.** Model variant, input resolution.

#### `CannyEdgePreprocessor`

**What it does.** Runs Canny edge detection per frame, producing a black/white edge-map video.
**Why it's needed.** Canny is one of the three Union IC-LoRA cue types — it pins the edge structure (object outlines, silhouettes) while leaving fill/style to the prompt. Good for preserving the exact composition of a reference while restyling it.
**Effect on video.** `low_threshold ↓` → more edges detected (more detail preserved, but noisier); `high_threshold ↑` → cleaner edges, fewer false positives. Typical: `(100, 200)` for clean references, `(50, 150)` for noisy footage.
**Widgets.** `low_threshold`, `high_threshold`.

#### `DWPreprocessor`

**What it does.** Runs DWPose (a fast, accurate pose estimator) per frame, producing a skeleton-overlay video.
**Why it's needed.** Pose is the third Union IC-LoRA cue — it constrains body/character poses per frame, letting you transfer choreography from a reference dancer to your generated character.
**Effect on video.** `detect_body=True` → full-body skeleton (required for pose IC-LoRA). `detect_hand=True` → adds finger joints (only helps if your model handles hands well; LTX-2.3 is weak at hands — §11.7). `detect_face=True` → facial keypoints (usually overkill; leave off).
**Widgets.** `detect_hand`, `detect_body`, `detect_face`.

#### `ResizeImageMaskNode` (KJNodes)

**What it does.** Resizes an image or mask to a target resolution using a chosen interpolation method.
**Why it's needed.** Anchor images (`LTXVAddGuide`) and guide videos must match the latent's spatial dimensions; otherwise you get shape errors or distortion. This is the standard resizer in LTX workflows.
**Effect on video.** `upscale_method="lanczos"` → sharpest result for upscales; `"bilinear"` → smoother, less ringing; `"nearest"` → pixel-art / hard edges. Wrong method on an anchor image can visibly bleed into the generated first frame.
**Widgets.** `width`, `height`, `upscale_method`.

---

### 12.9 Where to look when this table isn't enough

For full widget-by-widget docs, interaction quirks, and newer nodes not covered here:

- **Core ComfyUI nodes**: the built-in node list at `http://127.0.0.1:8188/` → right-click → `Help`, or the ComfyUI wiki (`docs.comfy.org`).
- **LTX-2.3 nodes**: `custom_nodes/ComfyUI-LTXVideo/README.md` and the shipped workflow JSONs in `example_workflows/2.3/` — the JSONs are the most authoritative reference for which widgets are expected to be connected vs. left at defaults.
- **VideoHelperSuite**: `github.com/Kosinkadink/ComfyUI-VideoHelperSuite` README.
- **ComfyUI-GGUF**: `github.com/city96/ComfyUI-GGUF`.
- **KJNodes**: `github.com/kijai/ComfyUI-KJNodes`.
- **controlnet_aux preprocessors**: `github.com/Fannovel16/comfyui_controlnet_aux`.
- **RIFE / frame interpolation**: `github.com/Fannovel16/ComfyUI-Frame-Interpolation`.
- **Advanced-ControlNet**: `github.com/Kosinkadink/ComfyUI-Advanced-ControlNet`.

When a widget's effect isn't obvious from its name, the fastest test is **lock the seed, change one widget by a large step, re-render at 512×384 / 49 frames** (§11.9). Most widgets reveal their direction within two iterations.

---

*This guide covers the universal building blocks. Each specific model will have its own loader nodes, conditioning quirks, and optimal settings — but the overall structure remains the same. §9–§11 demonstrate how the pipeline maps to a real audio+video model (LTX-2.3) including its advanced LoRA and keyframe controls, and its Gemma-based prompting grammar.*

For storytelling and production patterns on top of this pipeline (shot lists, style replication, character consistency, per-shot-type recipes, post-production, and deeper prompt craft), see the companion `animation_course/` series.
