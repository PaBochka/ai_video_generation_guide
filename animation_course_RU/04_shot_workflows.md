# Модуль 4 — Shot Workflows

Этот модуль — мост между планированием (модули 1–3) и пост-продакшном (модуль 5). Для каждого типа кадра из вашего shot list модуль даёт конкретный рецепт LTX-2.3.

Все рецепты предполагают, что у вас:
- Рабочая сборка LTX-2.3 по workflow guide §9.
- Знание workflow guide §10 (LoRA, IC-LoRA, anchor-фреймы, looping sampler).
- Ваш style preamble из модуля 2.
- Описание / sheet персонажа из модуля 3.

Если рецепт говорит «use Recipe A character technique», он ссылается на модуль 3.

> **Связанные разделы ComfyUI Guide** — [ComfyUI Video Workflow Guide](../ComfyUI_Video_Workflow_Guide.md):
> - **§9.4** — каноничная двухэтапная T2V цепочка нод (движок, в который включается каждый рецепт).
> - **§9.5** — I2V-цепочка с `LTXVPreprocess` + `LTXVImgToVideoInplace`. Используется в рецептах close-up, OTS и continuation.
> - **§9.7** — пресеты сэмплера/шедулера по этапам (dev vs. distilled, `LTXVScheduler` vs. `ManualSigmas`).
> - **§9.12** — разобранный 4-секундный кинематографический T2V-пример (карта 24 GB). Базовые настройки для большинства рецептов этого модуля.
> - **§10.2** — camera-control LoRA, привязанные к типам кадров (dolly-in → push-in CU, jib-up → rising EST, static → dialogue MS).
> - **§10.3** — IC-LoRA (depth/pose/Canny/motion-track). Используются для экшн-кадров, требующих структурного контроля.
> - **§10.5** — `LTXVLoopingSampler` для кадров длиннее ~10 секунд и для title/loop-рецепта.
> - **§10.9** — совместимость quick-reference (что с чем стакается). Проверяйте перед комбинированием camera LoRA + IC-LoRA + anchor в одном рецепте.

---

## 4.1 Общий шаблон рецепта кадра

Каждый кадр, независимо от типа, следует этой структуре:

```
[STYLE PREAMBLE]   (из модуля 2 — копипаст, никогда не перефразируется)
[CHARACTER DESCRIPTION(S)]   (из модуля 3 — копипаст)
[SHOT FRAMING]   (напр., «Medium close-up of...»)
[ACTION]   (что происходит в этом кадре)
[CAMERA]   (движение камеры, обычно совпадающее с camera LoRA)
[LIGHTING + ATMOSPHERE]   (специфика этого кадра)
[SOUND CUE]   (foreground звук + амбиент)
```

LoRA-стек:
- Distilled-speed LoRA (всегда, по workflow guide §9 & §10)
- Style LoRA (если есть, по модулю 2)
- Camera LoRA (по типу кадра)
- Character LoRA (если обучили, по модулю 3)

Потолок общего бюджета силы LoRA — около 1.8. Выше — деградация.

---

## 4.2 Рецепт — Establishing shot (EST)

**Применение**: открытие сцены, задание локации, без крупняка персонажа.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | Чистый T2V (без I2V-якоря) |
| Разрешение | 832×480 (16:9) или 768×512 (3:2) |
| Кадры | 97 (≈4 с @ 25fps) |
| Camera LoRA | `static` ИЛИ `jib-up` (медленное открытие) |
| Style LoRA | да |
| Character LoRA | НЕТ (в кадре нет персонажа) |
| Сэмплер | distilled `ManualSigmas` 8 шагов, `CFGGuider cfg=1.0` |

**Шаблон промпта:**

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

**Конкретное заполнение (стиль Spider-Man 94):**

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

Настройка:
- Если слишком generic / не хватает «место»: добавьте 2–3 конкретных ориентира («the Daily Bugle building visible in the distance», «a 1990s yellow checker cab in foreground»)
- Если слишком перегружено: уменьшите количество элементов
- Если слишком чисто: добавьте «wet textures, debris, peeling posters, weathered surfaces»

---

## 4.3 Рецепт — Wide shot с персонажем (WS)

**Применение**: ввод персонажа в окружении, экшн-сцены.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | T2V при свежей генерации; I2V при привязке к предыдущему кадру |
| Разрешение | 832×480 |
| Кадры | 73–97 (3–4 с) |
| Camera LoRA | `static`, `dolly-in` или `jib-up` |
| Character LoRA | да (одна) |
| Anchor-фрейм | рекомендован — «wide» статик из character sheet |

**Шаблон промпта:**

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

**Совет**: если этот кадр следует за establishing-кадром той же локации, **явно снова упомяните элементы локации** в этом промпте. Не рассчитывайте, что AI перенесёт их из предыдущей генерации. Он не переносит.

---

