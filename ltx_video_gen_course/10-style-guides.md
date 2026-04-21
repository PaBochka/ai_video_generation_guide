# 10 — Style Guides: 90s SpiderMan & Anime

This module is practical: prompt templates, LoRA stacks, camera
vocabularies, and shot lists for each track. Use as a reference
throughout the final project.

---

## Track A — 1990s SpiderMan animation

Reference touchpoints: *Spider-Man: The Animated Series* (1994–1998),
*X-Men: The Animated Series* (1992–1997), comic book splash pages
from Todd McFarlane / Mark Bagley era.

### Visual language

- **Line work**: thick black outlines, variable weight, heavier on
  shadow side. Inked, not pencilled.
- **Shading**: cel-shaded, 2–3 tones max. No gradients. Hard shadow
  shapes.
- **Color**: saturated primaries, especially red/blue hero costumes
  against grey-blue urban backgrounds.
- **Lighting**: strong rim lights on heroes, blown-out sun/moon
  sources, dramatic under-lighting for villains.
- **Backgrounds**: painted look, softer detail than the characters.
  Buildings are slightly caricatured — leaning, elongated, angular.
- **Motion**: limited animation — key poses hold, in-betweens are
  few. Speed lines, motion blur streaks, impact stars.

### Camera vocabulary

- Crash zoom to a clenched fist / tight face.
- Whip pan between combatants.
- Dutch-angle action frames.
- Overhead "spider" shots looking straight down.
- Rooftop parallax: camera tracks fast, skyline scrolls.
- Low-angle hero stance against sky.

### Prompt scaffold

```
[TRIGGER: 90s-saturday-morning-animation],
[TRIGGER: spider-man-1994-style],
[SUBJECT + POSE: e.g. spider-man crouched on a gargoyle, one hand gripping stone, other hand raised mid-web-shot],
[CAMERA: low angle, slight dutch tilt, crash zoom on the mask],
[STYLE: thick black ink outlines, cel-shaded with two-tone shadow,
saturated red and blue, limited animation key-pose,
comic book splash-page composition],
[LIGHTING: moonlit new york skyline behind, warm street-light rim on costume],
[ATMOSPHERE: distant car horns implied, cold night, light fog]
```

Negative:

```
photorealistic, 3d render, smooth shading, gradient, blurry lines,
modern cgi, cinematic, photo, watermark, text, bad hands
```

### Camera-motion-specific prompt additions

- Crash zoom: `"crash zoom forward into extreme close-up, speed lines
  converging, motion blur on frame edges"`
- Whip pan: `"camera whip pans right, intense horizontal motion
  streaks, subject enters frame mid-pan"`
- Rooftop chase: `"camera tracks alongside at high speed, buildings
  stream past in the background, parallax layers"`
- Impact: `"full-frame impact flash, white and yellow, radial speed
  lines, comic onomatopoeia shape"`

### Pacing (24 fps, considered 12 fps feel)

- Hold on key poses 4–8 frames.
- In-between a punch over only 3–5 frames.
- Static "smear" frame for extreme motion — a single elongated frame.
- Reaction holds 8–16 frames for impact.

### Shot ideas for a 15-second SpiderMan short

```
00.0 - 02.0   establishing NY skyline, camera slow dolly up, moon visible
02.0 - 04.0   crouch pose on gargoyle, crash zoom into mask, eyes glint
04.0 - 06.5   leap off, whip pan follows, trailing webs
06.5 - 09.0   swinging through buildings, alternating parallax, cuts on each web
09.0 - 11.0   villain reveal, dutch-angle low shot, menacing silhouette
11.0 - 13.0   impact punch, full-frame hit flash, onomatopoeia
13.0 - 15.0   hero landing stance, low angle, logo-friendly composition
```

### LoRA stack (T2I key stills; SDXL or Flux base)

```
LoRA                              strength
90s-saturday-morning-animation    0.85 / 0.80
comic-book-inking                 0.45 / 0.35
(optional) spider-man-character   0.60 / 0.50
```

