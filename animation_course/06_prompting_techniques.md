# Module 6 — Prompting Techniques

Modules 2–4 gave you prompt scaffolds. This module is the underlying craft — why those scaffolds work, how to write your own, and the high-leverage techniques that separate "AI output" from "shot you wanted."

Prompting for LTX-2.3 is different from prompting for SDXL, Flux, or Midjourney. Most guides online cover CLIP-based prompting (keyword soup, `(word:1.4)` weighting, comma-separated tags). **That grammar doesn't apply here.** LTX-2.3 uses Gemma 3 12B Instruct — a full language model — as its text encoder. Gemma reads natural language.

This module covers what actually works.

> **Related ComfyUI Guide sections** — [ComfyUI Video Workflow Guide](../ComfyUI_Video_Workflow_Guide.md):
> - **§11** — Prompting Gemma for LTX-2.3 (vs. CLIP). The condensed technical reference that mirrors this module — CLIP-vs-Gemma grammar differences, the five-paragraph structure, emphasis-without-weights, negation/sequences/counting/comparatives support, and the iteration protocol.
> - **§9.11** — audio prompting. There is no separate audio conditioning field; sound cues live as natural language in the same positive prompt. This module's ¶5 sound paragraph is built on those rules.
> - **§9.4** — the Lightricks short concept-level negative convention and verbatim template example prompts. Use these as the canonical calibration set for your own prompt drafts.

---

## 6.1 How the model reads your prompt

Gemma 3 12B is a decoder LLM, not CLIP. Practical consequences:

| SD / CLIP convention | Gemma / LTX-2.3 reality |
|----------------------|--------------------------|
| Comma-separated tokens (`red dress, long hair, forest, night`) | Fine but **inferior** to full sentences. Gemma understands prose. |
| `(word:1.4)` emphasis syntax | **Not parsed as weighting.** Gemma reads the parentheses as punctuation. Repetition and phrasing are how you emphasize. |
| 77-token context limit | Not a concern. Gemma's context is large enough for paragraph-length prompts. |
| Short prompts ("anime girl, blue eyes, sword") | Works poorly. Underspecified for Gemma, which is designed for long context. |
| Negative = long laundry list | Lightricks convention is short concept-level negative (see workflow guide §9.4). |
| Word order barely matters | Word order **matters**. First sentence anchors style; later sentences layer specifics. |
| Token dropout / emphasis tricks | Plain language works best — describe, don't list. |

**Bottom line**: write like a cinematographer describing a shot to a crew, not like an SD power-user tagging an image.

---

## 6.2 The five-paragraph shot prompt

Every shot prompt should fit this structure. Paragraphs exist for a reason — Gemma reads the blank lines as semantic breaks, which helps it organize the information.

```
¶1  STYLE + MEDIUM          (copy-paste across entire project)
¶2  SUBJECT + FRAMING       (who/what, in what shot type)
¶3  ACTION + CAMERA         (what happens + how the camera behaves)
¶4  LIGHTING + ATMOSPHERE   (look of the scene)
¶5  SOUND                   (audio layers)
```

This is the same scaffold as module 4, made explicit as a grammar. When you write a new shot:

1. Copy paragraph 1 from your style bible — never rewrite.
2. Copy-paste any character descriptions into paragraph 2.
3. Write paragraphs 3–5 fresh for this shot.

**Do not collapse paragraphs.** A single wall of text performs worse than the same sentences separated. Tested repeatedly.

---

## 6.3 Motion verbs — the most undervalued prompt element

Weak verbs produce static, ghostly output. Strong verbs produce motion. This is the #1 fix for "why does my video look frozen."

| Weak | Strong | What changes |
|------|--------|--------------|
| "The character is in the forest" | "The character walks through the forest" | Implies camera & body motion |
| "She looks sad" | "She lowers her gaze and exhales slowly" | Specific micro-motion |
| "Rain falls" | "Heavy rain streaks diagonally across the frame" | Direction + velocity |
| "The car moves" | "The car accelerates hard, tires spinning, toward camera-right" | Direction + physics |
| "Light in the room" | "Morning sunlight spills across the wooden floor as dust motes drift" | Motion of the light quality itself |

