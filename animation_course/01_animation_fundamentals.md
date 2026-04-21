# Module 1 — Animation Fundamentals for AI Video

Before you generate a single frame, you need to think like a director and an animator. AI is a generation engine; it doesn't know what makes a watchable scene. That's your job.

This module is short on tools and long on concepts. Skip it at your peril — every wasted GPU-hour downstream is usually a planning failure here.

> **Related ComfyUI Guide sections** — [ComfyUI Video Workflow Guide](../ComfyUI_Video_Workflow_Guide.md):
> - **§9.6** — resolution and frame constraints (`8n+1` frames, 32-stride W/H, aspect presets). Your shot list's durations have to snap to these.
> - **§10.2** — camera-control LoRAs (dolly-in/out, jib-up/down, static). Informs which camera moves are cheap vs. which require stitching.

---

## 1.1 The atomic unit: the shot

A **shot** is one continuous take from one camera. Cut, and you have a new shot.

Why this matters for AI video: **you are not generating a movie, you are generating shots**. Each shot is a separate ComfyUI run. A 30-second scene is typically 5–10 shots. A 22-minute episode is 200–400 shots. You don't tell the AI "make me a 22-minute episode" — you tell it "make me shot #47, a 3-second medium close-up of the hero looking up."

This reframe is the single most important mental shift in this course. Stop thinking in scenes. Start thinking in shots.

---

## 1.2 Shot types you'll use constantly

| Type | Abbrev. | What it shows | When to use |
|------|---------|---------------|-------------|
| Establishing | EST | Wide view of location | Open a scene; orient the viewer |
| Wide shot | WS | Full body in environment | Action, movement, geography |
| Medium shot | MS | Waist up | Dialogue, gesture |
| Medium close-up | MCU | Chest up | Close dialogue, intimate |
| Close-up | CU | Head/face | Emotion, decision, key moment |
| Extreme close-up | ECU | Eyes, mouth, detail | Reveal, tension peak |
| Over-the-shoulder | OTS | Behind char A looking at B | Conversations |
| Two-shot | 2S | Both characters in frame | Establish relationship |
| Point-of-view | POV | What a character sees | Subjective experience |
| Insert | INS | Detail shot (object, hand) | Plot point, atmosphere |
| Cutaway | CA | Brief break to something else | Reaction, parallel action |

A typical scene rhythm: **Establishing → Wide → Medium → Close → Insert → Reaction → Wide → Cut**. You don't need all of these; pick what serves the beat.

**Rule of thumb for AI**: close-ups and medium shots are easier than wides. Wides struggle with consistent backgrounds and tend toward generic. Plan for more close work.

---

## 1.3 Shot length

Average shot length in modern animation:

| Style | Avg length | Notes |
|-------|------------|-------|
| Modern action anime | 1.5–3s | Fast cutting; lots of impact frames |
| 90s Saturday-morning (Spider-Man '94) | 2–4s | Limited animation; held cels |
| Cinematic anime (Ghibli) | 4–8s | Slow, contemplative |
| Music video | 0.5–2s | Cuts on the beat |

LTX-2.3 sweet spot is **2–6 seconds per generation** (49–161 frames at 25 fps). Plan your shot lengths inside that window. For longer shots, use the looping sampler (workflow guide §10.5).

---

## 1.4 Camera moves

| Move | What it does | LTX LoRA |
|------|--------------|----------|
| Static / locked-off | Camera doesn't move; subject moves within frame | `static` LoRA |
| Pan (left/right) | Camera rotates horizontally on a fixed axis | not released — describe in prompt |
| Tilt (up/down) | Camera rotates vertically | not released |
| Dolly (in/out) | Camera physically moves toward/away from subject | `dolly-in` / `dolly-out` |
| Truck (left/right) | Camera physically moves laterally | `dolly-left` / `dolly-right` |
| Pedestal / Jib (up/down) | Camera rises or descends | `jib-up` / `jib-down` |
| Zoom | Lens focal length changes (different from dolly) | describe in prompt; no LoRA |
| Roll | Camera rotates around its lens axis | rare; describe in prompt |

**Animation convention**: less camera movement than live-action. Lots of static framing with motion happening within the frame. When the camera moves, it usually moves slowly and deliberately. Don't over-direct.

---

## 1.5 The twelve principles, abridged for AI

The classic Disney animation principles, filtered for what an AI generator can use:

| Principle | What it means | Apply to AI by... |
|-----------|---------------|--------------------|
| Squash and stretch | Volume distorts under force | Choose styles whose LoRAs have it baked in (anime impact frames) |
| Anticipation | Action is preceded by a wind-up | Describe the wind-up in the prompt; use `LTXVAddGuide` for a key wind-up frame |
| Staging | Composition serves the idea | Pick shot type and camera angle deliberately, not by default |
| Timing | Rhythm of motion sells the action | Choose frame count carefully; longer = more weight, shorter = quicker |
| Slow in / slow out | Motion accelerates and decelerates | Built into the model; supplement by writing speed-aware prompts ("slowly", "abruptly") |
| Arcs | Things move in curves, not lines | Built in; describe arcs in prompts ("the bird arcs across the frame") |
| Secondary action | Hair, cloth, smaller parts move alongside primary action | Describe in prompt explicitly; AI often forgets |
| Exaggeration | Push the pose past realism | Critical for cartoon styles; use prompt words like "exaggerated", "dynamic" |
| Solid drawing | Forms have weight and volume | Style LoRAs; otherwise expect floaty AI weirdness |
| Appeal | Characters are interesting to look at | Character design happens in your prompts and reference images |

You can't make AI follow these principles directly. But knowing them tells you what to **describe** in prompts and what to **fix in post**.

---

## 1.6 The production pipeline (idea → screen)

```
Idea
  ▼
Logline (1 sentence: who wants what, what's stopping them)
  ▼
Treatment (1 paragraph per scene)
  ▼
Script (dialogue + action lines)
  ▼
Storyboard (1 panel per shot — rough sketch)
  ▼
Shot list (spreadsheet with one row per shot)
  ▼
Style bible (palette, character refs, prop refs)
  ▼
GENERATE (this is where ComfyUI enters — modules 2–4)
  ▼
Edit / assemble (module 5)
  ▼
Sound design (module 5)
  ▼
Color grade + final mix
  ▼
Export
```

Most beginner mistakes happen because steps 1–6 were skipped. You opened ComfyUI, typed a vague prompt, and hoped. That's how you burn 8 hours and end up with nothing watchable. **Plan first, generate second.**

---

## 1.7 The shot list — your generation Bible

A shot list is a spreadsheet (or markdown table) where each row is one shot you'll generate. Minimum columns:

| Col | Example | Why |
|-----|---------|-----|
| `#` | 003 | Sortable identifier |
| `scene` | INT-CAFE-DAY | Group shots by scene |
| `type` | MCU | Drives prompt + workflow choice |
| `subject` | Hero, Sara | Who's in frame |
| `action` | Sara sets down coffee, looks up | What happens |
| `camera` | static | Drives camera LoRA |
| `length_s` | 3 | Drives frame count |
| `length_fr` | 73 | `length_s × fps` rounded to `8n+1` |
| `dialogue` | "You came back." | Drives audio prompt |
| `notes` | low light, neon outside | Drives style + atmosphere |
| `status` | done / WIP / blocked | Track progress |

Build this BEFORE you open ComfyUI. Treat it as the contract. When a shot is done, you mark it done. When you're tempted to "just regenerate one more time," check the shot list — does this match the intent? If yes, accept it and move on.

**Sample shot list for a 30-second cold open:**

| # | scene | type | subject | action | camera | length_s | length_fr | dialogue | notes |
|---|-------|------|---------|--------|--------|----------|-----------|----------|-------|
| 001 | EXT-CITY-NIGHT | EST | — | rain over neon skyline, no people | static | 4 | 97 | — | establishing mood |
| 002 | EXT-CITY-NIGHT | WS | Hero | Hero stands on rooftop, back to camera, cape moving in wind | jib-up | 3 | 73 | — | reveal hero |
| 003 | EXT-CITY-NIGHT | MCU | Hero | Hero turns head, looks down toward street | static | 2 | 49 | — | beat |
| 004 | EXT-CITY-NIGHT | POV | (Hero's view) | look down at street, distant figure running | static | 3 | 73 | distant siren wails | what hero sees |
| 005 | EXT-CITY-NIGHT | CU | Hero | Hero's eyes narrow | static | 2 | 49 | — | decision |
| 006 | EXT-CITY-NIGHT | WS | Hero | Hero leaps off the building toward the camera | dolly-out | 3 | 73 | wind whoosh | action |
| 007 | EXT-CITY-NIGHT | EST | — | logo title card over rooftop silhouette | static | 4 | 97 | music sting | title |

Total: 21 seconds of footage across 7 shots. With +5s for music tail and pacing, that's a ~26s short. You now know exactly what to generate.

---

## 1.8 What current AI video can / can't do (be honest)

**It can do well:**
- 2–10 second clips of a single character or scene with simple action.
- Atmospheric shots (weather, lighting, environments).
- Hand-drawn-adjacent looks with the right style LoRAs.
- Clean camera moves (dolly, jib) when the camera LoRA matches the prompt.
- Ambient and cinematic audio that matches the visual mood.

**It struggles with:**
- Long sustained shots (>10s) without seam artifacts.
- Specific recurring characters across many cuts (without a trained LoRA).
- Multi-character action with clear choreography.
- Clean text on signs, books, screens.
- Hands and fingers in close-up doing complex actions.
- Lip-sync to specific dialogue (need post-process).
- Identical background elements across shots (a specific painting, a specific car).

**It mostly can't do (yet):**
- 30-second continuous unedited shot with a complex action sequence.
- A full named character with consistent costume/face across 100 shots without training a LoRA.
- Live music videos where every cut hits a specific drum hit.

When you plan a sequence, **avoid the can't and lean on the can**. Replace a 12-second continuous shot with three 4-second cuts. Replace a complex two-character action with a single-character solo + insert + reaction. The constraints aren't obstacles; they're the actual creative grammar of this medium right now.

---

## 1.9 Style anchors — pick one before you start

Before module 2, decide what your short looks like. Three reference styles we'll use throughout:

**A. Spider-Man: The Animated Series (1994)**
- Bold black ink outlines on every form
- Limited color palette; warm browns, deep purples, saturated reds/blues
- Limited animation — many held cels, smear frames for fast moves
- Comic-book composition; characters often centered or thirds
- Mid-90s TV anime/cartoon aesthetic; not photoreal at all

**B. 80s OVA anime (Akira, Bubblegum Crisis, Ghost in the Shell)**
- Detailed line work, hand-drawn texture
- Heavy use of pre-CG practical effects (lens flares, neon)
- Saturated cool colors; lots of cyan/magenta
- Cinematic widescreen framing
- Atmospheric backgrounds with extreme detail

**C. Modern cel-shaded anime (Demon Slayer, Jujutsu Kaisen)**
- Clean line work; sometimes mixed with CG
- High-contrast color, dramatic lighting
- Impact frames, motion lines, exaggerated speed
- Faster cutting than 80s anime
- Crisper, more "digital" finish

Each of these has a distinct prompt scaffold and LoRA strategy. Module 2 covers them.

Other styles worth considering (covered briefly in module 2): Tarzan/Disney 1999 painted look; Into the Spider-Verse (chromatic aberration + halftone); Tartakovsky-era Cartoon Network (Samurai Jack — clean geometric); Studio Ghibli (soft watercolor backgrounds + clean cels).

**Pick one** and stick with it for your first project. Switching styles mid-project is the fastest way to a Frankenstein result.

---

## 1.10 Exercise

Before moving on:

1. Pick a style anchor (A, B, C, or another).
2. Pick a 30-second story idea. One sentence is enough. Example: *"A masked vigilante on a rainy rooftop spots a fleeing thief and decides to follow."*
3. Build the shot list (use the template in §1.7). Aim for 6–10 shots.
4. Write down the constraints: max VRAM, target FPS, target final length.

When the shot list is on paper, you're ready for module 2.
