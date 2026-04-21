# Модуль 2 — Воспроизведение стиля

Этот модуль — про то, как заставить сгенерированный материал **выглядеть** как конкретный анимационный стиль. Это разница между «AI-видео» и «смотрибельной анимационной короткометражкой».

Есть три уровня контроля в порядке влияния:

1. **Каркас промпта** — язык, смещающий модель в сторону стиля. Бесплатно, быстро, даёт ~60% look.
2. **Style LoRA** — адаптерные веса, обученные на этом стиле. Добавляет ещё ~25%. Многие скачиваются; некоторые обучают самостоятельно.
3. **Post-process style transfer** — применение стиля к уже сгенерированному видео. Добавляет финальные 10–15%, но самое медленное и хрупкое.

Большинство курсов останавливаются на #1. Мы покроем все три.

> **Связанные разделы ComfyUI Guide** — [ComfyUI Video Workflow Guide](../ComfyUI_Video_Workflow_Guide.md):
> - **§9.4** — Lightricks-конвенция короткого concept-level негативного промпта (`pc game, console game, video game, cartoon, childish, ugly`). Не тащите сюда SD-шные портянки.
> - **§10.1** — механика стекирования LoRA (`LoraLoaderModelOnly` vs. `LTXICLoRALoaderModelOnly`, практический потолок в 3 LoRA, рекомендации по силе для комбинаций style + speed + IC).
> - **§11** — промптинг Gemma (пятиабзацная структура, эмфаза без `(word:1.4)`, привязка к порядку слов). Style preamble идёт в ¶1.

---

## 2.1 Анатомия стиля — что определяет «look»

Прежде чем воспроизвести стиль, нужно уметь описывать его в конкретных визуальных терминах. «Выглядит как Spider-Man 1994» — слишком расплывчато для промпта. Разложите по компонентам:

| Измерение | Примеры значений |
|-----------|------------------|
| **Линия** | Отсутствует / тонкая карандашная / толстая чёрная tushь / цифровой аутлайн / без аутлайна |
| **Shading** | Cel-shaded (плоские зоны) / мягкий градиент / штриховка / живописный |
| **Цветовая палитра** | Насыщенные первичные / приглушённые earth tones / пастель / монохром с акцентом / неон |
| **Уровень детализации** | Редкая (limited animation) / умеренная / гиперполная (Akira) |
| **Стиль фона** | Painted matte / чистый cel / фото-композит / абстракция |
| **Характер движения** | Плавное (12+ fps анимация) / ограниченное (held cels) / импакт-ориентированное (аниме-драка) |
| **Композиция** | Комикс-панель / кинематографичный широкий экран / ТВ 4:3 / вертикально-мобильная |
| **Свет** | Плоский / драматический chiaroscuro / мягкий натуралистичный / неоновый practical |
| **Эра** | 80-е зернистость + хром. аберрация / 90-е VHS / 2000-е цифровая резкость / современный HDR |

**Упражнение**: возьмите целевой референс (один кадр из вашего любимого шоу) и заполните таблицу. То, что вы запишете — это словарь вашего промпта.

---

## 2.2 Каркас промпта — универсальный шаблон

Каждый промпт кадра для анимации должен следовать примерно этой структуре:

```
[STYLE PREAMBLE]. [SHOT TYPE + SUBJECT]. [ACTION + CAMERA]. [LIGHTING + ATMOSPHERE]. [SOUND CUE].
```

Конкретное заполнение:

```
1990s Saturday-morning American animated TV series, bold black ink outlines, flat
cel-shaded coloring, saturated comic-book color palette, limited animation feel.

Medium close-up of a masked vigilante in red and blue costume standing on a rooftop
at night, rain falling on his shoulders.

He slowly turns his head to the right, looking down toward the street below. The
camera is locked-off, no movement.

Wet asphalt reflections, distant neon signs in cyan and magenta, dramatic backlight
from the city below silhouetting his figure.

Light rain hisses against the rooftop, distant city ambience, a faint synth pad in
the background.
```

Это намеренно многословно. Первый абзац (**style preamble**) одинаков для каждого кадра в проекте — вы буквально копипастите его. Остальное меняется от кадра к кадру.

