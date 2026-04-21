# Module 4 — Shot Workflows

This module is the bridge between planning (modules 1–3) and post-production (module 5). For each shot type from your shot list, this module gives you a concrete LTX-2.3 recipe.

All recipes assume you have:
- A working LTX-2.3 setup per workflow guide §9.
- Familiarity with workflow guide §10 (LoRAs, IC-LoRAs, anchors, looping sampler).
- Your style preamble from module 2.
- Your character description / sheet from module 3.

If a recipe says "use Recipe A character technique" it refers to module 3.

> **Related ComfyUI Guide sections** — [ComfyUI Video Workflow Guide](../ComfyUI_Video_Workflow_Guide.md):
> - **§9.4** — canonical two-stage T2V node chain (the engine every recipe here plugs into).
> - **§9.5** — I2V chain with `LTXVPreprocess` + `LTXVImgToVideoInplace`. Used by the close-up, OTS, and continuation recipes.
> - **§9.7** — sampler/scheduler presets per stage (dev vs. distilled, `LTXVScheduler` vs. `ManualSigmas`).
> - **§9.12** — worked 4-second cinematic T2V example (24 GB card). Baseline settings for most recipes in this module.
> - **§10.2** — camera-control LoRAs keyed to shot types (dolly-in → push-in CU, jib-up → rising EST, static → dialogue MS).
> - **§10.3** — IC-LoRAs (depth/pose/Canny/motion-track). Used for action shots that need structural control.
> - **§10.5** — `LTXVLoopingSampler` for shots beyond ~10 seconds and for the title/loop recipe.
> - **§10.9** — compatibility quick-reference (what you can stack with what). Check before combining camera LoRA + IC-LoRA + anchor in the same recipe.

---

## 4.1 General shot recipe template

Every shot, regardless of type, follows this structure:

```
[STYLE PREAMBLE]   (from module 2 — copy-paste, never paraphrase)
[CHARACTER DESCRIPTION(S)]   (from module 3 — copy-paste)
[SHOT FRAMING]   (e.g., "Medium close-up of...")
[ACTION]   (what happens in this shot)
[CAMERA]   (camera move, often matching a camera LoRA)
[LIGHTING + ATMOSPHERE]   (specific to this shot)
[SOUND CUE]   (foreground sound + ambience)
```

LoRA stack:
- Distilled-speed LoRA (always, per workflow guide §9 & §10)
- Style LoRA (if you have one, per module 2)
- Camera LoRA (per shot type)
- Character LoRA (if you trained one, per module 3)

Cap total LoRA strength budget around 1.8. Above that, output degrades.

---

## 4.2 Recipe — Establishing shot (EST)

**Use case**: open a scene, set the location, no character closeup needed.

**Settings:**

| Param | Value |
|-------|-------|
| Type | Pure T2V (no I2V anchor) |
| Resolution | 832×480 (16:9) or 768×512 (3:2) |
| Frames | 97 (≈4s @ 25fps) |
| Camera LoRA | `static` OR `jib-up` (slow reveal) |
| Style LoRA | yes |
| Character LoRA | NO (no character in shot) |
| Sampler | distilled `ManualSigmas` 8 steps, `CFGGuider cfg=1.0` |

**Prompt template:**

```
[STYLE PREAMBLE]

Wide establishing shot of [LOCATION] at [TIME OF DAY]. [WEATHER]. [KEY VISUAL
ELEMENTS — building shapes, signage, vehicles, foliage]. No people in frame.

The camera [holds locked-off / slowly cranes upward / gently pushes in],
revealing the full scope of the location.

[LIGHTING — key light source, color temperature]. [ATMOSPHERIC EFFECTS — fog,
rain, dust motes, smoke]. [DEPTH CUES — foreground detail, mid-ground subject,
deep background].

[AMBIENT SOUND — wind, distant traffic, water, machinery]. [OPTIONAL MUSIC
HINT — a faint melody, a low drone].
```

**Concrete fill (Spider-Man 94 style):**

```
[Spider-Man 94 preamble]

Wide establishing shot of a rain-soaked Manhattan street at midnight. Yellow taxi
cabs splash through puddles. Steam rises from sewer grates. Neon signs in deep
red and electric blue reflect off wet asphalt. No people in frame.

The camera holds locked-off, slightly elevated, looking down the avenue toward
distant skyscrapers.

Single dramatic key light from a streetlamp casting deep shadows in the foreground,
with cool moonlight backlighting the city skyline. Light drifting fog gives depth.

Ambient city rain on asphalt, distant traffic hum, occasional taxi horn far away,
faint synth pad rising slowly.
```