If no "spider-man-character" LoRA is available, describe the costume
in the prompt explicitly (red and blue spandex with black webbing
pattern, white eye lenses on red mask) and lean on the inking LoRA.

---

## Track B — Anime

Sub-styles you can target:

- **90s Madhouse/Gainax** — dense linework, painted backgrounds,
  cel shading, often serious register.
- **Modern KyoAni / Shaft** — softer shading, pastel palette,
  expressive character faces, refined compositing.
- **Ghibli-esque** — watercolour backgrounds, earthy palette, gentle
  motion, nature detail.

Pick one for the course. Blending sub-styles requires careful LoRA
balance and is not a beginner move.

### Visual language (example: 90s Madhouse)

- **Line work**: medium-weight black lines, slightly loose.
- **Shading**: two-tone cel shading, occasional hatch shadow.
- **Color**: muted but saturated, pushed magentas and teals, filmic
  grain.
- **Backgrounds**: hand-painted, softer than foreground, atmospheric
  perspective.
- **Faces**: large expressive eyes with multiple catch-lights,
  small noses, defined chin.
- **Motion**: limited animation with strong key poses. Iconic
  "anime running" (arms back, head forward). Hair and cloth over-
  animated on dramatic beats.

### Camera vocabulary (anime)

- Slow push-in on emotional beats.
- Pan across a static painted background (Hanna-Barbera trick, adopted
  widely in anime).
- Speed-line BG replacement during action (character in foreground,
  BG becomes radial/horizontal lines).
- Pillar-boxing for dramatic close-ups.
- Rack focus between foreground and background subjects.

### Prompt scaffold

```
[TRIGGER: madhouse-90s-anime],
[CHARACTER DESCRIPTION: e.g. a young woman with shoulder-length
crimson hair, amber eyes, black leather jacket, determined expression],
[POSE: standing on a rooftop, wind catching her hair, hand resting on
a sheathed blade],
[CAMERA: medium shot, slight low angle, subject slightly off-centre
to the right],
[STYLE: 1990s anime cel animation, hand-painted background, thick
ink lines on foreground, subtle filmic grain, muted saturated palette],
[LIGHTING: golden-hour backlight, long shadow across rooftop],
[ATMOSPHERE: distant city haze, crows on a wire, faint wind sound implied]
```

Negative:

```
photorealistic, 3d render, cgi, smooth shading, photo, blurry,
watermark, text, modern digital anime, moe, chibi
```

(Adjust negatives depending on whether you *want* moe/chibi.)

### Pacing (anime)

- On 2s (12 fps equivalent) for most character animation — hold each
  drawn pose for 2 frames at 24 fps.
- On 3s (8 fps) for really static limited-animation sequences.
- On 1s (24 fps) for hero animation: big fights, transformations,
  signature action.
- Emotional holds: 24–48 frames of near-still on an expressive face.

### Shot ideas for a 15-second anime short

```
00.0 - 03.0   pan across painted landscape, no characters, establish mood
03.0 - 05.0   character enters frame, medium shot, stops, looks to camera
05.0 - 06.5   close-up on eyes, subtle wind motion, emotional beat
06.5 - 09.0   action trigger: she draws blade in three cel beats
09.0 - 11.5   action wide: motion lines, blade arc, enemy off-frame
11.5 - 13.0   impact still: white flash, silhouette, held for 1s
13.0 - 15.0   return to medium, blade sheathed, camera slow pull-back
```

### LoRA stack (T2I key stills; SDXL or Flux base)

```
LoRA                              strength
madhouse-90s-anime                0.80 / 0.70
cel-shading-painterly             0.30 / 0.25
(optional) character-lora         0.70 / 0.60
```

### Notes specific to anime workflow

- **The "on 2s" feel is a pacing choice, not a model setting.**
  Generate at 24 fps and either drop every second frame in the editor
  or generate at 12 fps natively (if LTX-2.3 supports) and let the
  choppiness breathe.
- **Resist interpolation.** RIFE to 60 fps makes anime look like a
  soap opera. Keep 24 fps final for 90s work, 24 or 30 for modern
  anime.

---

