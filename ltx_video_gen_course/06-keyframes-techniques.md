# 06 — Keyframes & Temporal Control

This is the central module of the course. Good video diffusion is 70%
keyframe design and 30% sampler fiddling. Internalise the patterns here
and the final project is straightforward.

## The keyframe mindset

Stop thinking "I will generate a 10-second clip". Start thinking:

- **Beats.** What are the 3–6 poses that define this shot?
- **Pairs.** Each pair of adjacent beats becomes an I2V first/last
  generation.
- **Stitches.** Pairs are concatenated with a small overlap, crossfaded
  or hard-cut.

A 10-second shot might be 4 pairs of 2.5 seconds each. Each pair is
cheap to regenerate. One pair goes wrong? Redo one pair, not the whole
shot.

## The keyframe production pipeline

```
   Storyboard (paper/figma/whatever)
            |
            v
   Per-beat still images (T2I with your style LoRA)
            |
            v
   Curate + pose-fix (maybe inpaint a hand, tweak composition)
            |
            v
   I2V first+last for each consecutive pair
            |
            v
   Stitch in a video editor or with concat nodes
            |
            v
   Frame interpolate (RIFE) if you want higher fps
            |
            v
   Upscale (optional, module 09)
            |
            v
   Final edit with audio
```

## Designing key poses

Rules of thumb:

- **Read in silhouette.** If the silhouette of the pose is ambiguous,
  so is the motion between it and the next pose. Strong silhouettes
  animate cleaner.
- **Change only what should change.** Between two consecutive keyframes,
  only the moving parts should differ. Same outfit, same lighting, same
  camera framing — unless the shot deliberately cuts.
- **Respect physics loosely.** The model is a physics student who
  skipped most lectures. Weight, balance, follow-through improve if
  your key poses already respect them.
- **Exaggerate.** Animation poses are more extreme than life. A
  SpiderMan punch pose is not "arm extended"; it is "full body
  torqued, back foot pivoting, cape mid-snap".

## Consistency across keyframes

The #1 problem is visual consistency between generated stills. Tools:

1. **Same seed, same sampler, same style LoRA, same character LoRA.**
2. **Same prompt scaffold with only the pose/action tokens varying.**
   Build a prompt template:
   ```
   [CHARACTER_TOKEN], [POSE_TOKEN], [CAMERA_TOKEN],
   [STYLE_TOKEN], [LIGHTING_TOKEN], [NEGATIVE_SCAFFOLD]
   ```
   Then per beat, only `POSE_TOKEN` changes.
3. **IP-Adapter / image prompting** on the T2I stage. If you have a
   good reference image of the character, feed it to an IP-Adapter to
   lock identity. (Requires SDXL/Flux-side work before LTX-2.3.)
4. **Inpainting.** Generate once, then inpaint only the pose change.
   Expensive but airtight.

## Timing and pacing

Video has a rhythm. Map it before you generate:

```
shot 01 | 00.0 - 02.5 | wide | establish setting
shot 02 | 02.5 - 04.0 | medium tracking | character enters
shot 03 | 04.0 - 05.0 | close-up | reaction beat, 1s hold
shot 04 | 05.0 - 08.0 | action wide | main action
shot 05 | 08.0 - 10.0 | close-up | aftermath, 2s linger
```

Each row = one I2V generation (or pair). Knowing your timing up front
stops you from over-generating pretty-but-unusable clips.

Typical durations:

- **Establishing shot:** 2–4 seconds. Enough to register the scene.
- **Dialogue / reaction:** 1–2 seconds per beat.
- **Action beat:** 0.5–2 seconds. Fast cuts read fast.
- **Linger / emotional hold:** 2–3 seconds.

LTX-2.3 generates in 97/161/etc. frame increments. Round your timing to
the nearest valid length.

## First-frame-only vs first+last

| Use first-frame only when | Use first+last when |
|---|---|
| The motion is ambient (wind, breathing, small gestures) | The motion is a specific, terminating action |
| The end pose doesn't matter | The end pose is the next shot's beginning |
| You want the model's creativity on where motion goes | You want to hit a choreographed beat |

Mixing them in the same project is normal. Ambient shots can drift; a
punch-landing shot cannot.

## Overlap and stitching

When two I2V clips share a frame (the end of clip A = the start of
clip B), you can do one of three things:

1. **Hard cut.** Drop the shared frame from one of them. Works if the
   match is exact.
2. **Short crossfade.** 2–4 frames of overlap, crossfade in the editor.
   Forgives small mismatches (lighting, position).
3. **Re-generate the overlap.** Take the last N frames of A and the
   first N frames of B and run a short I2V between them using first+last
   of those frames. Expensive but seamless.