Tuning:
- If too generic / lacks the "place": add 2–3 specific landmarks ("the Daily Bugle building visible in the distance", "a 1990s yellow checker cab in foreground")
- If too crowded: reduce element count
- If too clean: add "wet textures, debris, peeling posters, weathered surfaces"

---

## 4.3 Recipe — Wide shot with character (WS)

**Use case**: introduce a character in their environment, action scenes.

**Settings:**

| Param | Value |
|-------|-------|
| Type | T2V if generating fresh; I2V if anchoring to a previous shot |
| Resolution | 832×480 |
| Frames | 73–97 (3–4s) |
| Camera LoRA | `static`, `dolly-in`, or `jib-up` |
| Character LoRA | yes (one) |
| Anchor frame | recommended — character sheet "wide" still |

**Prompt template:**

```
[STYLE PREAMBLE]
[CHARACTER DESCRIPTION]

Wide shot of [character] standing on / walking through / arriving at [LOCATION
elements from prior establishing if same scene]. The camera frames the full
figure with the environment around.

[CHARACTER ACTION — movement, posture, gesture]. The camera [move].

[LIGHTING continues from prior establishing shot]. [character is lit by
specific source — backlit silhouette, side-lit hero shot, etc.].

[Environmental sound continues from EST]. [Footstep sound, fabric rustle,
specific character sound cue].
```

**Tip**: if this shot follows an establishing shot of the same location, **explicitly mention the location elements again** in this prompt. Don't assume the AI carries them over from the previous generation. It doesn't.

---

## 4.4 Recipe — Medium shot / Medium close-up (MS / MCU)

**Use case**: dialogue, emotional beats, character moments.

**Settings:**

| Param | Value |
|-------|-------|
| Type | I2V from character sheet still strongly recommended |
| Resolution | 768×512 or 704×512 |
| Frames | 49–97 (2–4s) |
| Camera LoRA | `static` (rare exception: slow `dolly-in` for dramatic reveal) |
| Character LoRA | yes |
| Anchor frame | character sheet medium / MCU still |

**Prompt template:**

```
[STYLE PREAMBLE]
[CHARACTER DESCRIPTION]

Medium close-up of [character] [posture]. [Character action — looks toward
camera-left, lifts hand to mouth, raises eyebrows]. Background is [softly
out of focus / clearly visible behind / dark and silhouetted].

The camera holds locked-off.

[Lighting — describe key light direction and quality]. [Subtle atmospheric
detail — single curl of breath in cold air, dust mote drift].

[Ambient bed]. [Character-specific sound — soft breath, fabric movement,
quiet object handling]. [Optional dialogue — see audio prompting in workflow
guide §9.11].
```

**Critical**: keep camera static for MCU dialogue. AI camera moves on tight framing usually look uncanny.

**Dialogue reality check**: LTX-2.3 generates speech-like audio that doesn't lip-sync. For real dialogue:
- Generate the shot silent (or with ambient only).
- Record / synth the dialogue separately (ElevenLabs).
- Lip-sync in post (module 5 covers Wav2Lip / SadTalker).

---

## 4.5 Recipe — Close-up / Extreme close-up (CU / ECU)

**Use case**: emotional peak, decision moment, reveal of detail.

**Settings:**

| Param | Value |
|-------|-------|
| Type | I2V mandatory (text-only CU is the worst case for character drift) |
| Resolution | 704×512 |
| Frames | 49 (2s) — usually short |
| Camera LoRA | `static` |
| Character LoRA | yes |
| Anchor frame | high-quality character close-up still — generate this carefully |

**Prompt template:**

```
[STYLE PREAMBLE]
[CHARACTER DESCRIPTION — full block, even though only face is visible]

Tight close-up on [character]'s face, filling the frame. [Specific micro-action
— eyes widen, jaw tightens, a single tear forms, lips part slightly].

Camera locked-off.

Dramatic lighting from [direction] casts half the face in shadow. [Background
detail — out-of-focus rain on a window behind, soft bokeh of city lights].

Quiet — [single isolated sound — heartbeat, breath, a clock ticking, distant
muted action]. The audio focuses attention on the face.
```

CU works best in held cels with tiny micro-motion. Don't ask for big actions in a CU.

---

## 4.6 Recipe — Action shot

**Use case**: fight, chase, leap, anything kinetic.

**Settings:**

| Param | Value |
|-------|-------|
| Type | T2V for fresh action; I2V if continuing from a previous beat |
| Resolution | 832×480 |
| Frames | 49–73 (2–3s — action shots cut fast) |
| Camera LoRA | matches the move (`dolly-in`, `dolly-out`, `dolly-left`, etc.) |
| IC-LoRA | **Motion-Track** if you can draw the desired motion path |
| Character LoRA | yes |

