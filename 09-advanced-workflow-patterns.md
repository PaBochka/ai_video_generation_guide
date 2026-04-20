# 09 — Advanced Workflow Patterns

Everything in this module is optional for a first short. Come back when
the basic pipeline feels comfortable and you are chasing a specific
quality ceiling.

## 1. Pose / depth / edge conditioning

Classical ControlNet-style conditioning is emerging in video diffusion.
For LTX-2.3 specifically, check the current node pack for:

- **Depth** conditioning (from a depth map video).
- **Pose** conditioning (OpenPose-style skeleton video).
- **Edge / Canny / HED** conditioning (line video).

The idea is the same as in SD: extract a control signal from a
reference video, feed it to the sampler alongside text, and the model
follows the control.

Typical pipeline:

```
   Reference video -> VHS_LoadVideo -> frames
                                       |
                          Depth/Pose estimator (per frame)
                                       |
                                    control video
                                       |
                  LTX-2.3 Sampler <-- control input
```

Use cases:

- **Pose-match a performance.** Film yourself doing the action, extract
  pose, drive an animated character through the exact same motion.
- **Camera matching.** Use depth from a live-action plate to drive a
  match-cut to an animated shot.
- **Rotoscope-style.** Strong edge control for a very on-model trace
  of reference footage. This is closest to the referenced mixed-media
  music-video aesthetic.

If LTX-2.3 2.3 in your install does not ship a first-party control node,
look at community node packs — they move quickly.

## 2. Img2vid with a "motion intensity" knob

Some LTX-2.3 sampler variants expose a **motion intensity** parameter
(also called "motion bucket", "motion strength", etc.). It biases the
noise toward more or less movement.

- **Low intensity:** subtle motion, good for portraits, reaction shots,
  slow atmospheric.
- **High intensity:** action, whip-pans, dynamic choreography. Quality
  of motion can degrade at extremes.

This is a cheap dial to try before adding more complex conditioning.
Grid-search it.

## 3. Denoise strength as a "style injection" dial

If you feed an existing animation (e.g. a RIFE-smoothed mediocre
generation) back into LTX-2.3 at low denoise strength (0.2–0.4), with a
new prompt, you can selectively re-paint style without losing the
underlying motion. This is a vid2vid pattern:

```
   Input video -> VHS_LoadVideo -> VAE encode
                                       |
                                noisy latent (partial)
                                       |
                                  LTX-2.3 Sampler -- new prompt + LoRAs
                                       |
                                   VAE Decode
```

Great for "I have the motion, I need a different look" situations.
Risks: flicker if denoise is too high, identity loss, prompt drift.

## 4. Upscaling

Native LTX-2.3 resolution is limited. For delivery, you upscale in post.
Common paths:

- **Image-model tile upscalers** run per-frame: e.g. `4xUltraSharp` or
  `RealESRGAN` via ComfyUI's image upscale nodes. Risk: temporal
  flicker because the upscaler has no temporal awareness.
- **Video-aware upscalers**: Topaz Video AI (commercial), or certain
  diffusion-based video upscalers. Cleanest output.
- **Latent upscaling + second pass**: upscale the latent video, then
  do a short low-denoise diffusion pass. Requires custom nodes and
  careful tuning. High quality when it works.

For most course projects, per-frame RealESRGAN at 2x is adequate.

## 5. Frame interpolation

RIFE (ComfyUI-Frame-Interpolation) is the standard. Workflow:

```
   VHS_VideoCombine (24 fps raw) -> frames out
                                       |
                                  RIFE 2x (or 4x)
                                       |
                                VHS_VideoCombine (48 fps final)
```

Notes:

- Interpolation is cheap compared to regenerating at higher fps.
- RIFE struggles with very fast cuts, flashes, or heavy motion blur.
  Expect artifacts on whip-pans and impact frames.
- Interpolating a 90s-SpiderMan shot to 60 fps destroys the era-
  specific feel. Be intentional.

## 6. Long shots by chaining

LTX-2.3's native length is limited. To produce a longer continuous shot:

- Generate clip A (0–4s).
- Take the last frame of A as the first frame of clip B (4–8s).
- Continue.

Risks:

- Drift: small errors accumulate shot over shot.
- Repetition: the prompt stays the same but you want the motion to
  evolve. Adjust the prompt for each chain step ("she is walking" ->
  "she stops and turns" -> "she draws her sword").