## 4.4 Рецепт — Medium / Medium close-up (MS / MCU)

**Применение**: диалог, эмоциональные биты, моменты персонажа.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | I2V с character sheet-статика строго рекомендуется |
| Разрешение | 768×512 или 704×512 |
| Кадры | 49–97 (2–4 с) |
| Camera LoRA | `static` (редкое исключение: медленный `dolly-in` для драматического reveal) |
| Character LoRA | да |
| Anchor-фрейм | статик medium / MCU из character sheet |

**Шаблон промпта:**

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

**Критично**: держите камеру статичной для MCU-диалога. AI-движение камеры в плотной рамке обычно выглядит uncanny.

**Реалити-чек по диалогу**: LTX-2.3 генерирует речеподобное аудио без lip-sync. Для реального диалога:
- Генерируйте кадр молча (или только с амбиентом).
- Запишите / синтезируйте диалог отдельно (ElevenLabs).
- Lip-sync в посте (модуль 5 покрывает Wav2Lip / SadTalker).

---

## 4.5 Рецепт — Close-up / Extreme close-up (CU / ECU)

**Применение**: эмоциональный пик, момент решения, reveal детали.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | I2V обязателен (text-only CU — худший случай для дрейфа персонажа) |
| Разрешение | 704×512 |
| Кадры | 49 (2 с) — обычно коротко |
| Camera LoRA | `static` |
| Character LoRA | да |
| Anchor-фрейм | качественный close-up-статик персонажа — генерируйте тщательно |

**Шаблон промпта:**

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

CU лучше всего работают в held cels с микро-движением. Не просите больших действий в CU.

---

## 4.6 Рецепт — Action-кадр

**Применение**: бой, погоня, прыжок, что-то кинетическое.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | T2V для свежего экшена; I2V при продолжении из предыдущего бита |
| Разрешение | 832×480 |
| Кадры | 49–73 (2–3 с — экшн монтируется быстро) |
| Camera LoRA | соответствует движению (`dolly-in`, `dolly-out`, `dolly-left` и т.д.) |
| IC-LoRA | **Motion-Track**, если можете нарисовать желаемый путь движения |
| Character LoRA | да |

**Шаблон промпта:**

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

**Motion-Track IC-LoRA — ваше секретное оружие для экшена.** Workflow guide §10.3 покрывает подключение. Вы буквально рисуете траекторию движения в `LTXVSparseTrackEditor`, и AI следует ей. Это единственный самый надёжный способ получить конкретную хореографию действия.

---

## 4.7 Рецепт — Over-the-shoulder (OTS)

**Применение**: диалог между двумя персонажами, реакция без показа обоих лиц.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | I2V с anchor-фреймом обязательно |
| Разрешение | 768×512 |
| Кадры | 49–97 |
| Camera LoRA | `static` (почти всегда) |
| Character LoRA | оба (если обучены оба); иначе — только основного + второго в тексте |

**Шаблон промпта:**

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

**Совет**: силуэтные персонажи проще полностью отрисованных. Используйте OTS-рамку как способ показать двух персонажей без оплаты consistency-налога на обоих.

---

## 4.8 Рецепт — Insert / Cutaway

**Применение**: короткий кадр детали — экран телефона, часы, рука, берущая оружие.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | T2V (обычно без персонажа — если рука: очень плотная рамка) |
| Разрешение | 704×512 или квадратный 640×640 |
| Кадры | 25–49 (1–2 с) — инсерты КОРОТКИЕ |
| Camera LoRA | `static` или очень лёгкий `dolly-in` |
| Character LoRA | обычно нет |
| Style LoRA | да |

**Шаблон промпта:**

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

**Инсерты — простые.** Часто это самые чистые кадры в AI-короткометражке, потому что нет персонажей и минимум движения. Используйте щедро — они дают передышку между более трудными персонажными кадрами.

---

## 4.9 Рецепт — Реакция-кадр

**Применение**: персонаж A что-то говорит/делает; склейка на реакцию персонажа B.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | I2V с anchor-фреймом |
| Разрешение | 704×512 (CU) или 768×512 (MCU) |
| Кадры | 25–49 (1–2 с — реакции быстрые) |
| Camera LoRA | `static` |
| Character LoRA | да |

**Шаблон промпта:**

```
[STYLE PREAMBLE]
[CHARACTER DESCRIPTION]

[CU / MCU] of [character], starts in [neutral expression], then [reaction —
eyes widen, mouth opens slightly, slight head turn].

Camera locked.

Same lighting as previous shot.

Single sound cue — sharp intake of breath, soft "oh", or absolute silence.
```

**Совет**: 1-секундная молчаливая реакция часто сильнее 3-секундной с аудио. Монтируйте туго.

---

## 4.10 Рецепт — POV-кадр

