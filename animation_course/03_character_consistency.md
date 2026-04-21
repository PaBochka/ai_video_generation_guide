# Module 3 — Character Consistency Across Shots

This is the hardest unsolved problem in AI animation right now. Get it right and your short feels professional. Get it wrong and viewers immediately notice the hero's face changes between cuts.

This module ranks the available techniques from most to least reliable, and gives you a practical recipe for each.

> **Related ComfyUI Guide sections** — [ComfyUI Video Workflow Guide](../ComfyUI_Video_Workflow_Guide.md):
> - **§10.4** — multi-keyframe / anchor-frame I2V. `LTXVAddGuide` (soft anchor with `frame_idx`), `LTXVImgToVideoInplace` (rigid boundary anchor), and the FLF2V / FMLF2V build pattern.
> - **§10.8** — worked example: FLF2V with an intermediate anchor at frame 48 and tuning notes for "middle pops/snaps" vs. "middle drifts". This is the pattern behind Recipe C (anchor-frame consistency within a shot).
> - **§9.5** — `LTXVPreprocess` is required on every anchor image. Skip it and the shot goes static/ghostly.

---

## 3.1 Why this is hard

When you generate shot #1 of "the hero in a red and blue costume on a rooftop," the AI invents a specific face, costume detail, body proportion, and color shade. When you generate shot #2 with the same prompt, the AI invents a **different** specific face, costume detail, etc. The text prompt is too low-bandwidth to lock these specifics.

Solutions all boil down to: **give the AI more information than text**. Reference images, trained adapters, or anchor frames.

---

## 3.2 The technique stack — ranked by reliability

| Rank | Technique | Reliability | Effort | When to use |
|------|-----------|-------------|--------|-------------|
| 1 | Trained character LoRA | Very high | High (4–8 hrs) | Recurring character, multi-project |
| 2 | IP-Adapter with reference images | Medium-high | Low | Single project, one or two characters |
| 3 | Anchor frames via FLF2V | Medium | Low | Within a single shot (start/end frames consistent) |
| 4 | Same seed + same starting frame | Low-medium | Trivial | Same character within consecutive shots only |
| 5 | Highly specific text description | Low | Trivial | When no other option; better than nothing |

You'll often combine multiple techniques. The capstone recipes in §3.8 do.

---

## 3.3 Technique 5: detailed text description (the floor)

Even with no LoRA, no IP-Adapter, no reference images, you can fight drift by writing the same character description verbatim in every shot prompt.

**Bad (drifts every generation):**
```
The hero stands on the rooftop.
```

**Good (locks the most you can with text alone):**
```
A male superhero, mid-20s, lean athletic build, wearing a tight red and blue
costume with a black spider emblem across the chest, full red mask covering the
entire face with large white eye lenses, no visible skin or hair. He stands on
the rooftop.
```

**Critical**: copy-paste this character description block into every shot prompt that includes the character. Don't paraphrase. The token sequence consistency matters more than you'd expect.

Build a `character_descriptions.md` file alongside your shot list:

```markdown
## Hero
A male superhero, mid-20s, lean athletic build, wearing a tight red and blue
costume with a black spider emblem across the chest, full red mask covering the
entire face with large white eye lenses, no visible skin or hair.

## Sara
A young woman, late 20s, shoulder-length dark brown hair, green eyes, wearing
a worn black leather jacket over a gray t-shirt, dark jeans, scuffed combat
boots. Slim build, alert posture.
```

Reference this file when writing shot prompts. Don't deviate.

This technique alone gets you maybe 30% consistency. Add the others to climb.

---

## 3.4 Technique 4: same seed + same first frame

For a series of consecutive shots of the same character in the same scene, lock the seed in `RandomNoise` and use the **last frame of the previous shot as the first frame of the next** (via `LTXVAddGuide(frame_idx=0)` — workflow guide §10.4).

This is essentially the manual stitch pattern from workflow guide §10.5. Reliability is medium for the next 1–2 shots; degrades after that.

**When it shines**: a single beat broken into 2–3 cuts (close-up reaction → cut to wide → cut back). The AI carries enough visual continuity from the anchor frame to keep the character close to consistent.

**When it fails**: cuts to a different angle, time, or location. Anchor frame doesn't help if the new shot needs a different framing.

---

## 3.5 Technique 3: anchor frames within a shot (FLF2V)

