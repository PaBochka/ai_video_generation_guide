# Module 2 — Style Replication

This module is about getting your generated footage to **look** like a specific animation style. It's the difference between "AI video" and "watchable animated short."

There are three layers of control, in order of impact:

1. **Prompt scaffolding** — language that biases the model toward a style. Free, fast, gets you to ~60% of the look.
2. **Style LoRAs** — adapter weights trained on that style. Adds another ~25%. Many are downloadable; some you'll train.
3. **Post-process style transfer** — apply a style to already-generated video. Adds the final 10–15% but is the slowest and most fragile.

Most courses stop at #1. We'll cover all three.

> **Related ComfyUI Guide sections** — [ComfyUI Video Workflow Guide](../ComfyUI_Video_Workflow_Guide.md):
> - **§9.4** — Lightricks' short concept-level negative prompt convention (`pc game, console game, video game, cartoon, childish, ugly`). Don't bring SD-style laundry-list negatives here.
> - **§10.1** — LoRA stacking mechanics (`LoraLoaderModelOnly` vs. `LTXICLoRALoaderModelOnly`, 3-LoRA practical ceiling, strength guidance for style + speed + IC combinations).
> - **§11** — Gemma prompting (five-paragraph structure, emphasis without `(word:1.4)`, word-order anchoring). The style preamble goes in ¶1.

---

## 2.1 Style anatomy — what defines a "look"

Before you can replicate a style, you have to be able to describe it in concrete visual terms. "Looks like Spider-Man 1994" is too vague for a prompt. Decompose it into:

| Dimension | Example values |
|-----------|----------------|
| **Line work** | None / thin pencil / thick black ink / digital outline / no outline |
| **Shading** | Cel-shaded (flat regions) / soft gradient / cross-hatched / painterly |
| **Color palette** | Saturated primary / muted earth tones / pastel / monochrome accent / neon |
| **Detail level** | Sparse (limited animation) / moderate / hyper-detailed (Akira) |
| **Background style** | Painted matte / clean cel / photo composite / abstract |
| **Motion style** | Smooth (12+ fps animation) / limited (held cels) / impact-driven (anime fight) |
| **Composition** | Comic-panel / cinematic widescreen / TV 4:3 / vertical mobile |
| **Lighting** | Flat / dramatic chiaroscuro / soft naturalistic / neon practical |
| **Era markers** | 80s grain + chromatic aberration / 90s VHS / 2000s digital sharp / modern HDR |

**Exercise**: pick your target reference (a single frame from your favorite show), and fill in this table. The values you write are your prompt vocabulary.

---

## 2.2 Prompt scaffolding — the universal template

Every shot prompt for animation should follow roughly this structure:

```
[STYLE PREAMBLE]. [SHOT TYPE + SUBJECT]. [ACTION + CAMERA]. [LIGHTING + ATMOSPHERE]. [SOUND CUE].
```

Concrete fill:

```
1990s Saturday-morning American animated TV series, bold black ink outlines, flat
cel-shaded coloring, saturated comic-book color palette, limited animation feel.

Medium close-up of a masked vigilante in red and blue costume standing on a rooftop
at night, rain falling on his shoulders.

He slowly turns his head to the right, looking down toward the street below. The
camera is locked-off, no movement.

Wet asphalt reflections, distant neon signs in cyan and magenta, dramatic backlight
from the city below silhouetting his figure.

Light rain hisses against the rooftop, distant city ambience, a faint synth pad in
the background.
```

This is wordy on purpose. The first paragraph (the **style preamble**) is the same across every shot in your project — you literally copy-paste it. The rest changes per shot.

**Negative prompt** (Lightricks convention from workflow guide §9.4):

```
pc game, console game, video game, cartoon, childish, ugly
```

For specific anti-style cleanup, add concept-level terms only — not the SD-style "blurry, distorted, extra fingers" laundry list. If your style prompt says "1990s American animation" and the result is anime-flavored, add `anime, manga, japanese animation` to the negative.

---

## 2.3 Style preambles for the three reference styles

### A. Spider-Man: The Animated Series (1994) — preamble

```
A 1994 American animated superhero TV series, hand-drawn cel animation, bold black
ink outlines on every form, flat saturated coloring with no gradients, limited
TV-budget animation, comic-book composition, warm color palette dominated by browns
and deep purples with saturated red and blue costume accents, dramatic
chiaroscuro lighting from off-screen sources, mid-1990s Saturday-morning network
broadcast aesthetic, slight VHS softness.
```

Anti-prompt additions: `anime, manga, photorealistic, 3D rendered, modern digital animation, smooth animation`.

Tuning notes:
- If results feel too modern: add "1990s analog broadcast", "TV cel animation", "vhs era"
- If results lack the bold ink: add "thick black outlines on all shapes", "comic book ink"
- If color feels muted: add "saturated comic book color palette"