## Track C — Mixed-media in the spirit of the reference video

The reference you linked is a mixed live-action + animated overlay
piece. Without rewatching it per-shot, the recipe most such pieces
follow is:

### Production approach

1. **Shoot or source a live-action plate.** Phone footage is fine.
   You want: a subject against a clean-ish background, some motion.
2. **Color and crop the plate.** Grade toward a consistent look —
   often crushed blacks, high contrast, slight desaturation.
3. **Rotoscope the subject** — either by hand, or with a segmentation
   model (SAM / RVM) to produce a per-frame mask.
4. **Generate an animated version** of a section using LTX-2.3 — either
   from the plate itself (vid2vid with edge conditioning) or
   independently from stills (I2V).
5. **Composite**: live-action plate + animated overlay, blended on
   mask, with a unifying grade.

### Visual language

- The hybrid look leans on contrast between media: live-action texture
  grain vs. clean animated line, live-action physics vs. animated
  exaggeration.
- Effects layers (particles, speed lines, scribble animation, comic
  onomatopoeia) unify the two media.
- A consistent color grade across both layers is essential — without
  it, the two media look like bad stock footage + bad clipart.

### LTX-2.3 side

- Use edge/depth conditioning (module 09) from the live-action plate
  to produce matching animation.
- Use I2V with a painted still of the subject for the "animated
  alternate take" shots that replace the live-action plate entirely.

### When mixed-media works

- As music video — motion driven by a beat, cuts covering
  imperfections.
- Short-form (15–60s) — the look is fatiguing over long durations.
- On stylised subjects — performers with distinctive silhouettes
  compose better than generic talking heads.

### LoRA stack

- Live-action plate: no LoRA, just grading.
- Animated overlay (T2I stage): a strong stylised LoRA — whichever of
  your style tracks you prefer. Think "scribble animation", "sharpie
  rotoscope", "comic inking".

---

## Track A deep dive — sub-styles within "90s action cartoon"

Lumping 1990s American action animation together obscures real
differences. Know which one you are chasing.

### A1 — Spider-Man: The Animated Series (1994)

- **Lines:** thick, clean, consistent weight.
- **Shadow:** hard cel shapes, 2 tones, shadow colour shifted towards
  cool blue-purple rather than black.
- **Palette:** saturated primary reds/blues on heroes, desaturated
  urban backgrounds (grey-blue, umber).
- **Motion:** moderate limited animation; hero action gets more
  drawings than civilians.
- **Iconic shots:** crouched poses on gargoyles, web-swing parallax,
  crash-zooms on the mask.

Prompt fragment library:

```text
POSE: "crouched atop gothic gargoyle, one hand gripping stone, head
       slightly turned, spider-sense alert pose"
POSE: "mid-web-swing, body fully extended, one arm gripping web line
       above, other arm trailing, cape of shadow behind"
POSE: "three-point landing on rooftop, one fist down, one knee down,
       head lowered, cape settling"
CAMERA: "low angle hero shot, skyline behind, slight dutch tilt"
CAMERA: "crash zoom into extreme close-up of mask, speed lines
         converging, motion streaks at edges"
LIGHTING: "cold moonlit rim from behind, warm street-light fill from
           below, deep blue sky gradient"
```

### A2 — X-Men: The Animated Series (1992)

- **Lines:** slightly rougher than Spider-Man TAS, more line-weight
  variation, more textured shading.
- **Palette:** bolder with more yellow/orange accents, heavier use
  of purple for villains and mutant energy.
- **Motion:** more frequent cel-animated effects (energy blasts,
  psychic overlays) than Spider-Man's "acrobat on city".
- **Iconic shots:** roster-line-up hero shots, pyrotechnic
  close-ups on powers, dramatic zoom-in on eyes igniting.

Prompt fragment library:

```text
POSE: "team of characters standing in confrontation stance, slight V
       formation, leader forward centre, flankers step behind"
POSE: "unleashing energy blast, arms forward, palms radiating
       particles, hair whipped backward by aura"
CAMERA: "slow push in on eyes as they ignite with energy glow,
         peripheral darkens"
LIGHTING: "character self-lit by power aura, backlights bleeding
           into the scene, colour cast from power temperature"
```