Already covered in workflow guide §10.4 and §10.8. The version relevant to character consistency:

**Pre-render the character.** Generate a single high-quality still image of your character (using SDXL, Flux, or even LTX-2.3 with `frames=1`) in the exact pose / costume / lighting you want. Use this still as the **first-frame anchor** in I2V.

```
LoadImage (your locked character still) ──▶ LTXVPreprocess ──▶
  LTXVAddGuide(frame_idx=0, strength=1.0) ──▶ positive conditioning
```

For each shot, generate a still that matches the shot's framing (CU, MS, etc.) using the same character description, then use that still as the I2V seed. Reliability inside that one shot is very high — the model literally starts from the image.

**Workflow:**

1. Build a "character sheet" of stills: front, 3/4, side, back, plus key expressions (neutral, angry, surprised). Generate these as stills, not video.
2. For each shot in your shot list, pick the character sheet entry that best matches the shot framing.
3. (Optional) Use Photoshop / Krita to repose or relight the character sheet entry to better match the shot.
4. Use that customized still as the FLF2V anchor.

This is the most labor-intensive of the no-LoRA techniques, but it's also the most reliable for a small project.

---

## 3.6 Technique 2: IP-Adapter with reference images

[IP-Adapter](https://github.com/cubiq/ComfyUI_IPAdapter_plus) is a ComfyUI extension that lets you condition generation on a reference image without training a LoRA. It encodes the reference into an adapter that biases generation toward matching the reference's identity.

**Status for LTX-2.3 as of 2026**: native LTX IP-Adapter support is limited. Common community workaround: generate stills with SDXL+IP-Adapter (which is mature), then use those stills as FLF2V anchors in LTX-2.3 (technique #3). This sidesteps the lack of native LTX IP-Adapter.

**Recipe — SDXL stills → LTX FLF2V anchors:**

1. In SDXL with IP-Adapter loaded, feed your reference character image (a single still — could be a photo, a drawing, anything).
2. Generate stills for each shot framing using SDXL prompts that match the shot.
3. Use each still as the FLF2V anchor in LTX-2.3 (per §3.5).

Reliability: high for face/identity, medium for costume detail, low for pose continuity (you still need to match poses across stills).

---

## 3.7 Technique 1: Trained character LoRA (gold standard)

If your character recurs across many shots — or across multiple projects — train a character LoRA. Reliability jumps to 80–90%.

**Process at a glance** (full training is its own course):

1. **Dataset** — 30–80 images of the character. Variety in pose, expression, lighting, framing. Single character per image (no other people in frame). Consistent costume.
2. **Caption** — write a caption per image. Use a unique trigger word: `<hero_v1>`. Caption template: `<hero_v1>, [pose], [expression], [framing]`.
3. **Train** — use kohya_ss / OneTrainer with an LTX-2.3 LoRA training config. Typical settings: rank 64–128, learning rate 1e-4, 2000–4000 steps, batch size 1.
4. **Test** — generate test shots using the trigger word in your prompt. If the character looks right and costume holds, ship it. If not, iterate on dataset.

**For an anime / cartoon character, the best trick**: generate your dataset using SDXL + a strong style LoRA + character description, **then** train an LTX-2.3 LoRA on those generated images. You're effectively distilling the SDXL+style+character combination into an LTX video LoRA.

**Trigger word usage** in shot prompts:

```
[STYLE PREAMBLE]. Medium close-up of <hero_v1> standing on a rooftop at night,
rain falling on his shoulders. He turns his head slowly...
```

The trigger word activates the LoRA's character bias. Combined with your style LoRA (module 2), you have repeatable style + repeatable character — the recipe for a coherent project.

**Multi-character LoRAs**: train one LoRA per character. Use multiple `LoraLoaderModelOnly` nodes (workflow guide §10.1) when both characters appear in the same shot. Cap each at strength 0.6 when stacked, 0.8 when solo.

---

## 3.8 Capstone recipes

### Recipe A — single character, no LoRA (zero training, minimal effort)

Best for: first project, one main character, ≤30s short.

1. Write a fixed character description (§3.3).
2. Generate 6–10 character stills (one per shot framing) using SDXL or LTX-2.3 with `frames=1`. Pick the best.
3. Use each still as the FLF2V anchor for its shot (§3.5).
4. Lock the seed across shots.
5. Use identical character description text in every shot prompt.

Expected consistency: 60–70%. Acceptable for a polished first short.

### Recipe B — single character, IP-Adapter assist

Best for: one main character you have a real reference image of (a photo, an illustration).

1. Get a single reference image of your character (or paint one in Krita).
2. In SDXL + IP-Adapter, generate 6–10 stills in your animation style with the reference fed to IP-Adapter at strength 0.7.
3. Use each still as FLF2V anchor in LTX-2.3.
4. Same locked text description across shots.

Expected consistency: 75–85%.

### Recipe C — single character, trained LoRA

Best for: a character you'll reuse across many shorts; or a 2+ minute project.

1. Generate a 50-image dataset of the character in your target style (SDXL + style LoRA + character description).
2. Caption with trigger word.
3. Train LTX-2.3 LoRA (rank 64, ~3000 steps).
4. Use trigger word + style preamble + locked description in every shot.

Expected consistency: 85–95%.

### Recipe D — two recurring characters

Best for: scenes with hero + sidekick, hero + villain.

1. Train a separate LoRA for each character (Recipe C).
2. In shots with both: load both LoRAs at str=0.6 each, mention both trigger words in the prompt.
3. In shots with one: load only that character's LoRA at str=0.7.
4. **For shots where the second character only appears as a silhouette / from behind**: use only the primary character's LoRA. Less LoRA stacking = more stability.

Expected consistency for the named characters: 75–85% in two-shots, 85% in solos.

---

## 3.9 What about props, environments, and vehicles?

Same techniques apply. A consistent hovercar across shots needs either:
- A locked text description (low reliability), or
- A reference image used as anchor / IP-Adapter source (medium), or
- A trained "object LoRA" on multi-angle renders of the prop (high — this is the standard concept-art workflow).

For a course project, accept that backgrounds will drift. **Mitigation**: shoot environment establishing shots as separate "plates" without characters, then re-use those plates for backgrounds when a character is composited in via post (module 5). This is how a lot of pro AI shorts get away with consistent locations.

---

## 3.10 Color and wardrobe consistency

Even with a perfect character LoRA, the model will drift on costume color saturation, hue, and small details (a belt buckle, a logo). Mitigations:

1. **Color description in every prompt**: name the exact color words ("cherry red", not "red"; "navy blue", not "blue"). Hex codes don't help; named shades do.
2. **Light direction**: "lit from screen-left by a single warm key light" anchors the lighting setup, which anchors apparent costume color.
3. **Post-color-grade ALL shots together** (module 5). Don't try to fix per-shot. Apply one color grade at the scene level and let it harmonize.

For exact logo / detail work (a chest emblem, a specific pattern), there's no AI fix at this scale. **Pre-mask the area in post and overlay the correct logo manually.** That's what professional pipelines do too.

---

## 3.11 Reality check

Even the best techniques don't get you 100% consistency. A trained LoRA at strength 0.8 with a locked prompt and an anchor frame will still produce visible drift across 50 shots. **Plan around this**:

- **Hide drift with cuts.** Faster cutting hides per-shot variation. Slow shots expose it.
- **Hide drift with motion.** A character mid-action is forgivable; a character holding still in close-up isn't.
- **Hide drift with lighting.** Strong shadow on the face hides feature drift.
- **Hide drift with framing.** OTS shots show less of the face than full close-ups. Reaction shots can be partial silhouettes.
- **Use the worst shots for cutaways.** When you generate a shot that looks "off-model", find an excuse to use it as a brief insert / cutaway / dream-flash where deviation reads as stylistic.

Pro animation also has model sheets and continuity supervisors. Without those, you're trading some perfection for speed. Embrace it as a stylistic choice.

---

## 3.12 Exercise

1. Pick your protagonist character. Write the locked text description (§3.3).
2. Generate a character sheet: 6 stills (front, 3/4, side, neutral expression, angry expression, surprised expression). Tools: SDXL or LTX-2.3 single-frame.
3. Pick **one** shot from your shot list that requires this character. Generate it three times using different recipes:
   - Pure text only (Recipe baseline)
   - Recipe A (FLF2V with character sheet anchor)
   - Recipe B (IP-Adapter SDXL → FLF2V anchor) — if you have IP-Adapter installed
4. Compare. Pick the technique you'll use for the remaining shots.

When you have a working character recipe, you're ready for module 4 — concrete shot workflows.