**Prompt template:**

```
[STYLE PREAMBLE]
[CHARACTER DESCRIPTION]

[Wide / medium] shot of [character] [action verb — leaps, swings, ducks,
strikes]. [Direction — toward camera, screen-right, into the air]. [Body
language — extended arms, taut tension, mid-stride].

The camera [matches the action — pulls back as character lunges forward,
tracks alongside, etc.].

[Motion lines, speed effects, dust trails, ground crack — describe in style
that matches your animation type]. [Lighting from prior shot].

[Sound — specific action sound first, then ambient — a thwip of webbing, a
heavy impact thud, fabric snap, breath]. Music swells.
```

**Motion-Track IC-LoRA is your secret weapon for action.** Workflow guide §10.3 covers wiring. You literally draw the motion path in `LTXVSparseTrackEditor` and the AI follows it. This is the single most reliable way to get a specific action choreography.

---

## 4.7 Recipe — Over-the-shoulder (OTS)

**Use case**: dialogue between two characters, reaction without revealing both faces.

**Settings:**

| Param | Value |
|-------|-------|
| Type | I2V with anchor frame mandatory |
| Resolution | 768×512 |
| Frames | 49–97 |
| Camera LoRA | `static` (almost always) |
| Character LoRA | both (if you have both trained); otherwise primary character only + describe second in text |

**Prompt template:**

```
[STYLE PREAMBLE]
[CHARACTER DESCRIPTIONS — both]

Over-the-shoulder shot from behind [character A — described as silhouette],
looking toward [character B in MCU / MS]. [Character A's shoulder and back of
head visible in foreground, soft focus]. [Character B in sharp focus].

[Character B's action / expression]. The camera holds.

[Lighting cohesive with the conversation setting]. [Atmospheric detail].

[Ambient bed]. [Sound of character B's action — coffee cup placed, paper
rustled].
```

**Tip**: silhouette characters are easier than fully-rendered ones. Use OTS framing as a way to feature two characters without paying the consistency cost on both.

---

## 4.8 Recipe — Insert / Cutaway

**Use case**: brief detail shot — a phone screen, a clock, a hand grabbing a weapon.

**Settings:**

| Param | Value |
|-------|-------|
| Type | T2V (no character usually — if hand: very tight framing) |
| Resolution | 704×512 or square 640×640 |
| Frames | 25–49 (1–2s) — inserts are SHORT |
| Camera LoRA | `static` or very slight `dolly-in` |
| Character LoRA | usually no |
| Style LoRA | yes |

**Prompt template:**

```
[STYLE PREAMBLE]

Tight insert shot of [object] in extreme close detail. [Texture, age, lighting
on the object]. [Optional small motion — clock hand ticks once, screen flickers
on, dust falls on surface].

Camera locked.

Dramatic single-source lighting raking across the object's surface. Shallow
focus.

Single isolated sound matching the object — clock tick, screen power-on chime,
metallic scrape.
```

**Inserts are easy.** They're often the cleanest-looking shots in an AI short because they have no characters and minimal motion. Use them generously — they buy you breathing room between harder character shots.

---

## 4.9 Recipe — Reaction shot

**Use case**: character A says/does something; cut to character B's reaction.

**Settings:**

| Param | Value |
|-------|-------|
| Type | I2V with anchor frame |
| Resolution | 704×512 (CU) or 768×512 (MCU) |
| Frames | 25–49 (1–2s — reactions are fast) |
| Camera LoRA | `static` |
| Character LoRA | yes |

**Prompt template:**

```
[STYLE PREAMBLE]
[CHARACTER DESCRIPTION]

[CU / MCU] of [character], starts in [neutral expression], then [reaction —
eyes widen, mouth opens slightly, slight head turn].

Camera locked.

Same lighting as previous shot.

Single sound cue — sharp intake of breath, soft "oh", or absolute silence.
```

**Tip**: a 1-second silent reaction shot is often more powerful than a 3-second one with audio. Cut it tight.

---

## 4.10 Recipe — POV shot

**Use case**: show what a character sees from their perspective.

**Settings:**