**Rule**: every sentence in paragraph 3 (action) should contain at least one concrete motion verb. If a sentence has no motion, the AI generates a still.

**Pacing verbs**: "slowly", "suddenly", "gradually", "smoothly", "abruptly" — these affect the tempo of the generated motion. Use deliberately.

---

## 6.4 Camera language — think cinematography

You have a full vocabulary of camera terms. Use it. The model was trained on cinematic data — it responds to the same terms a DP would use on set.

| Term | What it says |
|------|--------------|
| "locked-off camera" | Camera doesn't move. Pairs with `static` LoRA. |
| "handheld" | Organic wobble and float. Good for action/realism. |
| "dolly-in / dolly-out" | Camera physically moves toward/away. Pairs with dolly LoRAs. |
| "pushing in" / "pulling back" | Same as dolly-in/out but warmer prose. |
| "tracking shot" | Camera moves alongside subject. |
| "crane shot" / "jib rises" | Camera elevates. Pairs with `jib-up`. |
| "tilt down / pan left" | Rotation moves (no released LoRA; prompt-only). |
| "rack focus" | Focus shifts from foreground to background. |
| "shallow depth of field" | Background is blurred. |
| "wide angle lens" | Exaggerated perspective, distortion at edges. |
| "anamorphic" | Cinematic 2.39:1 feel, lens flares. |
| "over-the-shoulder" | Specific framing — model recognizes. |
| "Dutch angle" | Tilted horizon. Use for tension. |
| "low angle" / "high angle" | Camera position relative to subject. |
| "worm's eye view" / "bird's eye view" | Extreme angles. |

**Pair prompt camera language with your camera LoRA** — they should agree. `dolly-in` LoRA + "the camera slowly pushes in toward her face" is better than either alone.

---

## 6.5 Composition — put the subject somewhere specific

Don't write "a character in a field." Write where in frame they are.

| Phrase | Effect |
|--------|--------|
| "centered in frame" | Subject locked to middle |
| "in the left third" / "in the right third" | Rule-of-thirds composition |
| "framed against a clear horizon" | Negative space above |
| "surrounded by foreground leaves out of focus" | Natural framing |
| "filling the frame" | Tight CU feel |
| "tiny figure in the lower third, vast landscape above" | Scale emphasis |
| "silhouetted against the window" | Backlight + frame-within-frame |
| "foreground detail, mid-ground subject, deep background" | Explicit depth staging |

The AI has strong compositional priors (rule of thirds, centered portrait). Deviating from priors requires specific language.

---

## 6.6 Lighting — the biggest visual lever after style

Vague: "dramatic lighting."
Specific: "single warm key light from camera-right at shoulder height; cool blue fill from the window behind; rim light catches the edge of her hair."

Lighting vocabulary:

| Term | Use |
|------|-----|
| "key light" | The dominant light on the subject |
| "fill light" | Softer secondary light reducing shadow contrast |
| "rim light" / "backlight" | Light from behind outlining the subject |
| "practical" | An in-scene light source (lamp, window, fire) |
| "motivated" | Light whose direction matches a visible source |
| "hard light" / "soft light" | Shadow edge quality |
| "chiaroscuro" | Strong light/dark contrast |
| "flat lighting" | Even, no drama |
| "three-point lighting" | Classic portrait setup |
| "low-key" / "high-key" | Overall exposure — dark and moody vs bright and clean |
| "golden hour" | Warm, low-angle sun |
| "blue hour" | Cool, post-sunset twilight |
| "neon practical" | Colored light from in-scene neon sign |

**Animation-specific lighting phrases**: "cel-shaded lighting with sharp shadow edges," "soft anime glow on the face," "saturated comic-book highlights," "painted background with watercolor wash light."

---

## 6.7 Atmosphere — the cheap trick that elevates every shot

Weather, particles, fog, depth haze — these take prompts from "AI-looking" to "cinematic." Drop one into every shot that can support it.

| Atmospheric element | Effect |
|---------------------|--------|
| "light fog drifts across the mid-ground" | Adds depth, softens background |
| "dust motes float in the shafts of light" | Implies stillness and warmth |
| "light rain streaks the frame" | Motion + mood + softens backgrounds |
| "snow drifts slowly down" | Calm, wintry |
| "heat shimmer distorts the distance" | Hot environment |
| "gentle wind ripples the grass" | Living environment |
| "smoke curls upward in the foreground" | Dramatic, ominous |
| "steam rises from the pavement" | Wet-urban mood |
| "bokeh of city lights behind" | Dreamy, photographic |