### A3 — Batman: The Animated Series (1992)

A quieter register. Worth knowing because the techniques produce the
most mood per generation.

- **Lines:** medium weight, extremely confident (minimal).
- **Shadow:** heavy black shadows; art direction famously "Dark Deco".
- **Palette:** desaturated, noir-leaning; reds and yellows appear
  rarely and pop when they do.
- **Motion:** remarkably restrained — iconic silent rooftop glides,
  long holds on silhouette.
- **Iconic shots:** silhouettes on rooftops, gargoyle perches (again),
  rain-soaked streetlight pools, shadow falling across a face in
  two beats.

Prompt fragment library:

```text
POSE: "silhouetted figure on rain-slick rooftop, cape flowing in
       wind, back three-quarters to camera"
POSE: "face half-hidden in shadow, only the eyes catching a rim of
       light, dramatic low key"
LIGHTING: "noir two-point lighting, strong key from one direction,
           rim from opposite, crushed blacks"
ATMOSPHERE: "heavy rain, puddles, neon sign reflections, steam from
             grates, wet asphalt"
```

### A4 — Superman: The Animated Series (1996)

- **Lines:** cleaner and thinner than Batman TAS, bolder than
  Spider-Man TAS.
- **Palette:** brighter, more optimistic, heroic primary colours
  well-saturated even in day scenes.
- **Motion:** more fluid than Batman; flight and strength-feats get
  generous animation.
- **Iconic shots:** hovering hero against clouds, low-angle
  "statuesque" portraits, skyline fly-pasts.

Prompt fragment library:

```text
POSE: "hovering mid-air, arms at sides, cape billowing upward from
       below, serene expression, looking off to horizon"
CAMERA: "tracking alongside flight, metropolis skyline receding
         into distance, clouds streaming past"
LIGHTING: "bright midday sun, minimal shadow, saturated blue sky,
           white highlights on costume"
```

## Track A pitfalls — what will go wrong

- **Over-photorealism.** LTX-2.3 and Flux both default to photoreal
  skin textures; the 90s TV look needs flat, cel-shaded skin. Lean
  hard on the inking and cel LoRAs; reinforce in negative prompt
  (`photoreal, 3d render, skin texture pores`).
- **Modern superhero movie colour grading.** Output drifts toward
  teal-and-orange cinematic. Anchor with prompt: "saturated primary
  colours, Saturday morning palette, not cinematic".
- **Over-animation.** If you RIFE-interpolate the output to 60 fps,
  it stops looking 90s TV. Keep 24 fps. Consider dropping to 12 fps
  (on 2s) for sections.
- **Character proportions.** TV animation proportions are
  stylised (bigger heads, shorter legs than real). Without a
  character LoRA, prompts drift toward realistic proportions.
  Explicitly prompt: "stylised cartoon proportions, slightly
  exaggerated musculature, large expressive eyes behind mask".

## Track A — five more shot ideas

```text
1. Cold open: empty city skyline, slow pan left to right, single window lit,
   silhouette visible inside. 3s. No hero yet.

2. Newsstand rack spins, headlines flash ("BANK ROBBERY", "HERO SPOTTED"),
   classic comic-transition device. 2s, hard cuts.

3. Villain monologue close-up, lit from below, fan blowing cloth,
   eyes narrow during key line. 4s. Hold on eyes at end.

4. Web-swing passing between two buildings, camera locked on a static
   window — hero swings past, window reflection flashes, gone. 1.5s.

5. End-shot: hero crouched on rooftop, pulling up the mask from
   chin-level, revealing the civilian identity as dawn breaks. 5s.
   Slow zoom in to their eyes.
```

## Track A — character design template (for T2I key still)

Use this to generate consistent character reference. Fill in blanks.

