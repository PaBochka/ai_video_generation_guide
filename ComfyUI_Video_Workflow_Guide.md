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

| Stage | Steps | CFG | Sampler | Notes |
|-------|-------|-----|---------|-------|
| Dev, stage 1 | 20 (up to 25–35) | 4.0 (range 2–5) on `MultimodalGuider` (or `scale=28` per template) | `euler` / `euler_ancestral` | `LTXVScheduler` + `LTXVNormalizingSampler` |
| Dev, stage 2 (upsample) | 3–4 | 1.0 (`CFGGuider`) | `euler` | `LTXVScheduler` |
| Distilled (single stage) | 8 | 1.0 (`CFGGuider`) | `euler` | `ManualSigmas` (see below), no `LTXVScheduler` |

**`LTXVScheduler` parameter meaning:**

| Param | Default | Effect when raised |
|-------|---------|---------------------|
| `max_shift` | 2.05 | Pushes schedule toward noisier sigmas at start → stronger global motion, looser structure |
| `base_shift` | 0.95 | Tilts the whole curve toward higher noise; lower it for a more linear flow-matching curve |
| `stretch` | True | If True, rescales sigmas so the schedule terminates at `terminal` instead of ~0 |
| `terminal` | 0.1 | Cutoff sigma; denoising stops here, leaving headroom for stage-2 refinement. Lower = cleaner stage-1; higher = more detail headroom |

**Distilled workflow uses `ManualSigmas` instead of `LTXVScheduler`.** The shipped 2.3 single-stage distilled JSON hard-codes:

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

*This guide covers the universal building blocks. Each specific model will have its own loader nodes, conditioning quirks, and optimal settings — but the overall structure remains the same. §9–§11 demonstrate how the pipeline maps to a real audio+video model (LTX-2.3) including its advanced LoRA and keyframe controls, and its Gemma-based prompting grammar.*

For storytelling and production patterns on top of this pipeline (shot lists, style replication, character consistency, per-shot-type recipes, post-production, and deeper prompt craft), see the companion `animation_course/` series.