For the course final, (2) is the right default.

## Frame interpolation vs native frame rate

Generating at 24 fps and post-interpolating to 48 or 60 fps is legitimate:

- Cheaper than generating at higher fps natively.
- RIFE / FILM do a good job on the smooth, already-coherent output of
  a modern video diffusion model.
- The "TV anime look" often *wants* 24 fps (or 12 fps on 2s). Do not
  auto-interpolate anime-style work — it destroys the intended feel.

Default by style:

- Photoreal / live-action pastiche: generate 24 fps, interpolate to 48.
- 90s SpiderMan / cel animation: generate 24, keep at 24. Consider
  even dropping to 12 fps on selected shots for the "limited animation"
  rhythm.
- Modern anime: 24 fps, interpolate selectively on camera moves and
  smooth sections, keep key poses crisp.

## Camera-driven vs subject-driven motion

Decide which dominates each shot:

- **Camera-driven:** subject is relatively still, camera moves (dolly,
  whip-pan, crash-zoom). Prompt the camera explicitly. Works well in
  T2V mode.
- **Subject-driven:** camera is locked or smooth, subject does a
  specific action. Use I2V with first+last for choreography.
- **Both:** possible but harder. The model splits budget; motion quality
  drops. Save this for hero shots where you will iterate many times.

## Exercise — Keyframe a 3-beat action

Beat plan for a 6-second shot:

1. **Beat 1 (0.0s):** hero crouched, looking over shoulder, shadowed.
2. **Beat 2 (2.5s):** hero mid-leap, silhouetted against rooftop sky.
3. **Beat 3 (5.0s):** hero landing on next rooftop, one hand on the
   ground, cape settling.

Produce:

- Three T2I stills for beats 1, 2, 3 (same seed, same LoRAs, same
  prompt scaffold).
- Two I2V generations: (1->2) and (2->3), each 2.5 seconds (61 frames
  at 24 fps, rounded to nearest LTX-2.3-valid length).
- Concatenate with a 2-frame crossfade.

This is, structurally, 80% of the capstone.

## Pose descriptor vocabulary

Prompting a pose is its own skill. Build a private dictionary. Starter
entries:

### Body pose

| You want | Try |
|---|---|
| Action-ready | "low centre of gravity, knees bent, weight forward on back foot, shoulders squared" |
| Heroic landing | "three-point landing, one knee and one fist on ground, other arm flared out, head down" |
| Cowering | "knees drawn up, arms wrapping head, spine curled, small silhouette" |
| Mid-leap | "fully extended, both feet off ground, back arched, arms trailing, hair/cape streamed behind" |
| Running flat-out | "leaning forward 30 degrees, arms pumping, one foot barely touching ground, mouth open for breath" |
| Contemplative | "one hip cocked, weight on one leg, hand on chin or at hip, chin slightly down, eyes off-camera" |
| Drawing a weapon | "right hand gripping hilt at left hip, left hand steadying sheath, torso pre-torqued away from blade" |

### Face & gaze

| You want | Try |
|---|---|
| Resolve | "jaw set, brow slightly lowered, mouth a thin line, eyes fixed on a distant point" |
| Surprise | "eyes wide, pupils small, eyebrows raised and apart, mouth slightly open" |
| Anger | "brow lowered and pinched, teeth bared or jaw clenched, nostrils flared" |
| Sadness without crying | "eyes lowered, mouth closed and soft, head tilted 5 degrees, shoulders dropped" |
| Smirk (villain) | "one corner of mouth raised, eyes half-lidded, head tilted back slightly" |

### Camera-relative positioning

| You want | Try |
|---|---|
| Close-up, off-centre | "head and shoulders fill right third of frame, left two-thirds negative space for a title" |
| Over-the-shoulder | "camera behind subject A, looking past their right shoulder at subject B across the room" |
| Hero shot | "subject centred, low angle, silhouetted against a bright sky, full body in frame" |
| Reveal | "subject enters frame from left edge, camera was empty for 1 second prior" |

Use these as LEGO blocks inside the prompt scaffold. A clear pose
descriptor is worth ten poetic adjectives.

## First+last frame — gotchas and fixes

The first/last-frame workflow is powerful but has a narrow operating
range. Things that go wrong and how to recognise them:

### Morph instead of motion

**Symptom:** middle of clip looks like a smooth cross-dissolve between
the two images instead of real motion (subject seems to glide, parts
fade in/out of existence).

**Causes + fixes:**

- End image composition too different from start. Fix: regenerate the
  end image with the same seed and only the intended-to-move parts
  changed; inpaint to hold static parts identical.