```text
PROMPT TEMPLATE:
90s-saturday-morning-animation, [CHARACTER_TRIGGER],
[AGE] [GENDER] hero, [BUILD: athletic / lean / bulky],
wearing [COSTUME: fitted spandex bodysuit in red and blue with
black webbing pattern / sleeveless yellow and black tactical suit /
long dark cape over grey bodysuit],
[MASK/FACE: full face mask with white eye lenses / half mask
exposing lower face / no mask, square jaw, confident expression],
[HAIR: messy black hair falling over forehead / slicked back auburn /
short white crew cut],
stylised cartoon proportions, slightly exaggerated musculature,
thick black ink outlines, two-tone cel shading, saturated palette,
[POSE_FROM_POSE_LIBRARY],
[CAMERA_FROM_CAMERA_LIBRARY],
[LIGHTING_FROM_LIGHTING_LIBRARY],
[BACKGROUND: urban night rooftop / city skyline / industrial
interior / comic book splash page background]

NEGATIVE:
photoreal, 3d render, cgi, gradient shading, smooth skin,
modern superhero movie grading, teal and orange, blurry,
watermark, text, modern digital anime
```

## Track B deep dive — sub-styles within "anime"

"Anime" is an umbrella covering wildly different visual registers.
Pick one. Blending is a trap.

### B1 — 1990s Madhouse (dark drama register)

Reference: *Perfect Blue*, *Ninja Scroll*, early Satoshi Kon.

- **Lines:** crisp, confident, denser than daytime anime.
- **Shading:** cel shading with occasional painted detail; frequent
  hatch shadows on faces.
- **Palette:** muted but saturated, pushed teals and magentas,
  filmic grain visible.
- **Characters:** realistic proportions (seven-to-eight heads tall),
  subtle face geometry, occasional "Kon tells" — tight close-ups
  with unflinching eye contact.
- **Backgrounds:** hand-painted, atmospheric, often with a single
  striking colour source (neon sign, distant fire).

Prompt fragment library:

```text
POSE: "standing in a narrow alley, shoulders slightly hunched,
       hands in coat pockets, head turned to look behind"
FACE: "close-up, eyes slightly narrowed, mouth a thin line, catching
       distant neon as highlights on eyes, subtle grain across face"
CAMERA: "medium shot, slightly off-centre right, negative space on
         left, shallow depth of field"
LIGHTING: "single neon source off-frame right, magenta rim, cool
           shadow side, ambient haze"
```

### B2 — 1990s Ghibli (naturalistic pastoral)

- **Lines:** soft, flowing, less rigid than Madhouse.
- **Shading:** watercolour washes mixed with cel, lots of air and
  translucency.
- **Palette:** earthy greens, sky blues, warm yellows, natural
  wood/stone tones.
- **Characters:** rounded features, large expressive eyes, less
  stylised than moe but more than realism.
- **Backgrounds:** painted with visible brush texture, landscape-
  forward.

Prompt fragment library:

```text
POSE: "young character standing at a hillside fence, hands resting
       on the wood, hair lifting in breeze, looking into a valley"
CAMERA: "medium wide, eye-level, stable, slight backlight"
LIGHTING: "golden hour sun, warm light on grass, long shadows, soft
           rim on hair"
BACKGROUND: "rolling hills, scattered farmhouses with red roofs,
             distant mountains hazy, clouds hand-painted"
```

### B3 — Modern KyoAni / Shaft (contemporary clean)

- **Lines:** thinner than 90s, very precise.
- **Shading:** cel with occasional soft gradient accents, digital-
  clean highlights.
- **Palette:** pastels, pushed blues and pinks, occasional neon
  accents.
- **Characters:** larger eyes, expressive micro-movements, detailed
  hair.
- **Backgrounds:** photoreal-leaning when composited with characters.

Prompt fragment library:

```text
POSE: "seated on a window sill, one leg tucked, chin resting on
       knee, looking out at city"
FACE: "large expressive eyes with multiple catch-lights, small nose,
       soft mouth, subtle blush on cheeks"
LIGHTING: "soft sunlight through window, bloom on highlights,
           ambient pastel, slight chromatic aberration at edges"
```

### B4 — Shonen action (Trigger / Studio Bones register)