| Param | Value |
|-------|-------|
| Type | T2V |
| Resolution | 832×480 |
| Frames | 49–97 |
| Camera LoRA | matches POV character's head/body movement |
| Character LoRA | NO (we're seeing through their eyes) |

**Prompt template:**

```
[STYLE PREAMBLE]

First-person point-of-view. The viewer sees [WHAT'S VISIBLE — distant figure,
landscape, opponent, threat]. [Subtle frame movement implying the POV character's
breathing or motion].

The camera [matches the POV character's intended motion — slight bob if walking,
push-in if leaning forward, scan if looking around].

[Lighting matches scene]. [Atmospheric — rain on a visor, breath fog, blur of
visor edge].

[Ambient + breathing sound from POV character]. [Whatever sound is in the field
of view].
```

POV is powerful because **you sidestep the character consistency problem entirely** for that shot — they're not in frame.

---

## 4.11 Recipe — Title card / End slate

**Use case**: open or close the short with a logo / credits.

**Settings:**

| Param | Value |
|-------|-------|
| Type | T2V or pure stillimage held over time |
| Resolution | 832×480 |
| Frames | 49–97 (2–4s) |
| Camera LoRA | `static` |
| All other LoRAs | OFF or minimal |

LTX-2.3 cannot reliably render specific text on screen. Best approach:

1. Generate a title card "background" with no text via T2V (atmospheric, matching scene).
2. **Add the title text in post** using DaVinci Resolve / Premiere title tool.

Don't fight the model on text rendering — it loses.

---

## 4.12 Looping shot (for music videos, capstone option 3)

**Use case**: an 8-second clip that loops seamlessly.

**Settings:**

| Param | Value |
|-------|-------|
| Type | I2V with same image as both first AND last anchor (workflow guide §10.5) |
| Resolution | 768×512 |
| Frames | 161 (~6.4s @ 25fps — keep modest) |
| Camera LoRA | `static` (looping with camera moves is much harder) |
| Sampler | regular `SamplerCustomAdvanced` works; `LTXVLoopingSampler` if you go longer |

**Recipe:**

1. Generate a single high-quality character / scene still.
2. Use that still as `LTXVAddGuide(frame_idx=0, str=1.0)` AND `LTXVAddGuide(frame_idx=-1, str=1.0)`.
3. Prompt for a slow cyclic motion ("hair gently sways in wind", "neon sign flickers", "rain falls steadily") — easier to loop than directional motion.
4. In post, add a 3-frame crossfade between loop end and start to hide any seam.

---

## 4.13 Iteration strategy

For every shot, follow this loop:

1. **Lock the seed** in `RandomNoise`. Don't change it during iteration.
2. **Generate at low res first**: 512×384 or 640×384, 49 frames. Takes 30s on a 4090.
3. **Iterate on the prompt** until the framing, motion, and style are right. Each iteration: change ONE thing.
4. **Once locked**, regenerate at full target resolution. Only then run stage-2 upscaler.
5. **Save the prompt and seed** to a `shots_done.md` file. You'll need them when you re-render in 2 weeks.

Don't iterate at full resolution. Don't change five things between iterations. Don't unlock the seed until you're happy with the framing — seed change confounds prompt change.

---

## 4.14 When to give up on a shot

After ~15 iterations with no progress, **stop**. Either:

- **Re-frame the shot.** Maybe your CU should be an MCU; your wide should be a medium. Sometimes the AI just won't do a specific composition; reframe to a composition it will do.
- **Substitute a workaround.** Instead of "two characters fighting in this shot," cut to two solo shots with reaction inserts. Instead of a continuous 8-second action shot, three 2-second cuts.
- **Split the shot.** Generate the action in pieces and stitch.

The shot list you wrote in module 1 is a goal, not a contract. Adapt as you learn what the AI can and can't do for your specific style.

---

## 4.15 Project organization

For every shot, save:

```
project/
├── plan/
│   ├── shot_list.md
│   ├── style_preamble.md
│   ├── character_descriptions.md
│   └── character_sheet/
│       ├── hero_front.png
│       ├── hero_3q.png
│       └── ...
├── workflows/
│   ├── shot_001_T2V_EST.json   (saved ComfyUI workflow per shot)
│   ├── shot_002_I2V_WS.json
│   └── ...
├── renders/
│   ├── shot_001/
│   │   ├── v1.mp4
│   │   ├── v2.mp4
│   │   └── final.mp4   (the keeper)
│   ├── shot_002/
│   └── ...
├── audio/
│   ├── dialogue/
│   ├── music/
│   └── sfx/
└── final/
    ├── timeline.drp     (DaVinci Resolve project)
    └── export.mp4
```

You will lose your mind without this structure. Set it up day one.

---

## 4.16 Exercise

Generate every shot from your shot list using the recipe matching its type. Don't worry about audio yet — you'll add it in module 5. Just get clean visuals for every shot.

Save each `final.mp4` in its `renders/shot_XXX/` folder. When all shots are done, you're ready for post-production.
