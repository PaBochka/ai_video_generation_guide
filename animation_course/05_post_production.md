# Module 5 — Post-Production

You have a folder full of `shot_XXX/final.mp4` files. This module turns them into a watchable short. Topics:

1. Editing tool setup (DaVinci Resolve free)
2. Assembling the cut
3. Color grading for consistency
4. Frame interpolation (smoothing motion)
5. Sound design (music, SFX, dialogue, lip-sync)
6. Final mix and color
7. Export
8. Project hygiene

> **Related ComfyUI Guide sections** — [ComfyUI Video Workflow Guide](../ComfyUI_Video_Workflow_Guide.md):
> - **§2.7** — video output / export node reference (`VHS_VideoCombine`, frame-sequence save, fps/codec/CRF).
> - **§3.4** — in-ComfyUI frame interpolation via RIFE/FILM nodes. Alternative to RIFE-App if you'd rather stay in the same graph.
> - **§6** — essential custom node packs (VideoHelperSuite, Frame-Interpolation) — install these before attempting the stitch-in-ComfyUI flow.
> - **§9.11** — audio prompting conventions. If you're re-rendering shots with generated audio (rather than overlaying Suno/Udio in Resolve), this is the prompt grammar.

---

## 5.1 Tool setup

| Tool | Use | Cost |
|------|-----|------|
| **DaVinci Resolve (free)** | Edit, color, audio mix, export — all in one | Free |
| Adobe Premiere | Same as Resolve, industry standard | Subscription |
| FFmpeg | Command-line cutting / concat / format conversion | Free |
| Audacity | Audio editing if Resolve's Fairlight is overkill | Free |
| **ElevenLabs** | High-quality TTS for dialogue | Paid (free tier limited) |
| **Suno / Udio** | AI music generation | Paid (free tier exists) |
| Wav2Lip / SadTalker | Lip-sync video to audio | Free, finicky setup |
| Topaz Video AI | Frame interpolation, upscaling | Paid |
| RIFE-App | Free frame interpolation | Free |

**Recommended starting stack (all free):** DaVinci Resolve + Audacity + RIFE-App + Suno free tier + ElevenLabs free tier.

---

## 5.2 Assembling the cut

1. Open DaVinci Resolve. Create new project. Set timeline to your target FPS (25 if you matched LTX-2.3 default).
2. Import all `shot_XXX/final.mp4` files. Resolve's media pool sorts them numerically.
3. Drag shots onto timeline in shot-list order.
4. **Don't trim yet.** Just lay them down end-to-end. Watch the rough cut.
5. Note where it drags or rushes. Adjust shot lengths by trimming heads and tails.

**Editing rules of thumb for AI animation:**