- Action span too long. If the prompt is "draws the sword" but you
  set a 6s clip, the model interpolates a 6-second draw, which is
  unphysical. Fix: drop to 1.5–2.5s and chain with another clip.
- Prompt too passive. "From standing to sword drawn" describes states,
  not the action. Fix: "sharp pull from sheath, right elbow drives
  back, blade clears hip and rises to shoulder".

### Too-fast snap to end pose

**Symptom:** motion completes in the first 30% of the clip, then the
subject just sits in the end pose doing nothing.

**Causes + fixes:**

- End-frame conditioning weight too high. Lower it in the sampler
  (look for `end_strength`, `last_frame_influence`, etc., naming varies).
- Clip length too short for the action. Give it breathing room.

### Drift from start

**Symptom:** first 0.5s is on-model, then face/hands slip off the
character.

**Causes + fixes:**

- Start-frame conditioning decays too quickly. Raise start strength
  if your sampler exposes it.
- Character not locked by text. Add three concrete identity tokens
  (e.g. "green eyes, silver streak in black hair, scar on right
  cheekbone") to the prompt.
- The start image itself is too ambiguous (hair over eyes, face
  turned away). Fix it in T2I first.

## Worked example — a 12-second SpiderMan beat, fully specified

A full breakdown you can steal, adjust, and run. Target: 12s at 24fps,
three shots, keyframe-driven.

### Shot 1 — gargoyle crouch, crash-zoom on mask (0–4s)

**Beat A (frame 0):** wide establishing. Gargoyle at night, skyline
behind.

Prompt (T2I):
```
90s-saturday-morning-animation, spider-man-1994-style,
wide shot, spider-man crouched on a weathered stone gargoyle,
knees bent, one hand on stone, head turned slightly left and down,
full body in frame, new york skyline behind with scattered lit windows,
deep blue night sky, full moon partially behind buildings,
thick black ink outlines, two-tone cel shading, saturated red and blue
on costume, cold rim light on shoulders and mask, slight fog in distance,
comic book splash page composition
```

**Beat B (frame 96, ~4s):** extreme close-up on mask, eyes glinting.

Prompt (T2I):
```
90s-saturday-morning-animation, spider-man-1994-style,
extreme close-up on spider-man mask, only eyes and forehead in frame,
eye lenses glinting with caught moonlight, thick black web pattern,
low angle, dramatic under-rim light from below,
thick ink outlines, flat red with hard black shadow,
speed lines along frame edges suggesting incoming crash zoom
```

**I2V prompt (Beat A -> Beat B), 4s, first+last:**
```
crash zoom from wide shot into extreme close-up on the mask,
camera accelerates forward, buildings streak past in motion lines,
gargoyle blurs out of frame, mask eyes sharpen into focus,
speed lines converge toward centre, single smooth push-in, no cuts
```

Settings: 97 frames @ 24fps, CFG 3.5, Euler 25 steps. If motion feels
stuttery, reduce to 65 frames @ 24 (2.7s) and accept a shorter shot.

### Shot 2 — web-swing sequence (4–9s)

Break into two sub-clips so you have a cut for rhythm:

**Sub-clip 2a (4–6.5s):** takeoff from gargoyle.

Beat A: mask close-up (reuse last frame of Shot 1).
Beat B: mid-leap silhouette against moon.

Prompt pair similar to above, I2V motion:
```
spider-man launches off gargoyle toward camera-right, body uncoils
explosively from crouch, web line shoots from right wrist upward
out of frame, cape of shadow trails behind, camera whip-pans right
to follow, moon swings across frame
```

**Sub-clip 2b (6.5–9s):** swinging parallax.

Beat A: mid-swing, wide shot.
Beat B: arrival on darker rooftop.

I2V motion:
```
continuous web-swing, foreground buildings rush past right to left,
parallax distant skyline moving slowly opposite direction,
spider-man silhouetted in centre, web line arcing upward off-frame,
camera tracks alongside at matched speed, constant horizontal motion
```

### Shot 3 — confrontation and impact (9–12s)

**Beat A:** hero lands crouched on dark rooftop, villain silhouette
upstage.
**Beat B:** impact frame — white-yellow flash, hero's fist extended,
villain silhouette knocked back.

Cut between them hard, do not I2V interpolate (the impact should
snap). Generate each as short I2V of their own (1.5s each) with
internal motion, then hard-cut.

Sub-clip 3a motion prompt: `landing impact, dust plume radiating
outward, cape settling downward, villain silhouette turns toward camera`

Sub-clip 3b motion prompt: `fist-strike connects, full-frame white-
yellow impact flash, radial speed lines, onomatopoeia shape forming,
villain silhouette reeling back`

### Timing summary

```
00.0 - 04.0   Shot 1 (I2V first+last, 97 frames)
04.0 - 06.5   Shot 2a (I2V first+last, 61 frames)
06.5 - 09.0   Shot 2b (I2V first+last, 61 frames)
09.0 - 10.5   Shot 3a (I2V first-only + motion, 37 frames)
10.5 - 12.0   Shot 3b (I2V first-only + motion, 37 frames)
```

Round frame counts to your LTX-2.3 build's valid lengths (often multiples
of 8 + 1). Total project: 5 generations. Budget ~2 regenerations per
shot = 15 generations. That is an evening at ~3 minutes each on a 24GB
card.