Alternative: **overlap windows**. Generate overlapping 4s clips with
2s overlap. Blend the overlaps. Smoother but 2x the compute.

## 7. Sound design and sync

Video is half picture, half sound. For the course final:

- Pick music *before* you lock picture. Cut to the beats. 120 BPM
  gives you a cut every 0.5s on beat, every 2s on bar — align shot
  boundaries to these.
- For 90s action styles: bass-heavy synth/funk beds, drum kit impacts
  on punches.
- For anime: string or piano motif, thematic leitmotif over action.
- Free-to-use music: YouTube Audio Library, Pixabay Music, Kevin
  MacLeod (attribution required).
- Foley: short whooshes on camera moves, impact hits on punches,
  ambient rain/wind under exterior shots. Freesound.org is the
  standard.

Tools: DaVinci Resolve (free) handles editing and audio competently.
If you only need to mash shots together, CapCut or iMovie work.

## 8. Color grading

Color is where amateurs look amateur. Minimum viable grading:

- **Lift blacks slightly, crush whites slightly.** Diffusion output
  often has washed-out blacks; a gentle S-curve fixes it.
- **One color overall.** Cool night, warm day, desaturated rain. A
  consistent temperature across the short is worth more than shot-
  by-shot creativity.
- **Match cuts.** If two adjacent shots have different color casts,
  they fight. Grade the pair as one.

In DaVinci Resolve: use the Color page, apply a single node per
timeline that pulls the whole project toward your target palette.

## 9. Fixing bad generations instead of re-rolling

Common patch-ups that are faster than re-generating:

- **Bad hands / faces on a key frame**: export the frame, inpaint in
  your T2I tool, replace the frame in the sequence, re-interpolate
  the neighbours.
- **Flicker on a small region**: mask the region, temporal-blur or
  median-filter only the mask across frames.
- **Seam between two stitched clips**: add a 2–4 frame crossfade, or
  insert a short "motion blur" overlay.

Video editing is problem-solving. The generator is one tool among
many.

## 10. Reproducibility at project scale

Once you have more than ~5 shots:

- Keep a `shots.csv` with columns: shot ID, prompt, negative, seed,
  LoRAs+strengths, input images, workflow file, duration, status.
- Keep a `prompts/` folder of reusable prompt fragments (your style
  boilerplate, your character description).
- Commit your workflow `.json` files to git. They are source.

This sounds like busywork. It is the difference between "I kind of
finished my short" and "I can rebuild, re-edit, or re-grade my short
next month".

## ControlNet for video — deep dive

Control conditioning is the single biggest quality upgrade once you
have the basics down. It lets you drive a generation with a structural
signal (pose, depth, edges) instead of relying on prompts alone.

### The three signals and when to use each

| Signal | What it captures | Best for | Weakness |
|---|---|---|---|
| Pose (OpenPose / DWPose) | Skeleton of joints | Matching a performance, choreography, blocking | Loses body shape, clothing, scale |
| Depth (MiDaS / DepthAnything) | Distance from camera per pixel | Camera matching, parallax preservation, scene geometry | Blurs small detail, can hallucinate on textureless areas |
| Edges (Canny / HED / LineArt) | High-contrast line map | Strong trace of reference, rotoscope feel | Baked-in identity of reference — hard to restyle |

Most productions use *one* signal, sometimes two combined. Three at
once usually over-constrains and the model collapses to a re-
rendering of the input.

### Worked pipeline — pose-driven anime action

Goal: film yourself doing a sword-swing motion, retarget to an anime
character.

Steps:

1. **Record** yourself swinging (phone, any angle, good lighting).
   Export to mp4 at matching resolution/fps for LTX-2.3.
2. **Extract pose** per frame: load the mp4 with `VHS_LoadVideo`, run
   each frame through `DWPose Estimator` (or `OpenPose`). Output: a
   sequence of skeleton images.
3. **Preview the skeleton video** via `VHS_VideoCombine` to confirm
   it tracks cleanly — if the skeleton jitters on a limb, your
   output will too. Re-shoot with better lighting if needed, or try
   `DWPose` which is more temporally stable than basic OpenPose.