**Применение**: показать, что персонаж видит с его точки зрения.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | T2V |
| Разрешение | 832×480 |
| Кадры | 49–97 |
| Camera LoRA | соответствует движению головы/тела POV-персонажа |
| Character LoRA | НЕТ (мы смотрим его глазами) |

**Шаблон промпта:**

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

POV — мощный инструмент, потому что **вы полностью обходите проблему консистентности персонажа** в этом кадре — его нет в рамке.

---

## 4.11 Рецепт — Title card / End slate

**Применение**: открытие или закрытие короткометражки с лого / титрами.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | T2V или чистый статик, удерживаемый во времени |
| Разрешение | 832×480 |
| Кадры | 49–97 (2–4 с) |
| Camera LoRA | `static` |
| Все прочие LoRA | OFF или минимально |

LTX-2.3 не умеет надёжно рендерить конкретный текст на экране. Лучший подход:

1. Сгенерируйте «фон» титульного кадра без текста через T2V (атмосферный, под сцену).
2. **Добавьте текст титра в посте** с помощью title tool в DaVinci Resolve / Premiere.

Не боритесь с моделью на рендеринге текста — она проигрывает.

---

## 4.12 Looping-кадр (для музыкальных клипов, capstone вариант 3)

**Применение**: 8-секундный клип, который бесшовно зацикливается.

**Настройки:**

| Параметр | Значение |
|----------|----------|
| Тип | I2V с одним и тем же изображением как first И last anchor (workflow guide §10.5) |
| Разрешение | 768×512 |
| Кадры | 161 (~6.4 с @ 25fps — держите скромно) |
| Camera LoRA | `static` (looping с движениями камеры намного труднее) |
| Сэмплер | обычный `SamplerCustomAdvanced` работает; `LTXVLoopingSampler` если длиннее |

**Рецепт:**

1. Сгенерируйте одно высококачественное статичное изображение персонажа / сцены.
2. Используйте его как `LTXVAddGuide(frame_idx=0, str=1.0)` И `LTXVAddGuide(frame_idx=-1, str=1.0)`.
3. Промптите медленное циклическое движение («hair gently sways in wind», «neon sign flickers», «rain falls steadily») — зациклить проще, чем направленное движение.
4. В посте добавьте 3-кадровый crossfade между концом и началом цикла, чтобы скрыть шов.

---

## 4.13 Стратегия итерации

Для каждого кадра следуйте циклу:

1. **Залочьте сид** в `RandomNoise`. Не меняйте во время итерации.
2. **Генерируйте сначала в низком разрешении**: 512×384 или 640×384, 49 кадров. На 4090 — 30 с.
3. **Итерируйте на промпте**, пока рамка, движение и стиль не встанут. На итерацию — меняйте ОДНО.
4. **Когда залочено** — перегенерируйте в целевом разрешении. Только тогда запускайте stage-2 апскейлер.
5. **Сохраните промпт и сид** в файл `shots_done.md`. Пригодятся, когда будете перерендеривать через 2 недели.

Не итерируйте в полном разрешении. Не меняйте пять вещей между итерациями. Не снимайте лок с сида, пока не довольны рамкой — смена сида конфаундит смену промпта.

---

## 4.14 Когда бросать кадр

После ~15 итераций без прогресса — **стоп**. Либо:

- **Перекадрируйте.** Может, ваш CU должен быть MCU; ваш wide — medium. Иногда AI просто не делает конкретную композицию; рефрейминг — на ту, которую сделает.
- **Подмените обходом.** Вместо «two characters fighting in this shot» — два соло-кадра с реакция-инсертами. Вместо непрерывного 8-секундного экшн-кадра — три 2-секундных склейки.
- **Разделите кадр.** Генерируйте экшн кусками и сшивайте.

Shot list из модуля 1 — это цель, а не контракт. Адаптируйтесь по мере того, как учитесь, что AI может и не может для вашего стиля.

---

## 4.15 Организация проекта

Для каждого кадра сохраняйте:

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
│   ├── shot_001_T2V_EST.json   (сохранённый ComfyUI workflow под кадр)
│   ├── shot_002_I2V_WS.json
│   └── ...
├── renders/
│   ├── shot_001/
│   │   ├── v1.mp4
│   │   ├── v2.mp4
│   │   └── final.mp4   (финалист)
│   ├── shot_002/
│   └── ...
├── audio/
│   ├── dialogue/
│   ├── music/
│   └── sfx/
└── final/
    ├── timeline.drp     (DaVinci Resolve проект)
    └── export.mp4
```

Без этой структуры вы сойдёте с ума. Настраивайте с первого дня.

---

## 4.16 Упражнение

Сгенерируйте каждый кадр из shot list рецептом, соответствующим его типу. Про аудио пока не беспокойтесь — добавите в модуле 5. Получите чистую визуалку каждого кадра.

Сохраняйте каждый `final.mp4` в своей папке `renders/shot_XXX/`. Когда все кадры готовы — вы готовы к пост-продакшену.