- **Lines:** kinetic, thicker on action frames, thin on normal.
- **Shading:** bold cel with frequent "impact frame" over-drawings.
- **Palette:** high saturation, strong colour contrast, lens flares.
- **Motion:** the most animated of anime sub-styles, frequent on-1s
  for hero action, wild smear frames.

Prompt fragment library:

```text
POSE: "mid-air in full action pose, fist cocked back, body fully
       torqued, energy aura crackling, hair whipping from motion"
CAMERA: "dutch-angle wide, low-angle, speed-line background
         replacement"
EFFECTS: "energy particles trailing from limbs, lens flare from
          background, impact frame overlay"
```

## Track B pitfalls

- **Style collision with proportion.** A "modern clean" style LoRA
  + a 90s character LoRA = proportion mismatch. Use modern-proportion
  characters with modern-clean style; use stockier characters with
  90s styles. Mixing is doable but you must know what you're fighting.
- **Eye catch-lights drift.** Without a character LoRA, eye design
  varies shot-to-shot. Be explicit: "circular highlight upper-left
  of each eye, smaller highlight lower-right, no other reflections".
- **Hair interpolation in I2V.** Anime hair is semi-rigid but
  expressive. LTX-2.3 tends to over-animate it into physical-simulation
  realism. Prompt: "anime hair physics, stylised not realistic, moves
  in clumps not strands".
- **Backgrounds going photoreal.** Anime backgrounds are painted;
  Flux/LTX-2.3 default toward photographic. Prompt: "hand-painted
  background, visible brushwork, not photographic".

## Track B — five more shot ideas

```text
1. Cherry blossoms falling past a static close-up of a character's face,
   focus racks from petals to eyes, 4s, slow.

2. Train window POV, countryside streams past, character reflection on
   glass ghosts across the landscape, 5s, continuous.

3. Sword draw in three clean beats: hand on hilt / blade half drawn with
   light flash / fully drawn above head, pose held. 1.2s total, no
   interpolation, beats on frames 0, 8, 24.

4. Dinner-table scene, soft lamp lighting, pan across three bowls of
   food before landing on a character's contemplative face. 4s.

5. Rooftop-at-dawn: two characters sit, one speaks facing forward,
   other listens profile, sky cycles from indigo to pink behind. 6s.
```

## Track B — character design template

```text
PROMPT TEMPLATE:
[STYLE_TRIGGER], [CHARACTER_TRIGGER_IF_ANY],
[AGE] [GENDER], [HEIGHT_IMPRESSION: petite / average / tall],
[HAIR: shoulder-length crimson cut in layers, side-swept bangs /
       short black with one streak of white / long braided blonde],
[EYES: large amber with multiple catch-lights / sharp green, narrow
       eyebrows / pale blue with dark outer rim],
[OUTFIT: school uniform with modifications / traditional kimono with
         modern tailoring / black leather jacket, denim, boots],
[SIGNATURE_PROP: sheathed katana at left hip / glowing crystal
                 pendant / worn leather-bound book],
stylised anime proportions [7-to-8 heads / 6-to-7 heads / chibi 4-heads],
[CEL_SHADING_DESCRIPTION_MATCHING_SUB_STYLE],
[HAND-PAINTED BACKGROUND / CLEAN DIGITAL BACKGROUND],
[POSE_FROM_POSE_LIBRARY],
[CAMERA_FROM_CAMERA_LIBRARY],
[LIGHTING_FROM_LIGHTING_LIBRARY]

NEGATIVE:
photoreal, 3d render, cgi, smooth photographic skin, western cartoon,
moe if not wanted, chibi if not wanted, watermark, text
```

## Track C deep dive — mixed-media

The reference video you linked belongs to a broad family of mixed-
media work: live-action-plus-scribble, rotoscope-over-photography,
scribble-animated overlay on photographic plates. A few concrete
sub-styles within this umbrella.

### C1 — Sharpie rotoscope (scribble-over-footage)

Reference: music videos in the A-ha "Take On Me" lineage, many
indie artists' hand-drawn-over-film pieces.