4. **Generate the driven video** via LTX-2.3 with a `ControlNet` or
   equivalent conditioning node, fed the skeleton sequence alongside
   your character prompt + I2V key still.
5. **Tune the control strength.** 0.5–0.7 gives prompt room to
   breathe; 0.9–1.0 locks tightly to the skeleton.

### Worked pipeline — depth-driven match cut

Goal: cut from a live-action rooftop plate to an animated version of
the same rooftop with a character on it, preserving camera perspective.

Steps:

1. Extract depth from the plate via `DepthAnything V2` per frame.
2. Run LTX-2.3 T2V (not I2V — the character is new) conditioned on the
   depth map. Prompt: "anime-style rooftop, [character description]
   standing near the edge, matching camera perspective".
3. Blend: editor crossfade between plate and generation at the cut.
   Because depth matches, the blend reads as a stylistic transition,
   not a jump in space.

### Combining pose + depth

When choreography + stage matter: pose for the character, depth for
the environment. Keep control strengths low (~0.4 each) so the model
has room to make it coherent. Generate a grid of strength combos
before committing.

### Pose retargeting — size mismatch

You are 6' tall, your character is a 4'-tall anime protagonist.
OpenPose retargeting at raw scale gives a stretched skeleton driving
a normal-sized character — proportions break.

Fix options:

1. **Scale the skeleton video** before feeding to ControlNet: use an
   image scale node with different X and Y factors to reshape the
   skeleton bounding box to match your character's proportions.
2. **Ignore the head/face keypoints** (mask them out of the control
   image). The body pose drives the motion; the face is painted by
   the character LoRA and prompt.
3. **Shoot the reference with awareness**. Keep your arms close to
   body for a short-limbed character; exaggerate for a tall one.

## Vid2vid style transfer — walkthrough

Scenario: you have a live-action clip and want to stylise it as anime.

### Pipeline

```text
live_action.mp4
      |
VHS_LoadVideo (extract frames + fps)
      |
VAE Encode (per frame) -> video latent
      |
Add noise (partial, e.g. 40% strength)
      |
LTX-2.3 Sampler (new prompt + style LoRA + character LoRA)
      |
VAE Decode
      |
VHS_VideoCombine (output stylised mp4)
```

### Denoise strength as a "how stylised" dial

| Denoise | Result |
|---|---|
| 0.2–0.3 | Barely touched; colour/lighting shift, motion preserved |
| 0.4–0.5 | Recognisably restyled, subject identity mostly preserved |
| 0.6–0.7 | Heavy restyling, subject may re-read as the character in the prompt |
| 0.8–0.95 | Essentially a new generation loosely guided by the plate |
| 1.0 | Ignores the plate entirely — use T2V instead |

### Common vid2vid failures

- **Flicker on faces / hands.** The style LoRA is resolving inconsistently
  per-frame. Fix: lower denoise; or add edge control alongside
  denoise to anchor the face structure.
- **Drifting identity.** Subject at frame 1 is the target character;
  by frame 50 it's someone else. Fix: lower denoise; or add IP-Adapter
  with a reference image of the target character.
- **Backgrounds collapse into abstract shapes.** Style LoRA has a
  low background-detail training bias. Fix: add explicit background
  description to the prompt; or mask background and denoise it less.

## RIFE interpolation — deep dive

RIFE is a flow-based interpolation model. It looks at frame N and
N+1, estimates optical flow between them, and synthesises intermediate
frames along the flow field.

### RIFE strengths

- Smooth continuous motion (walking, camera pans, flowing water).
- Cheap — far cheaper than re-generating at higher fps.
- Temporally stable — no new artifacts, just smoother existing motion.

### RIFE weaknesses

- **Occlusion** — something appearing/disappearing frame-to-frame
  causes ghosting or melting.
- **Extreme motion** (> 30 pixels/frame at 24fps) — flow estimates
  fail, interpolations tear.
- **Flashes / whites** — impact frames averaged with their
  neighbours lose their snap.
- **Intentionally limited animation** — interpolating a 2s-hold
  destroys the on-2s rhythm you wanted.

### RIFE usage patterns

**Pattern A — global 2x.** Simple: put RIFE between `VideoCombine`
and the final save; export at 2x fps. Use for photoreal / live-
action pastiche.