**Негативный промпт** (Lightricks-конвенция из workflow guide §9.4):

```
pc game, console game, video game, cartoon, childish, ugly
```

Для специфической антистилевой чистки добавляйте только concept-level-термины — не SD-шный список «blurry, distorted, extra fingers». Если ваш стилевой промпт говорит «1990s American animation», а результат отдаёт аниме — добавьте `anime, manga, japanese animation` в негатив.

---

## 2.3 Style preamble для трёх референсных стилей

### A. Spider-Man: The Animated Series (1994) — preamble

```
A 1994 American animated superhero TV series, hand-drawn cel animation, bold black
ink outlines on every form, flat saturated coloring with no gradients, limited
TV-budget animation, comic-book composition, warm color palette dominated by browns
and deep purples with saturated red and blue costume accents, dramatic
chiaroscuro lighting from off-screen sources, mid-1990s Saturday-morning network
broadcast aesthetic, slight VHS softness.
```

Добавки в анти-промпт: `anime, manga, photorealistic, 3D rendered, modern digital animation, smooth animation`.

Заметки по настройке:
- Если результат ощущается слишком современным: добавьте «1990s analog broadcast», «TV cel animation», «vhs era»
- Если не хватает жирной tushи: добавьте «thick black outlines on all shapes», «comic book ink»
- Если цвет ощущается блёклым: добавьте «saturated comic book color palette»

### B. 80s OVA аниме — preamble

```
A 1988 Japanese OVA anime film, hand-drawn cel animation in the style of late-1980s
Tokyo studios, detailed line work with subtle pencil texture, painted backgrounds
with extreme architectural detail, cool color palette dominated by cyan, deep
indigo, and magenta neon accents, cinematic widescreen 16:9 composition, dramatic
lens flares from practical light sources, moody atmospheric lighting, slight
analog film grain, anamorphic lens character, Akira-era visual sophistication.
```

Анти-промпт: `american cartoon, modern digital anime, 3D rendered, smooth flash animation, TV budget`.

Настройка:
- Для мокро-асфальтового киберпанка: «rain, wet streets, neon reflections, fog»
- Для живописного фона: «matte painted background, watercolor washes»
- Если слишком современно: «cel film stock, 1988, OVA»

### C. Современное cel-shaded аниме — preamble

```
A modern Japanese television anime in the style of contemporary action series,
clean digital cel-shading with crisp line work, high-contrast dramatic lighting,
saturated character colors against painterly background art, occasional impact
frames with bold motion lines, exaggerated character expressions, cinematic
camera angles, polished modern Japanese animation production, 24fps anime feel.
```

Анти-промпт: `realistic, 3D rendered, american cartoon, retro anime, vhs grain`.

Настройка:
- Для экшн-сцен: «speed lines, impact frame, dynamic angle»
- Для диалогов: «soft anime lighting, expressive eyes»

---

## 2.4 «Consistency lock» — не меняйте каркас

**Не переписывайте style preamble для каждого кадра.** Копипастите выбранный preamble в начало промпта каждого кадра. Меняются только строки action/subject/camera/atmosphere.

Это единственный самый значимый фактор межкадровой стилистической консистентности. Модель видит одни и те же открывающие токены и смещается одинаково каждый раз.

Если хотите подкрутить preamble — сделайте это один раз, затем **перерендерьте каждый кадр проекта** с новым preamble. Не смешивайте версии preamble в одном финальном монтаже.

---

## 2.5 Style LoRA — когда промптов недостаточно

Промпты дают 60% пути. Остальные 40% — от LoRA, обученных на целевом стиле.