- **Lines:** single-colour ink over footage, frame-by-frame jitter.
- **Fill:** mostly outline, occasional loose fill.
- **Colour:** black-on-white or a single accent colour.
- **Motion:** "boiling" jitter from redrawn lines each frame is
  intentional and part of the aesthetic.

Pipeline:

1. Live-action plate, graded high-contrast.
2. Per-frame edge extraction (Canny / HED).
3. Stylise via SDXL img2img with a sharpie-rotoscope LoRA, denoise
   0.6–0.75 to encourage loose redrawing.
4. Composite the stylised output back over a processed plate, or
   use as-is.

### C2 — Painted-over-live-action (Linklater style)

Reference: Richard Linklater's *A Scanner Darkly*, *Waking Life*.

- **Lines:** painterly, thicker, fewer than sharpie style.
- **Fill:** full colour painted fills, texturing from brushwork.
- **Motion:** slightly floaty, colours shift frame-to-frame.

Pipeline:

1. Live-action plate.
2. LTX-2.3 vid2vid with painterly style LoRA at denoise 0.5.
3. Optional temporal smoothing to reduce colour jitter if too busy.

### C3 — Scribble overlay with live-action core

The animation lives *on top of* the live-action, not replacing it.
Classic for music videos and title sequences.

- **Plate:** live-action, minimally touched.
- **Overlay:** hand-animated (or diffusion-generated) looping
  elements — doodles, effects, text, rotoscoped accents around a
  subject.
- **Composite:** overlay on top of plate with screen/add blend
  modes or alpha masks.

Pipeline:

1. Shoot the plate.
2. Separately generate overlay elements — either hand-drawn or via
   diffusion on black background.
3. Composite in Resolve Fusion or After Effects, with motion
   tracking if overlays need to follow subjects.

## Track C pitfalls

- **Flicker overload.** Frame-by-frame stylisation is inherently
  boilier than modern diffusion expects. A little flicker is the
  aesthetic; too much is nausea. Control via temporal smoothing
  nodes or by generating every *second* frame and interpolating.
- **Colour grade mismatch.** Plate and overlay grade differently,
  making the composite look like two layers stuck together. Apply
  a unifying grade *after* composite.
- **Overlay lag.** Hand-animated overlays that don't track the
  subject read as amateurish. Either commit to tracked overlays
  (more work) or to floating-independent overlays (different
  aesthetic, also legitimate).
- **Reference footage limitations.** A shaky, low-light phone video
  will produce a shaky, low-light stylised result. Fix the input
  first, every time.

## Track C — five shot ideas

```text
1. Live-action subject walking, sharpie rotoscope runs frame-by-frame
   on them while the background stays photographic. 4s.

2. Full-screen painted-over scene from a photograph, slow push-in,
   camera move drives the motion (subject static). 3s.

3. Live-action close-up of face; animated thought-bubble doodles
   orbit the head, subject remains photoreal. 5s.

4. Rapid cross-fade between live-action and painted version of same
   frame every 8 frames, rhythmically. 2s.

5. Photographic wide shot, animated subject drawn entering frame
   from the right, stops centre, the photograph "zooms" by crop.
   3s.
```

## Cross-track camera-shot reference card

Save this card. Reuse freely.

| Effect | Prompt phrase | Notes |
|---|---|---|
| Crash zoom | `crash zoom into [subject], speed lines converging` | Hits hard; use sparingly |
| Whip pan | `camera whip pans [left/right], motion streaks` | Good cut cover |
| Dolly in | `slow dolly forward, subject centred, lens slightly compresses` | Tension builder |
| Dolly out | `slow pull back, subject becomes smaller against [environment]` | Emotional distance |
| Overhead | `top-down birds-eye shot, subject below` | Disorients nicely |
| Dutch angle | `dutch angle tilted right, horizon slanted` | Action / unease |
| Rack focus | `rack focus from foreground to background, shallow depth of field` | Hard for LTX-2.3 without strong prompt |
| Static / locked | `static locked-off camera, no movement` | Lets subject motion breathe |

Continue to [11-final-project.md](11-final-project.md).