**Pattern B — shot-selective.** Interpolate only shots where you
want smoothness. Keep anime shots raw. In Resolve: apply RIFE as a
plugin only on selected timeline clips.

**Pattern C — masked interpolation.** Interpolate only a region of
the frame (e.g. only the background, preserving on-2s character
animation). Requires a masked flow node; available in some node
packs, fiddly to set up.

### RIFE parameter notes

- `multiplier` or `times` (2x, 4x) — 2x is safe; 4x starts to show
  weaknesses.
- `scale` — lower scales speed up on high-res inputs at small
  quality cost. Default 1.0.
- `fast mode` — use only for preview; final renders on full quality.
- `ensemble` — averages forward and backward flow for better
  quality at ~2x compute. Worth it for final renders.

## Upscaling — comparison table and recipe

| Method | Per-frame quality | Temporal stability | Speed | Cost |
|---|---|---|---|---|
| RealESRGAN 4x (per frame) | Good | Medium (some flicker) | Fast | Free |
| 4xUltraSharp / 4xNMKD-Siax | Very good on clean source | Medium | Fast | Free |
| Topaz Video AI | Excellent | Excellent | Slow | Paid |
| Diffusion video upscaler (e.g. SUPIR on video) | Excellent | Good | Very slow | Free (heavy VRAM) |
| Latent upscale + low-denoise pass | Excellent (if tuned) | Medium–high | Slow | Free |

### Recipe — quick and cheap (RealESRGAN path)

```text
LTX-2.3 output 768x512  -> VHS_LoadVideo
                     -> ImageUpscaleWithModel (4xUltraSharp)
                     -> ImageScale (target 1920x1080, Lanczos)
                     -> VHS_VideoCombine (mp4, h264, crf 18)
```

Use 4x first then scale down to 1080p — preserves more detail than
direct-to-1080.

### Recipe — quality (latent upscale + second pass)

```text
LTX-2.3 output latent -> LatentUpscale 2x
                   -> LTX-2.3 Sampler (denoise 0.25, same prompt)
                   -> VAE Decode (upscaled)
                   -> per-frame ImageUpscale to final 1080p
```

Expensive but gives you genuinely added detail, not just sharpened
low-res.

## Long-shot chaining — concrete example

Goal: a 16-second continuous tracking shot. LTX-2.3 native max reliable
length per generation is ~4s (model-dependent). Chain four shots.

### Planning

Break the 16s into four beats with a prompt progression:

```text
Clip 1 (0-4s):   "character walks into room, scanning surroundings"
Clip 2 (4-8s):   "character stops, turns toward a desk, notices a letter"
Clip 3 (8-12s):  "character approaches the desk, picks up the letter"
Clip 4 (12-16s): "character reads the letter, expression shifts to concern"
```

### Chain mechanics

Each clip uses the **last frame of the previous clip** as its I2V
first frame.

Workflow:

1. Generate Clip 1 (T2V or from a storyboard start still).
2. Extract frame 96 of Clip 1 as `last_01.png`.
3. Generate Clip 2 with `last_01.png` as I2V input.
4. Repeat.

In the editor: butt-join the clips. The shared frame means no
discontinuity.

### Keeping identity across the chain

Without help, identity drifts by clip 3. Mitigations:

- **Reference still.** Feed a fixed character reference image into an
  IP-Adapter across every clip in addition to the chain frame.
- **Low-variance seeds.** Use seeds `N, N+1, N+2, N+3` rather than
  randoms.
- **Shared prompt scaffold.** Character identity tokens identical
  across all four prompts; only the action clause changes.

### Overlap-blend variant

To eliminate any visible seam at clip boundaries, generate overlaps:

```text
Clip 1: 0-5s
Clip 2: 4-9s   (1s overlap with Clip 1)
Clip 3: 8-13s
Clip 4: 12-17s
```

In the editor, cross-fade the overlaps. Adds 25% generation cost;
removes hard seam frames entirely.

## Sound design — concrete examples

### Free sources

- **YouTube Audio Library** — royalty-free music, searchable by mood
  and tempo.
- **Pixabay Music** — similar, no account needed.
- **Freesound.org** — SFX. Creative Commons, check individual licences.
- **BBC Sound Effects Archive** — massive, free for personal/
  educational use.
- **Kevin MacLeod (incompetech.com)** — music, attribution required.