### B. 80s OVA anime — preamble

```
A 1988 Japanese OVA anime film, hand-drawn cel animation in the style of late-1980s
Tokyo studios, detailed line work with subtle pencil texture, painted backgrounds
with extreme architectural detail, cool color palette dominated by cyan, deep
indigo, and magenta neon accents, cinematic widescreen 16:9 composition, dramatic
lens flares from practical light sources, moody atmospheric lighting, slight
analog film grain, anamorphic lens character, Akira-era visual sophistication.
```

Anti-prompt: `american cartoon, modern digital anime, 3D rendered, smooth flash animation, TV budget`.

Tuning:
- For the wet-asphalt cyberpunk look: add "rain, wet streets, neon reflections, fog"
- For the painted background look: add "matte painted background, watercolor washes"
- If too modern: add "cel film stock, 1988, OVA"

### C. Modern cel-shaded anime — preamble

```
A modern Japanese television anime in the style of contemporary action series,
clean digital cel-shading with crisp line work, high-contrast dramatic lighting,
saturated character colors against painterly background art, occasional impact
frames with bold motion lines, exaggerated character expressions, cinematic
camera angles, polished modern Japanese animation production, 24fps anime feel.
```

Anti-prompt: `realistic, 3D rendered, american cartoon, retro anime, vhs grain`.

Tuning:
- For action sequences: add "speed lines, impact frame, dynamic angle"
- For dialogue: add "soft anime lighting, expressive eyes"

---

## 2.4 The "consistency lock" — keep the same scaffold

**Do not rewrite your style preamble per shot.** Copy-paste the chosen preamble at the top of every shot's prompt. Only the action/subject/camera/atmosphere lines change.

This is the single biggest determinant of cross-shot style consistency. The model sees the same opening tokens and biases the same way every time.

If you want to tweak the preamble, do it once, then **re-render every shot in the project** with the new preamble. Don't mix preamble versions in one final cut.

---

## 2.5 Style LoRAs — when prompts aren't enough

Prompts get you 60% of the way. The other 40% comes from LoRAs trained on the target style.