## Stitching cookbook

Concrete editor recipes. Assume DaVinci Resolve terminology.

### Recipe 1 — Hard cut on impact

When: action beats (punches, gunshots, transitions).

Steps:
1. Place clip A ending on the frame just before impact.
2. Place clip B starting on the impact flash frame.
3. No transition. The cut *is* the impact.
4. Optional: insert a 1-frame white solid between them for extra punch.

### Recipe 2 — Crossfade on ambient

When: continuous motion, one shot flowing into another at the same
location/mood.

Steps:
1. Overlap clips by 4–8 frames.
2. Apply a linear cross-dissolve across the overlap.
3. If the two clips have a colour mismatch, add a temporary grade
   node across the overlap that averages them.

### Recipe 3 — Match-cut on motion

When: subject action in A continues in B with a different angle or
location.

Steps:
1. Find the "peak" of the motion in A (e.g. apex of a jump).
2. Cut to B on the matching peak frame of the new angle.
3. No transition. The brain fills in the continuity.

### Recipe 4 — Re-generate the seam

When: the transition must be invisible and neither cut nor crossfade
hides the discontinuity.

Steps:
1. Grab the last 15 frames of clip A and the first 15 frames of clip B.
2. Export them as two stills: A's last and B's first.
3. Run I2V first+last on those two stills for ~1 second.
4. Splice the new 1s bridge where the seam was.
5. Trim A to end before the old seam, B to start after. Slot the
   bridge in.

This is the most expensive, slowest option. Reserve for hero seams.

## Timing templates by genre

Handy starting rhythms. Genre A values go into your shot-length plan
before you pick a prompt.

### 90s action cartoon

| Beat type | Duration |
|---|---|
| Establishing wide | 2.0–3.0s |
| Hero stance hold | 1.0s |
| Crash zoom | 0.8–1.2s |
| Action swing/leap | 2.0–3.0s (may sub-cut) |
| Impact frame | 4–8 frames held |
| Reaction close-up | 0.8–1.5s |
| End-shot hero pose | 2.0–3.0s |

### Anime, 90s drama register

| Beat type | Duration |
|---|---|
| Landscape pan (empty of character) | 3.0–5.0s |
| Character entry | 1.5–2.5s |
| Emotional close-up | 2.0–3.5s |
| Dialogue beat (no dialogue, just screen time) | 1.5–3.0s |
| Action trigger | 1.0–2.0s |
| Motion-line action | 1.5–2.5s |
| Impact still (held frame) | 0.8–1.2s |
| Pull-back resolution | 3.0–4.0s |

### Music video / mixed-media

Cuts should land on beats. At 120 BPM, a beat is 0.5s. Typical cutting:

- Hard cuts on every downbeat (2s intervals at 120 BPM) for steady
  sections.
- Every half bar (1s) for energetic sections.
- Every 2 bars (4s) for emotional breathers.

Measure the song first. Build a beat grid in your editor. Drop shots
onto beats.

## Hand-drawn storyboard → diffusion keyframes

If you draw storyboards (even rough thumbnails), this is the cleanest
pipeline:

1. Draw your storyboard panels. Stick figures are fine.
2. Photograph or scan each panel.
3. Use each drawing as an **SDXL img2img** reference with low denoise
   (0.55–0.70) and your style LoRA stack.
4. The output will respect your composition, blocking, and camera
   while inheriting the LoRA style.
5. Use those outputs as LTX-2.3 I2V keystills.

This preserves *authorial composition* — the #1 thing pure text
prompting tends to lose.

## What to save before moving on

- `04-i2v-keyframe-pair.json` — your workhorse first+last workflow.
- `04b-i2v-firstonly-ambient.json` — first-frame-only variant for
  ambient shots.
- A 6–12 second stitched clip from the worked-example exercise.
- Your private pose descriptor dictionary (just a text file of
  phrases that worked). Add to it every project.
- Notes on what broke and how you fixed it. Keyframe debugging
  intuition only comes from reps.

Continue to [07-lora-fundamentals.md](07-lora-fundamentals.md).