### SFX layering — worked example (SpiderMan shot)

For a 2-second rooftop-leap shot:

```text
Frame  0-8   : ambient city rumble (low, continuous bed, -25 dB)
Frame  8-16  : muscle tension cue (subtle low thud, -18 dB)
Frame 16-24  : launch whoosh (0.3s, -10 dB, fast attack)
Frame 24-40  : sustained cloth/cape flap (continuous, -15 dB)
Frame 40-48  : web-shoot thwip (0.2s, -8 dB, sharp)
```

Each layer on its own Resolve track. Keyframe volumes. The result
sounds 10x richer than a single "action music" drop would.

### Tempo-to-cut math

Formula: seconds-per-beat = 60 / BPM.

| BPM | Beat | Bar (4 beats) |
|---|---|---|
| 90 | 0.67s | 2.67s |
| 100 | 0.60s | 2.40s |
| 120 | 0.50s | 2.00s |
| 140 | 0.43s | 1.71s |
| 160 | 0.38s | 1.50s |

Plan shot durations as multiples of the bar. A 12-second piece at
120 BPM = 6 bars = 6 natural cut points.

## Color grading — step-by-step in DaVinci Resolve

Minimum-viable grade for a stylised short.

### Step 1 — Global balance node

On the Color page, with your timeline active:

1. Create a new node (Shift+S).
2. Use the **Lift/Gamma/Gain** wheels:
   - Lift slightly cool (towards blue) for nighttime/action pieces.
   - Gamma neutral or slight warm (faces read healthier).
   - Gain slight warm (highlights catch a warm edge).
3. Lift the gamma wheel upward by 0.02 for "airy", downward by 0.02
   for "heavy". Most diffusion output benefits from -0.02 (more
   blacks, more contrast).

### Step 2 — Contrast and saturation

Add a second node:

1. Contrast: +15 to +25. Diffusion output is typically undercooked
   on contrast.
2. Saturation: -10 to -20 for cinematic looks; +10 to +25 for comic
   book / 90s cartoon looks.
3. Pivot: keep near 0.5; push higher to preserve highlights when
   pushing contrast.

### Step 3 — LUT (optional)

A third node with a creative LUT (Teal & Orange, Kodak 2383, anime-
style S-curves). Apply at 50–75% opacity by mixing node output.

### Step 4 — Per-shot match

Only if needed. If two shots with identical settings don't match,
it's usually because one has more ambient light than the other.
Fix on the gamma + offset wheels, not the LUT.

### Step 5 — Final checks

- Scopes: Waveform shouldn't crush below 0 or clip above 100.
- Vectorscope: skin tones on the "skin line" (the 12 o'clock-ish
  diagonal).
- Watch on phone + TV if you can. Monitors lie.

## Music-picture alignment — worked example

Scenario: 12-second short, 120 BPM track, 6 shots planned.

### Build the beat grid

In Resolve:

1. Drop music on audio track 1.
2. With the music selected, use the "audio markers on beats"
   approach: manually tap out Cmd/Ctrl+M on each downbeat as the
   music plays. You get a marker every 2 seconds.
3. Snap-to-markers: enable.

### Assign shots to bars

```text
Bar 1 (0.00-2.00s):  Shot A (establishing)
Bar 2 (2.00-4.00s):  Shot B (hero crouch, zoom)
Bar 3 (4.00-6.00s):  Shot C (leap + pan)
Bar 4 (6.00-8.00s):  Shot D (mid-swing parallax)
Bar 5 (8.00-10.00s): Shot E (villain reveal)
Bar 6 (10.00-12.00s):Shot F (impact + landing)
```

### Cut on beat, action on beat

Two separate things to align:

- **Cut points** land on downbeats. Snap the edit cursor to the
  marker and trim clip boundaries there.
- **Key visual events inside a shot** (impact, reveal, zoom
  apex) land on off-beats or the next downbeat. Slide the clip
  within its bar until the event aligns.

### The "pre-beat" trick

Sometimes a visual event hitting *just before* the downbeat
(~2 frames early) feels more urgent than on-beat. Try both. Music
editors call this an anacrusis.

## Continue to style guides

When you've internalised the patterns above — even without running
all of them — continue to [10-style-guides.md](10-style-guides.md).