- **Cut on motion.** When a character is mid-gesture, cut to the next shot. The motion masks the edit.
- **Avoid cutting on stillness.** Two static shots back-to-back read as a slideshow.
- **Action cuts faster than dialogue.** Action shots: 1–3 seconds. Dialogue shots: 3–6 seconds.
- **The L-cut is your friend.** Audio from shot B starts before video from shot B (under shot A's tail). Smooths transitions.

---

## 5.3 Color grading for consistency

This is where AI shorts go from "obviously AI" to "looks intentional." Even with identical prompts, AI-generated shots have inconsistent contrast, white balance, and saturation. Color grading harmonizes them.

**Workflow in DaVinci Resolve:**

1. Switch to the **Color** page.
2. **Step 1 — Normalize**: For each shot, drop a basic color correction node. Match black point and white point across shots. Scopes (waveform + parade) are your friend.
3. **Step 2 — Apply a scene LUT**: Create or import a LUT that defines your "look" — color cast, contrast curve, saturation. Apply to every shot. This is the master grade that makes them feel like one production.
4. **Step 3 — Per-shot tweaks**: Some shots will still pop wrong. Add a small adjustment node per shot to bring outliers in line.

**LUTs for animation styles:**

- Spider-Man 94 → warm shadows, slightly crushed blacks, oversaturated mids, slight magenta cast in lights. Search "90s VHS LUT" or "Saturday morning cartoon LUT" online.
- 80s anime OVA → cool shadows, deep cyan blacks, magenta highlights, film grain. Search "Akira LUT" or "anime film stock LUT".
- Modern anime → high contrast, clean whites, slight teal-orange contrast. Search "anime cinematic LUT".

**Even simpler**: dial in a single grade by eye. Increase contrast 10–15%, drop saturation 5%, add a slight color cast (warm shadows, cool highlights) to match your style. Apply to all clips.

---

## 5.4 Frame interpolation (smoothing motion)

LTX-2.3 outputs are typically 25 fps. For smoother motion, interpolate to 50 or 60 fps.

**Use cases:**
- Action shots benefit from interpolation (smoother camera moves, less jitter on fast motion).
- Dialogue / static shots — leave at 25 fps. Anime is often deliberately at lower fps; interpolating it makes it feel weird.

**Tools:**
- **RIFE** in ComfyUI: install [`ComfyUI-Frame-Interpolation`](https://github.com/Fannovel16/ComfyUI-Frame-Interpolation). Add `RIFE VFI` node after VAE Decode. Set interpolation factor to 2 (25→50) or 4 (25→100).
- **RIFE-App** (standalone): drag MP4 in, pick output FPS, export.
- **Topaz Video AI**: paid, best quality.

**Tip for limited animation styles** (Spider-Man 94, old anime): **don't interpolate**. Limited animation gets its character from the held cels and choppy motion. Smoothing it ruins the look.

---

## 5.5 Sound design — the secret weapon

AI shorts often look mediocre but sound terrible. Fix the sound and your short jumps two quality levels.

**Layer order in your audio mix (from bottom to top):**

| Layer | Source | Volume |
|-------|--------|--------|
| Ambient bed | Generated by LTX-2.3 OR royalty-free ambience library | -25 dB |
| Music score | Suno / Udio / royalty-free | -15 to -10 dB |
| Foley / SFX | Freesound.org / generated | -8 to -3 dB per cue |
| Dialogue | ElevenLabs / your voice | -6 dB peak, -12 dB average |

**You will often replace the LTX-2.3-generated audio entirely**. The model's audio is good for atmosphere but rarely for finished mix. Mute the original audio track and rebuild.

### 5.5.1 Music

[Suno](https://suno.com) and [Udio](https://udio.com) generate full-length music from text prompts. For your style:

- Spider-Man 94 → "1990s American cartoon TV theme music, brass, electric guitar, synth pad, energetic, 90s production"
- 80s anime → "1988 Japanese anime synth score, analog synthesizers, ambient, cinematic, Vangelis-influenced"
- Modern anime → "modern Japanese anime opening, orchestral hybrid with electric guitar, hopeful, cinematic"

Generate 2-3 candidates. Pick the one that fits the scene's emotional arc. Trim to scene length. Loop or fade as needed.

**Royalty-free alternatives** (no AI): YouTube Audio Library, Pixabay Music, Free Music Archive.

### 5.5.2 SFX

For specific cues (a door slam, a punch impact, a web-shooter):

- [Freesound.org](https://freesound.org) — massive free library, Creative Commons.
- [Pixabay SFX](https://pixabay.com/sound-effects) — free, no attribution.
- AI generation: [ElevenLabs Sound Effects](https://elevenlabs.io/sound-effects), Stable Audio.

Don't skip foley. The sound of a footstep, a coat moving, a hand placing a cup — these sell the reality of the scene. AI rarely generates good foley; layer it in manually.

### 5.5.3 Dialogue

LTX-2.3 generates speech-like audio that doesn't lip-sync. For real dialogue:

1. **Write the lines** in a `dialogue.md` per scene.
2. **Generate or record voices**:
   - [ElevenLabs](https://elevenlabs.io) — best AI voices, voice cloning.
   - Record yourself with a USB mic (Shure MV7, Blue Yeti).
   - Royalty-free voice libraries.
3. **Trim each line** in Audacity / Resolve. Get clean takes.
4. **Drop into the timeline** under the matching shot.

### 5.5.4 Lip-sync

Two open-source options:
- **Wav2Lip** — older, more reliable. Requires Python setup. Inputs: video + audio. Outputs: same video with mouth area replaced to match audio.
- **SadTalker** — newer, slightly better for stylized faces. Same workflow.

**Reality**: lip-sync on stylized animation faces (anime, cel-shaded) is much harder than on photo-real faces. Wav2Lip was trained on real video; results on anime are uncanny. **For animated styles, often the right call is to not lip-sync precisely** — animation has historically been loose on lip-sync (90s Saturday-morning was famously bad), and viewers accept it. Just match the timing roughly.

For close-ups where lip-sync matters: consider re-rendering the shot with the character looking away or in profile, sidestepping the problem.

---

## 5.6 Final mix

1. Set your master output level. Aim for **-14 LUFS integrated** (YouTube target) or **-16 LUFS** (Apple/streaming).
2. Use a limiter on the master bus, ceiling -1 dBTP.
3. Listen on headphones AND on phone speakers AND on a TV/laptop. Each surfaces different problems.
4. EQ-clean your dialogue (high-pass at 100Hz, slight presence boost at 3kHz).
5. Sidechain music to dialogue (ducks music when dialogue plays).

This is full audio engineering territory; if it's new to you, even a basic pass (level matching + limiter) is enough for a watchable result.

---

## 5.7 Final color pass

After audio is locked, do one final color review:

1. Watch the cut start to finish.
2. Note any shot that pops out (too dark, too saturated, wrong tint).
3. Adjust those individual shots.
4. Apply a final master grade — slight S-curve for contrast, slight saturation pull (-3 to -5%) — to give it a "finished" feel.

---

## 5.8 Export

**For YouTube / web playback:**

| Setting | Value |
|---------|-------|
| Codec | H.264 |
| Container | MP4 |
| Resolution | match your master (1920×1080 if you upscaled, or native LTX output) |
| FPS | 25 (or 50 if you interpolated) |
| Bitrate | 12–20 Mbps (1080p) |
| Audio | AAC, 320 kbps stereo |
| Color space | Rec. 709 |

**For TikTok / Instagram (vertical):**

| Setting | Value |
|---------|-------|
| Aspect | 9:16 |
| Resolution | 1080×1920 |
| Length | <60s for max reach |
| Audio | mono or stereo |

**For archival / future re-edit:**

| Setting | Value |
|---------|-------|
| Codec | ProRes 422 HQ or DNxHR HQ |
| Container | MOV |
| Audio | uncompressed WAV |

Export the YouTube-ready MP4 for distribution. Keep the ProRes archive for future edits.

---

## 5.9 Project hygiene — keep your work findable

After every project, organize:

```
project/
├── 00_PLAN/        (shot list, character descriptions, style preamble)
├── 01_REFERENCES/  (style refs, character sheets, mood images)
├── 02_RAW_RENDERS/ (every generation attempt — yes, keep them)
├── 03_PICKS/       (one final.mp4 per shot)
├── 04_AUDIO/
│   ├── music/
│   ├── sfx/
│   ├── dialogue/
│   └── ambient/
├── 05_EDIT/        (DaVinci Resolve project, exports, intermediate WIP)
├── 06_FINAL/       (delivered files, ready to publish)
└── README.md       (one-paragraph: what this project is, status, where to pick up)
```

Future-you will thank present-you when you come back in three months wanting to make a sequel.

---

## 5.10 Common post-production issues and fixes

| Issue | Fix |
|-------|-----|
| Shots feel disconnected | Add a 10–15 frame audio crossfade across cuts; tighter color grade across shots |
| Color jumps between shots | Per-shot grade in DaVinci color page; consider re-rendering outliers |
| Motion looks janky | RIFE interpolate to 50fps (action only); leave dialogue at 25 |
| Audio sounds amateur | Add ambient bed under everything; add specific foley on every action |
| Dialogue feels detached | Add room reverb matching the visual space; sidechain music duck |
| Title text looks AI-rendered (warped) | Generate background only; add real title text in Resolve title tool |
| Logo / chest emblem inconsistent across shots | Mask the area in post; overlay the correct logo as PNG with motion tracking |
| Hands / fingers look bad in close-up | Cut the shot tighter (frame out the hands), or replace with insert shot |
| Lip movement doesn't match speech | Ignore for stylized animation; or re-render with character in profile / silhouette |
| Final feels short | You probably trimmed too tight in editing. Add 5–10 frames of breath at the head/tail of each shot. |

---

## 5.11 Capstone delivery checklist

- [ ] All shots edited in shot-list order
- [ ] Audio mixed across all four layers (ambient, music, SFX, dialogue)
- [ ] Color grade harmonized across all shots
- [ ] No black frames or audio gaps between shots
- [ ] Title card with project name (in post, not generated)
- [ ] End slate with credits (in post)
- [ ] Exported H.264 MP4 at 1080p
- [ ] Exported ProRes archive
- [ ] Project folder organized per §5.9
- [ ] README.md updated with date, length, status

When all boxes check, you have a finished short.

---

## 5.12 Where to go from here

You've completed the course end-to-end. You can now produce 30-second to multi-minute animated shorts in a chosen style with consistent characters and audio.

Next steps to grow:

- **Train a style LoRA** (module 2.7) so you stop tuning prompts every project.
- **Train a character LoRA per recurring character** (module 3.7) for series work.
- **Build a template project folder** (the §5.9 structure) and clone it for every new short.
- **Watch your finished short with non-creator friends.** Note where they laugh, lean in, or look away. That's your real feedback.
- **Ship more.** The next short will be 2x faster than this one because the pipeline is now muscle memory. The fifth short will look 10x better than the first.

The constraints are real. The aesthetic is also real, and emerging fast. You're early.
