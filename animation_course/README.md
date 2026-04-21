# AI Animation Course — Building Watchable Animated Shorts in ComfyUI

A practical, hands-on course for going from "I have a working LTX-2.3 workflow" to "I can produce a 30-second to multi-minute animated short in a chosen style with consistent characters and a coherent story."

The course is opinionated about the current state of the tech: in 2026, AI video tools generate clips of ~5–10 seconds at a time, drift on character identity across cuts, and rarely nail a specific stylized look on the first try. **Everything in this course is built around those constraints**, not around pretending they don't exist.

---

## Prerequisites

You should already be comfortable with:

- ComfyUI basics — loading workflows, queueing, finding nodes, installing custom node packs.
- The [**ComfyUI Video Workflow Guide**](../ComfyUI_Video_Workflow_Guide.md) in this repo, especially:
  - **§9** — LTX-2.3 base pipeline (T2V / I2V two-stage with audio).
  - **§10** — LoRA stacking, camera-control LoRAs, multi-keyframe anchors, looping sampler.
  - **§11** — Prompting Gemma for LTX-2.3 (the text encoder is not CLIP — the grammar is different).

If you don't have a working LTX-2.3 single-shot workflow yet, build one from §9 first. The course recipes assume that pipeline as the engine.

---

## What you'll be able to do at the end

- Plan a sequence: idea → script → storyboard → shot list → prompts.
- Replicate a target style consistently across many shots (e.g., **Spider-Man: The Animated Series (1994)**, **80s OVA anime**, **modern cel-shaded anime**, **limited-animation Saturday-morning cartoon**).
- Keep a character (face, costume, color palette) recognizable across cuts.
- Generate per-shot footage with the right shot type, camera move, and motion.
- Assemble shots into a scene with audio, music, dialogue, and color continuity.
- Export at the right format for YouTube, TikTok, or local playback.

---

## Course map

| # | File | What it teaches |
|---|------|-----------------|
| 1 | [01_animation_fundamentals.md](01_animation_fundamentals.md) | Shot types, animation terms, idea → shot list, what AI video can / can't do |
| 2 | [02_style_replication.md](02_style_replication.md) | Replicating specific animation styles — prompt scaffolds + LoRAs + post-process |
| 3 | [03_character_consistency.md](03_character_consistency.md) | Keeping a character recognizable across cuts — the hardest problem |
| 4 | [04_shot_workflows.md](04_shot_workflows.md) | Concrete LTX-2.3 recipes per shot type (establishing, close-up, action, transition) |
| 5 | [05_post_production.md](05_post_production.md) | Assembling shots, color grading, frame interpolation, sound, export |
| 6 | [06_prompting_techniques.md](06_prompting_techniques.md) | Prompt techincs |

---

## Suggested learning path

**Week 1 — Foundations**
- Read [01_animation_fundamentals.md](01_animation_fundamentals.md). Watch one episode of your target style with a notepad. Write down: shot count, average shot length, camera moves, recurring color palette.
- **Exercise**: pick a 30-second scene from existing animation. Break it into a shot list with timecodes.

**Week 2 — Style**
- Read [02_style_replication.md](02_style_replication.md). Test the prompt scaffolds for your chosen style on three different scenes (interior, exterior, character close-up).
- **Exercise**: produce one 5-second clip in your target style at acceptable quality. Iterate on the prompt scaffold until it locks.

**Week 3 — Characters**
- Read [03_character_consistency.md](03_character_consistency.md). Build a **character sheet** for one character (front, 3/4, side, expressions).
- **Exercise**: generate the same character in three different shots (close-up, medium, wide) and compare consistency. Try the IP-Adapter route, then the LoRA route if you want better results.

**Week 4 — Shot production**
- Read [04_shot_workflows.md](04_shot_workflows.md). Pick the right recipe for each shot in your Week 1 shot list.
- **Exercise**: produce all shots in your shot list. Don't worry about audio yet.

**Week 5 — Post-production**
- Read [05_post_production.md](05_post_production.md). Stitch your shots in DaVinci Resolve (free). Add music, sound effects, basic color grade.
- **Exercise**: export the final 30-second short. Watch it three times. Note what breaks immersion. Re-render those shots.

**Week 6+ — Scale up**
- Apply the same pipeline to a 2–5 minute short.
- Train a custom character LoRA (covered briefly in module 3).
- Build a recurring style LoRA so you don't have to re-tune the prompt scaffold.

---

## Capstone project

Pick one:

1. **30-second action scene** — Spider-Man-94 style web-swinging across a rooftop. Two characters, three shots, one music cue.
2. **30-second anime cold open** — moody establishing shot, character introduction, hard cut to title. Single character, three shots, ambient audio.
3. **Looping music video clip** — 8-second loopable scene synced to a music bed. Single character, single shot, looping sampler.

Whichever you pick, the course teaches all the techniques you need. The capstone is where you put them together end-to-end.

---

## Glossary

| Term | Meaning |
|------|---------|
| **Shot** | One continuous take of footage — one camera, no cuts. The atomic unit of editing. |
| **Scene** | A series of shots happening in one place / time. |
| **Sequence** | A series of scenes that form a coherent story beat. |
| **Cut** | The transition between two shots. |
| **Establishing shot** | Wide shot that shows the location and sets the scene. |
| **Close-up (CU)** | Tight framing on a face or object. |
| **Medium shot (MS)** | Subject from waist up. |
| **Two-shot** | Frame contains two characters. |
| **Over-the-shoulder (OTS)** | Frame from behind one character looking at another. |
| **Insert** | Brief shot of a detail (a clock, a hand, an object). |
| **Beat** | A single dramatic moment within a scene. |
| **Storyboard** | Sequence of drawings (one per shot) showing camera, action, and dialogue. |
| **Shot list** | Spreadsheet/document with one row per shot: number, type, length, camera, action, dialogue. |
| **Cel-shaded** | Look defined by flat color regions and bold ink outlines (cartoons, anime). |
| **Limited animation** | Style with few drawings per second; uses held cels, smear frames, panning. |
| **Held cel** | A single frame held for many video frames (the lips move, the body doesn't). |
| **Smear frame** | A single distorted frame that implies fast motion. |
| **Key frame / anchor frame** | A frame the AI must hit exactly. In LTX-2.3: an `LTXVAddGuide` input. |
| **In-between** | A frame between two key frames. The AI generates these implicitly. |
| **LoRA** | Small adapter weights that bias a model toward a style or character. See workflow guide §10. |
| **IP-Adapter** | Lets you condition generation on a reference image without training a LoRA. |
| **FLF2V** | First-Last Frame to Video. A form of multi-keyframe I2V. |
| **Looping sampler** | LTX-2.3 sampler that generates long clips in overlapping tiles. See workflow guide §10.5. |

---

## Honest limits as of 2026

- **Clip length per generation**: ~10s at 25 fps without quality dropoff (LTX-2.3 with looping sampler).
- **Character consistency across cuts**: hard. Best with a trained character LoRA. IP-Adapter + reference frame is second-best.
- **Lip-sync to dialogue**: requires post-process tools (Wav2Lip, SadTalker). Not native in LTX-2.3.
- **Action choreography**: simple actions OK; complex multi-character fights are very hard. Plan around this.
- **Specific stylization** (hand-drawn anime, cel-shaded): requires style LoRA or post-process. Pure prompts get you 60% of the way.
- **Multiple recurring characters in same shot**: harder than single character. Often easier to generate solo shots and composite.

These limits will move. Re-read this section in 6 months and a year of model releases will have shifted what's reasonable.