**Where to find LoRAs:**
- [Civitai](https://civitai.com) — largest LoRA library; filter by base model. As of 2026, native LTX-2.3 LoRAs are still rare; many are for SDXL / Flux and require conversion or are video-incompatible.
- [HuggingFace](https://huggingface.co) — search `LTX video LoRA` or `LTX-2 LoRA`. Lightricks publishes camera and IC-LoRAs there.
- [Tensorart](https://tensor.art) — similar to civitai.

**As of 2026, the LTX-2.3 LoRA ecosystem is thin.** Most style LoRAs are still SDXL/Flux. You have three options:

1. Use the model's natural style range via prompt (covered above) — works for general anime/cartoon styles.
2. Train your own style LoRA on LTX-2.3 (covered in §2.7).
3. Generate at base style with LTX-2.3, then apply style transfer in post (covered in §2.8).

### Loading a style LoRA on LTX-2.3

Use `LoraLoaderModelOnly` (workflow guide §10.1). Chain it after the distilled-speed LoRA:

```
CheckpointLoaderSimple
  └── MODEL ──▶ LoraLoaderModelOnly (distilled, str=0.5)
                  └── MODEL ──▶ LoraLoaderModelOnly (style LoRA, str=0.4–0.7)
                                  └── MODEL ──▶ guider / sampler
```

Strength tuning:
- **Style LoRAs trained on still images**: start at 0.4. They were trained on stills, so video coherence can suffer at higher strengths.
- **Style LoRAs trained on video**: start at 0.6.
- If output looks like the LoRA but motion is mush: lower strength.
- If output looks like the base model with hint of style: raise strength.

Cap at 0.8 unless you've trained the LoRA yourself and know its tolerance.

---

## 2.6 Combining style LoRAs

You can stack two style LoRAs (e.g., `cel-shaded look` + `90s color palette`). Limits:

- **Max 2 style LoRAs** in addition to the speed LoRA. Three or more start to fight each other.
- **Total LoRA strength budget ≈ 1.5**. If style LoRA A is at 0.7, style LoRA B should be at 0.3–0.5 max.
- **Same training base** matters. LoRAs trained on different LTX versions don't mix well; LoRAs trained on different model families (LTX vs Wan vs Hunyuan) don't mix at all.

Combination examples:
- `90s_animation_style` (0.6) + `wet_neon_atmosphere` (0.4) = Spider-Man 94 in the rain
- `anime_lineart` (0.7) + `cyberpunk_palette` (0.3) = synthetic Akira

---

## 2.7 Training your own style LoRA (overview)

Out of scope for full coverage here, but the process at a high level:

1. **Collect dataset** — 50–200 stills (or short video clips) in your target style. Higher quality > more quantity. Avoid logos, watermarks, text.
2. **Caption** — write or generate a caption per image. Tools: [WD14 Tagger](https://github.com/picobyte/stable-diffusion-webui-wd14-tagger), [BLIP-2], or manual.
3. **Train** — use [kohya_ss](https://github.com/kohya-ss/sd-scripts), [OneTrainer](https://github.com/Nerogar/OneTrainer), or [diffusion-pipe](https://github.com/tdrussell/diffusion-pipe) with an LTX-2.3 training config. Expect 2–8 hours on a 24 GB GPU for a rank-128 LoRA.
4. **Test** — generate sample shots with a fixed prompt scaffold and various LoRA strengths.
5. **Iterate** — bad results usually mean dataset issues. Re-caption, prune, re-train.

For a course-scale project (your first 30-second short), **don't train a style LoRA yet**. Use prompt scaffolds + post-process. Save LoRA training for after you've shipped one short and know what specifically isn't working.

---

## 2.8 Post-process style transfer

When prompts and LoRAs aren't enough, apply the style after generation.

**Tool options:**

| Tool | What it does | Cost | Quality |
|------|--------------|------|---------|
| [RunwayML Gen-Restyle](https://runwayml.com) | Apply a reference style to a video clip | Paid | Good for stylized looks |
| [DomoAI](https://domoai.app) | Anime / cartoon restyle | Paid | Very good for anime-fication |
| [EBSynth](https://ebsynth.com) | Propagate a hand-painted keyframe across video | Free | Best for true cel-look; tedious |
| [Diffusion-based vid2vid in ComfyUI](https://github.com/Fannovel16/comfyui_controlnet_aux) | Use AnimateDiff / SDXL with ControlNet on each frame | Free | Inconsistent across frames |
| Manual rotoscope + paint | Frame-by-frame in Photoshop / Krita | Free, very slow | Best quality, impractical for >10s |

**The EBSynth recipe** (free, surprisingly good for cel/anime looks):

1. Generate your video in LTX-2.3 (semi-realistic base style).
2. Export frame 0 as a still.
3. Hand-paint or AI-restyle that single frame in your target look.
4. Feed the original video + the styled frame into EBSynth.
5. EBSynth propagates the painted style across the rest of the frames using optical flow.

Caveats: EBSynth degrades over long clips (>2s) and breaks on fast motion. Use it for short held shots; not for action.

**For a course project, recommended path:**
- Spider-Man 94 / cel-shaded → try DomoAI or RunwayML Restyle on each shot
- 80s anime → strong prompt + IC-LoRA Union for line work + DomoAI for finishing
- Modern anime → strong prompt + style LoRA from civitai (search "anime LTX")

---

## 2.9 Style continuity across shots — the practical checklist

When you've generated all shots in your shot list and you're stitching them together, walk this checklist:

- [ ] Same style preamble used in every shot prompt
- [ ] Same negative prompt across all shots
- [ ] Same speed LoRA + same style LoRA(s) at same strengths in all shots
- [ ] Same resolution and aspect ratio
- [ ] Same FPS in `LTXVConditioning` and `VideoCombine`
- [ ] Color grading applied to ALL shots in post (module 5) — the AI will color-drift even with identical prompts
- [ ] Lighting direction described consistently in every prompt (e.g., "key light from camera-left" everywhere)
- [ ] Time of day / weather described identically across the scene
- [ ] If a recurring character appears: identical character description in every prompt (module 3)

If any of those are off, your final cut will look like five different short films stapled together.

---

## 2.10 Style for sound

Don't forget audio is part of style. A Spider-Man-94 shot with a cinematic Hans Zimmer score sounds wrong; a Ghibli shot with chiptune sounds wrong.

In LTX-2.3, audio is generated from the same text prompt as the video. So your style preamble should also imply the audio aesthetic:

- "1990s Saturday-morning American animated TV series" → tends toward synth scores, MIDI horns, cartoon foley
- "1988 Japanese OVA anime" → tends toward analog synths, big choirs, 80s drum machines
- "Modern Japanese anime" → tends toward orchestral hybrid, J-pop influence

If the AI's audio doesn't match, **replace it in post** (module 5). Module 5 covers Suno / Udio for music and ElevenLabs for dialogue.

---

## 2.11 Exercise

1. Pick your style.
2. Write your style preamble. Use one of the templates above as a starting point.
3. Generate three test shots from your shot list (an EST, a CU, an action shot) using the same preamble.
4. Compare them side-by-side. Are they recognizably the same style?
5. If yes → move to module 3.
6. If no → tune the preamble. Add the missing dimension (line work? color? era?). Re-test.

Don't proceed until your test shots feel like they belong to the same animated property.