**Где искать LoRA:**
- [Civitai](https://civitai.com) — самая большая библиотека LoRA; фильтр по базовой модели. По состоянию на 2026 нативные LTX-2.3 LoRA всё ещё редки; многие для SDXL / Flux и требуют конверсии или несовместимы с видео.
- [HuggingFace](https://huggingface.co) — поиск `LTX video LoRA` или `LTX-2 LoRA`. Lightricks публикует камеру и IC-LoRA там.
- [Tensorart](https://tensor.art) — аналогично civitai.

**По состоянию на 2026 экосистема LoRA под LTX-2.3 тонкая.** Большинство стилевых LoRA всё ещё под SDXL/Flux. У вас три варианта:

1. Использовать естественный стилистический диапазон модели через промпт (покрыто выше) — работает для общих аниме/мульт-стилей.
2. Обучить собственную style LoRA под LTX-2.3 (покрыто в §2.7).
3. Генерировать в базовом стиле LTX-2.3, затем применять style transfer в посте (покрыто в §2.8).

### Загрузка style LoRA на LTX-2.3

Используйте `LoraLoaderModelOnly` (workflow guide §10.1). Подключайте после speed LoRA:

```
CheckpointLoaderSimple
  └── MODEL ──▶ LoraLoaderModelOnly (distilled, str=0.5)
                  └── MODEL ──▶ LoraLoaderModelOnly (style LoRA, str=0.4–0.7)
                                  └── MODEL ──▶ guider / sampler
```

Настройка силы:
- **Style LoRA, обученные на стилях**: начните с 0.4. Обучались на статичных изображениях, видеосвязность страдает на бóльших силах.
- **Style LoRA, обученные на видео**: начните с 0.6.
- Если результат похож на LoRA, но движение размазано: снижайте.
- Если результат похож на базовую модель с намёком на стиль: повышайте.

Потолок 0.8, если только вы сами не обучили LoRA и не знаете её толерантность.

---

## 2.6 Комбинирование style LoRA

Вы можете стакать две style LoRA (например, `cel-shaded look` + `90s color palette`). Ограничения:

- **Максимум 2 style LoRA** в дополнение к speed LoRA. Три и больше начинают конфликтовать.
- **Суммарный бюджет силы ≈ 1.5**. Если style LoRA A на 0.7, style LoRA B — максимум 0.3–0.5.
- **Одна обучающая база** важна. LoRA, обученные на разных версиях LTX, плохо смешиваются; LoRA разных семейств моделей (LTX vs Wan vs Hunyuan) не смешиваются вовсе.

Примеры комбинаций:
- `90s_animation_style` (0.6) + `wet_neon_atmosphere` (0.4) = Spider-Man 94 под дождём
- `anime_lineart` (0.7) + `cyberpunk_palette` (0.3) = синтетический Akira

---

## 2.7 Обучение собственной style LoRA (обзор)

Полностью тема за рамками этого раздела, но на высоком уровне:

1. **Соберите датасет** — 50–200 стилов (или коротких видеоклипов) в целевом стиле. Качество важнее количества. Без логотипов, водяных знаков, текста.
2. **Caption** — пишите или генерируйте подпись к каждому изображению. Инструменты: [WD14 Tagger](https://github.com/picobyte/stable-diffusion-webui-wd14-tagger), [BLIP-2] или вручную.
3. **Обучите** — [kohya_ss](https://github.com/kohya-ss/sd-scripts), [OneTrainer](https://github.com/Nerogar/OneTrainer) или [diffusion-pipe](https://github.com/tdrussell/diffusion-pipe) с тренинг-конфигом LTX-2.3. Ожидайте 2–8 часов на 24 GB GPU для LoRA ранга 128.
4. **Тест** — генерируйте sample-кадры с фиксированным каркасом и разной силой LoRA.
5. **Итерация** — плохие результаты обычно = проблемы датасета. Перепишите caption'ы, почистите, переобучите.

Для учебного проекта (вашей первой 30-секундной короткометражки) **пока не тренируйте style LoRA**. Используйте каркас промпта + пост-обработку. Оставьте тренировку LoRA на потом, когда уже отгрузите одну короткометражку и будете знать, что именно не работает.

---

## 2.8 Post-process style transfer

Когда промптов и LoRA недостаточно, применяйте стиль после генерации.

**Варианты инструментов:**

| Инструмент | Что делает | Стоимость | Качество |
|-----------|------------|-----------|----------|
| [RunwayML Gen-Restyle](https://runwayml.com) | Применяет референсный стиль к видеоклипу | Платно | Хорошо для стилизованных look |
| [DomoAI](https://domoai.app) | Аниме / мультяшный restyle | Платно | Очень хорошо для аниме-фикации |
| [EBSynth](https://ebsynth.com) | Распространяет расписанный вручную кейфрейм по видео | Бесплатно | Лучшее для настоящего cel-look; занудно |
| [Diffusion-based vid2vid в ComfyUI](https://github.com/Fannovel16/comfyui_controlnet_aux) | AnimateDiff / SDXL с ControlNet покадрово | Бесплатно | Несогласованно между кадрами |
| Ручной ротоскоп + рисование | Покадрово в Photoshop / Krita | Бесплатно, очень медленно | Лучшее качество, непрактично для >10 с |

**Рецепт EBSynth** (бесплатно, неожиданно хорош для cel/аниме-look):

1. Сгенерируйте видео в LTX-2.3 (полу-реалистичный базовый стиль).
2. Экспортируйте 0-й кадр как статик.
3. Вручную перерисуйте или AI-restyle этот один кадр под целевой look.
4. Скормите оригинальное видео + стилизованный кадр в EBSynth.
5. EBSynth распространит стиль по остальным кадрам через optical flow.

Оговорки: EBSynth деградирует на длинных клипах (>2 с) и ломается на быстром движении. Используйте для коротких held-кадров; не для экшена.

**Для учебного проекта рекомендованный путь:**
- Spider-Man 94 / cel-shaded → попробуйте DomoAI или RunwayML Restyle на каждом кадре
- 80s аниме → сильный промпт + IC-LoRA Union для линии + DomoAI на финиш
- Современное аниме → сильный промпт + style LoRA с civitai (ищите «anime LTX»)

---

## 2.9 Стилевая непрерывность между кадрами — практический чек-лист

Когда все кадры shot list сгенерированы и вы их сшиваете, пройдитесь по чек-листу:

- [ ] Один и тот же style preamble использован в каждом промпте кадра
- [ ] Один и тот же негативный промпт во всех кадрах
- [ ] Одна и та же speed LoRA + те же style LoRA на той же силе во всех кадрах
- [ ] Одно и то же разрешение и аспект
- [ ] Один FPS в `LTXVConditioning` и `VideoCombine`
- [ ] Цветокор применён ко ВСЕМ кадрам в посте (модуль 5) — AI будет цветодрейфовать даже при идентичных промптах
- [ ] Направление света описано единообразно во всех промптах (например, «key light from camera-left» везде)
- [ ] Время суток / погода описаны идентично через сцену
- [ ] Если появляется рекуррентный персонаж: идентичное описание персонажа во всех промптах (модуль 3)

Если что-то из этого не совпадает — ваш финальный монтаж будет выглядеть как пять разных короткометражек, скреплённых степлером.

---

## 2.10 Стиль для звука

Не забывайте — аудио часть стиля. Spider-Man 94 кадр с кинематографическим саундтреком Hans Zimmer звучит неправильно; Ghibli кадр с chiptune звучит неправильно.

В LTX-2.3 аудио генерируется из того же текстового промпта, что и видео. Поэтому ваш style preamble должен подразумевать и аудио-эстетику:

- «1990s Saturday-morning American animated TV series» → тянет к синт-саундтрекам, MIDI-трубам, мультяшному foley
- «1988 Japanese OVA anime» → тянет к аналоговым синтам, большим хорам, 80-м драм-машинам
- «Modern Japanese anime» → тянет к оркестровому гибриду, J-поп-влиянию

Если AI-аудио не попадает, **замените его в посте** (модуль 5). Модуль 5 покрывает Suno / Udio для музыки и ElevenLabs для диалогов.

---

## 2.11 Упражнение

1. Выберите стиль.
2. Напишите style preamble. Возьмите один из шаблонов выше как стартовую точку.
3. Сгенерируйте три тестовых кадра из shot list (EST, CU, action) с одним и тем же preamble.
4. Сравните бок о бок. Они узнаваемо в одном стиле?
5. Если да → в модуль 3.
6. Если нет → настраивайте preamble. Добавьте недостающее измерение (линия? цвет? эра?). Перетестите.

Не двигайтесь дальше, пока тестовые кадры не начнут ощущаться как принадлежащие одной анимационной франшизе.