**Rule**: every environment shot should have at least one atmospheric element. Without one, the shot feels clean but sterile.

---

## 6.8 Color description — named shades beat hex codes

The model doesn't know `#B22222`. It knows "cherry red," "brick red," "crimson," "oxblood" — each with slightly different associations. Use named colors.

| Generic | Specific |
|---------|----------|
| "blue" | "navy blue", "cobalt", "electric blue", "periwinkle", "teal", "sky blue", "denim" |
| "red" | "cherry red", "crimson", "scarlet", "brick", "oxblood", "coral", "rose" |
| "green" | "emerald", "forest green", "sage", "chartreuse", "jade", "olive" |
| "yellow" | "golden", "mustard", "cream", "lemon", "amber" |
| "orange" | "terra cotta", "burnt orange", "tangerine", "peach" |

**Palette phrasing**: "muted earth-tone palette dominated by olive and amber with occasional brick red accents" is infinitely better than "green and brown and red."

---

## 6.9 Character pose, expression, gesture

The hardest prompts. Be surgical.

**Pose vocabulary:**
- "standing with weight on the right leg, left hand on hip"
- "crouched low, right knee on the ground, left arm extended"
- "mid-stride, leading with the right foot"
- "leaning forward against a counter, elbows resting"

**Expression vocabulary** (avoid basic emotion labels when you can):
- Not "happy" → "the corners of her mouth lift slightly; eyes crinkle at the edges"
- Not "angry" → "his jaw is set; a small muscle flickers at his temple"
- Not "sad" → "she looks slightly off-camera, her eyes not focused on anything"
- Not "surprised" → "his eyebrows lift sharply; his mouth forms a small 'o'"

The descriptive version activates motion priors the emotion label doesn't.

**Gesture**:
- "her hand drifts up to touch her throat"
- "he rolls his shoulders back"
- "she exhales slowly through her nose"

Small gestures add life. One gesture per shot is usually right; two fight.

---

## 6.10 Sound prompting — layered writing

From module 2, LTX-2.3 uses the same text for audio. To get a full soundscape rather than generic ambience, layer explicitly:

**Bad (single layer)**: "ambient city sound."

**Good (three layers)**:
```
Rain hisses on the rooftop and distant traffic hums far below (ambient bed).
A single taxi horn sounds once from three blocks away (foreground element).
A slow synth pad rises beneath the visuals (musical bed).
```

**Sound layer template**:

| Layer | What to describe |
|-------|------------------|
| Ambient bed | The quiet background of the space (wind, traffic hum, room tone) |
| Foreground sound | A specific cue tied to action (footstep, impact, object handled) |
| Atmospheric | Weather, water, environmental (rain, thunder, waves) |
| Musical | Genre + mood + instrument ("faint synth pad", "low orchestral drone") |
| Vocalization | Breath, laugh, sigh (not dialogue — LTX doesn't lip-sync) |

Not every shot needs all five. Pick the 2–3 that serve the moment.

---

## 6.11 Negative prompts — Lightricks convention

Workflow guide §9.4 established this: short, concept-level negatives. **Do not** paste the SD laundry list.

Reusable starter negative:
```
pc game, console game, video game, cartoon, childish, ugly
```

Add to this list **only** when you see specific problems:

| Problem you see | Add to negative |
|-----------------|-----------------|
| Output is too anime when you want American animation | `anime, manga, japanese animation` |
| Output is too 3D-rendered | `3D rendered, CGI, pixar, video game` |
| Output looks cheap / flat | `childish, cheap, amateurish` |
| Output feels too modern for retro style | `modern, HDR, digital sharpness` |
| Output feels too realistic for stylized style | `photorealistic, photography, live action` |

**Don't** add:
- "blurry, low quality, jpeg artifacts" — Lightricks training didn't weight these; they're SD conventions.
- "extra fingers, bad hands, deformed" — these don't work on video models the way they work on image models.
- More than ~8 concepts total — diminishing returns after that.

---

## 6.12 Emphasis without weights

Since `(word:1.4)` doesn't work in Gemma, how do you emphasize?

**1. Repetition across paragraphs.** Mention the key visual element in two different paragraphs. If it's critical, three.

> "She wears a cherry-red costume." ... (paragraph later) "The cherry-red fabric catches the rim light."

**2. Specificity.** A specific noun beats an emphasized generic noun.

> Weak: `(red costume:1.4)`
> Strong: "a fitted cherry-red costume with black spider-web pattern stitched across the chest"

**3. Front-loading.** Things mentioned early in the prompt carry more weight. Critical elements go in paragraph 2 (subject), not paragraph 5.

**4. Conjugation density.** Multiple adjectives on one noun add weight: "a tall, gaunt, deeply shadowed figure" emphasizes the figure more than "a tall figure."

---

## 6.13 The iteration protocol — change one thing

When a prompt isn't producing what you want, the biggest mistake is changing five things at once. You lose signal.

**Protocol**:

1. **Lock the seed.** No exceptions while iterating.
2. **Identify ONE thing that's wrong.** Not a vibe — a specific visual element.
3. **Edit ONE paragraph.** Usually the paragraph that "owns" that thing (motion → ¶3, lighting → ¶4, etc.).
4. **Rerun at low res.** Don't regenerate at full res while iterating.
5. **Compare against previous version.** Did the one thing change? Did other things stay the same?
6. **Keep or revert.** If the edit helped, move on. If it hurt, revert and try a different edit.

Typical cycle time on a 4090 at 512×384 with 49 frames: ~25 seconds. You can do 30 iterations in an evening.

---

## 6.14 Reusable prompt fragments — build a library

Create a `prompt_fragments.md` in your project. As you discover phrases that work, save them by category:

```markdown
## Lighting
- "warm key light from camera-left at shoulder height; cool blue fill from behind"
- "single practical light from a flickering neon sign casts magenta across the scene"
- "harsh overhead fluorescents flatten shadows; slight green tint"

## Atmospheres
- "light fog drifts across the mid-ground; dust motes in the key light"
- "heavy rain streaks diagonally; puddles reflect neon"

## Camera moves
- "camera holds locked-off; subject moves within frame"
- "camera slowly dollies in toward the face over the duration of the shot"

## Sound beds
- "rain on pavement, distant taxi horns, faint synth pad underneath"
- "room tone of an empty apartment, a clock ticks once every second"
```

After your first project, you'll have 40–80 reusable fragments. For the next project, you're assembling more than writing. This is how pipelines become muscle memory.

---

## 6.15 Gemma-specific quirks worth knowing

**Gemma is smarter than CLIP but also more literal.**

- "A woman" → generates a woman. "A young woman in her late twenties with observable life experience" → generates a specific woman. Specificity is rewarded.
- Gemma understands **negation in the positive prompt** ("no camera movement", "no dialogue", "without any text on screen"). CLIP often inverts these; Gemma respects them.
- Gemma understands **sequences** ("first she stands still, then she turns toward the camera"). Use this for shots with two beats.
- Gemma understands **counting** ("three figures in the background"). Use numerals, not words ("3" parses more reliably than "three" in some prompts — test for your case).
- Gemma understands **comparatives** ("taller than the figure behind her", "quieter than the previous shot's audio"). Useful for relative sizing.
- Gemma understands **causality** ("because of the heavy rain, the streets are empty"). Adds motivation the model can express.

**Known weaknesses:**
- Reading-order text: rendering specific text on screen fails. Add text in post.
- Precise counting above ~5. "Seven candles" often renders 4 or 8.
- Hands performing specific finger actions in CU.
- Brand-name items (specific logos, specific car models) — drift badly.

---

## 6.16 Common prompting pitfalls

| Pitfall | Fix |
|---------|-----|
| Prompt is too short (<3 sentences) | Expand to the five-paragraph structure |
| Prompt is a tag soup (commas only) | Rewrite as prose |
| Using `(word:1.4)` emphasis | Drop it; use repetition and specificity |
| Style preamble differs per shot | Copy-paste the exact same preamble every shot |
| Describing static-ness | Every sentence in ¶3 needs a motion verb |
| Vague lighting ("dramatic lighting") | Specify direction, color, quality |
| No atmospheric element | Add fog/rain/dust to every environment shot |
| Mixing SD and LTX conventions | Pick one style — for LTX it's prose |
| Over-descriptive character (every costume detail in CU) | In CU, drop items not visible. Conserve tokens for what's in frame. |
| Audio layer is vague ("ambient sound") | Layer three sound elements explicitly |
| Negative prompt is 40 SD tags | Keep to ≤8 concept tokens |
| Changing prompt + seed simultaneously during iteration | Lock seed first; isolate changes |

---

## 6.17 Before / after evolution — one shot, three drafts

Take this as a working example of how a prompt matures.

### Draft 1 — beginner (tag soup)

```
anime, cyberpunk, night, rain, woman, red jacket, rooftop, looking down, neon,
cinematic, 4k, detailed
```

Result: generic AI anime, static, floaty. Character looks nothing in particular. Rain is decorative.

### Draft 2 — expanded prose

```
A modern Japanese anime, digital cel-shading, cinematic composition.

A young woman in her mid-twenties with short dark hair and a red leather jacket
stands on the edge of a rooftop at night, looking down at the street below.

She exhales slowly; her breath fogs in the cold. The camera holds locked-off.

Heavy rain falls around her. Neon signs below cast magenta and cyan reflections
on the wet rooftop. Backlight from the city silhouettes her figure.

Rain hisses on the rooftop surface. Distant city traffic hums far below. A
slow synth pad underneath.
```

Result: much better. Style is clear, motion is present, atmosphere works.

### Draft 3 — polished

```
A modern Japanese anime in the style of a contemporary urban action series,
clean digital cel-shading with crisp line work, high-contrast dramatic
lighting, saturated character colors against painterly neon-lit backgrounds,
24fps anime feel.

Medium shot of a young woman in her mid-twenties, short dark hair cut just
above the shoulders, wearing a fitted red leather jacket over a gray t-shirt,
standing on the edge of a rooftop at night. She is framed in the left third
of the composition, with the city stretching to the right.

She exhales slowly through her nose; her breath fogs visibly in the cold air.
Her gaze drops toward the street below. The camera holds locked-off.

Heavy rain streaks diagonally across the frame. Saturated magenta and cyan
neon signs glow on wet asphalt in the distance, painting colored reflections
across the rooftop surface. A warm orange rim light from a nearby window
catches the edge of her hair; cool backlight silhouettes her shoulders
against the deep city haze behind.

Rain hisses on the rooftop and gutter. Distant city traffic hums steadily. A
single taxi horn sounds once, far below. A slow analog synth pad drifts
underneath, rising gradually.
```

Result: specific, cinematic, consistent. This prompt will reproduce reliably across seeds.

---

## 6.18 Prompt linting checklist

Before you queue a shot, run this mental check:

- [ ] Five paragraphs, blank lines between them
- [ ] Paragraph 1 is copy-pasted from your style bible (unchanged)
- [ ] Paragraph 2 names the shot type and framing explicitly
- [ ] Every sentence in ¶3 has a motion verb
- [ ] Camera behavior is stated in ¶3 (matches your camera LoRA if any)
- [ ] Lighting in ¶4 has a direction and a color/quality
- [ ] ¶4 contains at least one atmospheric element
- [ ] ¶5 layers at least two sound elements
- [ ] No `(word:1.4)` weights
- [ ] Negative prompt is ≤8 concept tokens
- [ ] Character description block (if applicable) is identical to the one in other shots featuring this character

If all boxes check, the prompt is ready.

---

## 6.19 Exercise

1. Pick one shot from your shot list that hasn't been prompted yet.
2. Write **Draft 1** as tag soup (≤15 words).
3. Generate at low res with a locked seed. Save the video.
4. Rewrite as **Draft 2** following the five-paragraph structure (~8–12 sentences).
5. Generate at same locked seed. Save and compare.
6. Rewrite as **Draft 3** adding cinematography, lighting detail, atmosphere, layered sound.
7. Generate at same locked seed. Compare all three.
8. Note which prompt elements made the biggest difference. Add those patterns to your `prompt_fragments.md`.

You will use this exercise's output as a calibration reference for every future project.
