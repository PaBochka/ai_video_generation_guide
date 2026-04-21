# Модуль 6 — Техники промптинга

Модули 2–4 дали вам каркасы промптов. Этот модуль — то ремесло, которое лежит под ними: почему эти каркасы работают, как писать свои и какие приёмы с высокой отдачей отделяют «AI-вывод» от «того кадра, который вам был нужен».

Промптинг для LTX-2.3 отличается от промптинга для SDXL, Flux или Midjourney. Большинство онлайн-гайдов описывают CLIP-ориентированный промптинг (каша из ключевых слов, синтаксис весов `(word:1.4)`, теги через запятую). **Эта грамматика здесь не применима.** LTX-2.3 использует Gemma 3 12B Instruct — полноценную языковую модель — в качестве текстового энкодера. Gemma читает естественный язык.

Этот модуль — про то, что действительно работает.

> **Связанные разделы ComfyUI Guide** — [ComfyUI Video Workflow Guide](../ComfyUI_Video_Workflow_Guide.md):
> - **§11** — Prompting Gemma for LTX-2.3 (vs. CLIP). Сжатый технический справочник, зеркалящий этот модуль — различия в грамматике CLIP vs Gemma, пятиабзацная структура, эмфаза без весов, поддержка отрицания/последовательностей/счёта/сравнений и протокол итерации.
> - **§9.11** — audio prompting. Отдельного поля для аудио-кондиционирования нет; звуковые подсказки живут естественным языком в том же позитивном промпте. Пятый абзац этого модуля (sound) построен на тех правилах.
> - **§9.4** — конвенция Lightricks «короткий концептуальный негатив» и шаблонные промпты из официальных JSON. Используйте их как калибровочный эталон для ваших собственных черновиков.

---

## 6.1 Как модель читает ваш промпт

Gemma 3 12B — декодер-LLM, а не CLIP. Практические следствия:

| Конвенция SD / CLIP | Реальность Gemma / LTX-2.3 |
|---------------------|----------------------------|
| Токены через запятую (`red dress, long hair, forest, night`) | Работает, но **хуже** полноценных предложений. Gemma понимает прозу. |
| Синтаксис эмфазы `(word:1.4)` | **Не парсится как вес.** Gemma читает скобки как пунктуацию. Эмфаза — через повторение и формулировку. |
| Лимит контекста 77 токенов | Не проблема. Контекст Gemma достаточен для абзацев. |
| Короткие промпты («anime girl, blue eyes, sword») | Работают плохо. Gemma рассчитана на длинный контекст и такие промпты недоспецифицированы. |
| Негатив = длинный список | Конвенция Lightricks — короткий концептуальный негатив (см. §9.4 workflow-гайда). |
| Порядок слов почти не важен | Порядок слов **важен**. Первое предложение якорит стиль; следующие наслаивают детали. |
| Трюки с дропаутом/эмфазой токенов | Лучше всего работает обычный язык — описывайте, а не перечисляйте. |

**Итог**: пишите как кинооператор, объясняющий кадр команде на площадке, а не как SD-пауэр-юзер, расставляющий теги картинке.

---

## 6.2 Пятиабзацный промпт кадра

Любой промпт на кадр должен вписываться в эту структуру. Абзацы существуют не просто так — Gemma читает пустые строки как семантические границы, что помогает ей организовать информацию.

```
¶1  STYLE + MEDIUM          (копипаста через весь проект)
¶2  SUBJECT + FRAMING       (кто/что, в каком типе кадра)
¶3  ACTION + CAMERA         (что происходит + как ведёт себя камера)
¶4  LIGHTING + ATMOSPHERE   (внешний вид сцены)
¶5  SOUND                   (звуковые слои)
```

Это тот же каркас, что и в модуле 4, явно сформулированный как грамматика. Когда пишете новый кадр:

1. Скопируйте ¶1 из вашего style bible — никогда не переписывайте заново.
2. Скопируйте блок описания персонажей в ¶2.
3. ¶3–5 пишите с нуля под этот кадр.

**Не склеивайте абзацы.** Сплошной текст работает хуже, чем те же предложения, разделённые пустыми строками. Проверено многократно.

---

## 6.3 Глаголы движения — самый недооценённый элемент промпта

Слабые глаголы дают статичный, призрачный результат. Сильные глаголы дают движение. Это фикс №1 для «почему у меня видео будто замороженное».

| Слабо | Сильно | Что меняется |
|-------|--------|--------------|
| "The character is in the forest" | "The character walks through the forest" | Подразумевает движение камеры и тела |
| "She looks sad" | "She lowers her gaze and exhales slowly" | Конкретная микромоторика |
| "Rain falls" | "Heavy rain streaks diagonally across the frame" | Направление + скорость |
| "The car moves" | "The car accelerates hard, tires spinning, toward camera-right" | Направление + физика |
| "Light in the room" | "Morning sunlight spills across the wooden floor as dust motes drift" | Движение самого качества света |

**Правило**: каждое предложение в третьем абзаце (action) должно содержать хотя бы один конкретный глагол движения. Если в предложении нет движения, AI генерирует стоп-кадр.

**Глаголы темпа**: "slowly", "suddenly", "gradually", "smoothly", "abruptly" — они влияют на темп сгенерированного движения. Используйте осознанно.

---

## 6.4 Камерный язык — мыслите как оператор

У вас есть полноценный словарь камерных терминов. Используйте его. Модель обучалась на киноданных — она откликается на те же термины, которыми DP говорит на площадке.

| Термин | Что говорит |
|--------|-------------|
| "locked-off camera" | Камера не движется. Парится с LoRA `static`. |
| "handheld" | Органический трясь и плавание. Хорошо для экшена/реализма. |
| "dolly-in / dolly-out" | Камера физически движется к/от. Парится с dolly-LoRA. |
| "pushing in" / "pulling back" | То же, что dolly-in/out, но более литературно. |
| "tracking shot" | Камера движется параллельно субъекту. |
| "crane shot" / "jib rises" | Камера поднимается. Парится с `jib-up`. |
| "tilt down / pan left" | Вращательные движения (LoRA не выпущена; только промпт). |
| "rack focus" | Фокус переходит с переднего плана на задний. |
| "shallow depth of field" | Фон размыт. |
| "wide angle lens" | Усиленная перспектива, искажение по краям. |
| "anamorphic" | Ощущение кино 2.39:1, линзовые блики. |
| "over-the-shoulder" | Конкретный кадр — модель распознаёт. |
| "Dutch angle" | Наклонённый горизонт. Для напряжения. |
| "low angle" / "high angle" | Позиция камеры относительно субъекта. |
| "worm's eye view" / "bird's eye view" | Крайние ракурсы. |

**Сочетайте камерный язык промпта с вашей camera-LoRA** — они должны соглашаться. `dolly-in` LoRA + «the camera slowly pushes in toward her face» лучше, чем каждое по отдельности.

---

## 6.5 Композиция — поместите субъект куда-то конкретно

Не пишите «a character in a field». Пишите, где в кадре он находится.

| Фраза | Эффект |
|-------|--------|
| "centered in frame" | Субъект заякорен по центру |
| "in the left third" / "in the right third" | Композиция по правилу третей |
| "framed against a clear horizon" | Негативное пространство сверху |
| "surrounded by foreground leaves out of focus" | Естественное кадрирование |
| "filling the frame" | Ощущение тесного CU |
| "tiny figure in the lower third, vast landscape above" | Акцент на масштабе |
| "silhouetted against the window" | Контровой свет + рамка-в-рамке |
| "foreground detail, mid-ground subject, deep background" | Явное построение глубины |

У AI сильные композиционные преподсылки (правило третей, центрированный портрет). Отклонения от этих априори требуют конкретного языка.

---

## 6.6 Свет — самый большой визуальный рычаг после стиля

Расплывчато: "dramatic lighting".
Конкретно: "single warm key light from camera-right at shoulder height; cool blue fill from the window behind; rim light catches the edge of her hair".

Световая лексика:

| Термин | Применение |
|--------|------------|
| "key light" | Доминирующий свет на субъекте |
| "fill light" | Мягкий второстепенный свет, уменьшающий контраст теней |
| "rim light" / "backlight" | Свет сзади, очерчивающий силуэт |
| "practical" | Источник света внутри сцены (лампа, окно, огонь) |
| "motivated" | Свет, направление которого совпадает с видимым источником |
| "hard light" / "soft light" | Качество края тени |
| "chiaroscuro" | Сильный контраст свет/тень |
| "flat lighting" | Ровный, без драмы |
| "three-point lighting" | Классическая портретная схема |
| "low-key" / "high-key" | Общая экспозиция — тёмно-мрачно vs ярко-чисто |
| "golden hour" | Тёплое, низкоугольное солнце |
| "blue hour" | Холодные послезакатные сумерки |
| "neon practical" | Цветной свет от неоновой вывески в кадре |

**Световые фразы для анимации**: "cel-shaded lighting with sharp shadow edges", "soft anime glow on the face", "saturated comic-book highlights", "painted background with watercolor wash light".

---

## 6.7 Атмосфера — дешёвый трюк, поднимающий любой кадр

Погода, частицы, туман, атмосферная дымка — это переводит промпты из категории «AI-лук» в «кинематографично». Добавляйте хотя бы одно в каждый кадр, где это уместно.

| Атмосферный элемент | Эффект |
|---------------------|--------|
| "light fog drifts across the mid-ground" | Добавляет глубину, смягчает фон |
| "dust motes float in the shafts of light" | Подразумевает тишину и тепло |
| "light rain streaks the frame" | Движение + настроение + смягчение фона |
| "snow drifts slowly down" | Спокойно, зимне |
| "heat shimmer distorts the distance" | Жаркое окружение |
| "gentle wind ripples the grass" | Живое окружение |
| "smoke curls upward in the foreground" | Драматично, тревожно |
| "steam rises from the pavement" | Мокрый урбан |
| "bokeh of city lights behind" | Сновидческое, фотографичное |

**Правило**: в каждом кадре с окружением должен быть хотя бы один атмосферный элемент. Без него кадр выглядит чистым, но стерильным.

---

## 6.8 Описание цвета — именованные оттенки бьют hex-коды

Модель не знает `#B22222`. Она знает "cherry red", "brick red", "crimson", "oxblood" — каждый со своими ассоциациями. Используйте именованные цвета.

| Общее | Конкретно |
|-------|-----------|
| "blue" | "navy blue", "cobalt", "electric blue", "periwinkle", "teal", "sky blue", "denim" |
| "red" | "cherry red", "crimson", "scarlet", "brick", "oxblood", "coral", "rose" |
| "green" | "emerald", "forest green", "sage", "chartreuse", "jade", "olive" |
| "yellow" | "golden", "mustard", "cream", "lemon", "amber" |
| "orange" | "terra cotta", "burnt orange", "tangerine", "peach" |

**Фраза про палитру**: "muted earth-tone palette dominated by olive and amber with occasional brick red accents" бесконечно лучше, чем "green and brown and red".

---

## 6.9 Поза, выражение, жест персонажа

Самые трудные промпты. Работайте хирургически.

**Лексика поз:**
- "standing with weight on the right leg, left hand on hip"
- "crouched low, right knee on the ground, left arm extended"
- "mid-stride, leading with the right foot"
- "leaning forward against a counter, elbows resting"

**Лексика выражений** (избегайте базовых эмоциональных ярлыков, когда можно):
- Не "happy" → "the corners of her mouth lift slightly; eyes crinkle at the edges"
- Не "angry" → "his jaw is set; a small muscle flickers at his temple"
- Не "sad" → "she looks slightly off-camera, her eyes not focused on anything"
- Не "surprised" → "his eyebrows lift sharply; his mouth forms a small 'o'"

Описательная версия активирует моторные априори, которых ярлык эмоции не активирует.

**Жесты**:
- "her hand drifts up to touch her throat"
- "he rolls his shoulders back"
- "she exhales slowly through her nose"

Мелкие жесты добавляют жизни. Один жест на кадр обычно — правильно; два конфликтуют.

---

## 6.10 Промптинг звука — слоистое письмо

Из модуля 2: LTX-2.3 использует тот же текст для аудио. Чтобы получить полный саундскейп, а не генерический эмбиент, явно расслаивайте:

**Плохо (один слой)**: "ambient city sound".

**Хорошо (три слоя)**:
```
Rain hisses on the rooftop and distant traffic hums far below (ambient bed).
A single taxi horn sounds once from three blocks away (foreground element).
A slow synth pad rises beneath the visuals (musical bed).
```

**Шаблон звуковых слоёв**:

| Слой | Что описать |
|------|-------------|
| Ambient bed | Тихий фон пространства (ветер, гул трафика, room tone) |
| Foreground sound | Конкретная подсказка, привязанная к действию (шаг, удар, манипуляция с предметом) |
| Atmospheric | Погода, вода, среда (дождь, гром, волны) |
| Musical | Жанр + настроение + инструмент ("faint synth pad", "low orchestral drone") |
| Vocalization | Дыхание, смех, вздох (не диалог — LTX не делает лип-синк) |

Не каждому кадру нужны все пять. Выберите 2–3, которые служат моменту.

---

## 6.11 Негативные промпты — конвенция Lightricks

§9.4 workflow-гайда это установил: короткие, концептуальные негативы. **Не** вставляйте SD-шный длинный список.

Стартовый переиспользуемый негатив:
```
pc game, console game, video game, cartoon, childish, ugly
```

Добавляйте в этот список **только** при конкретных проблемах:

| Что видите в выводе | Добавьте в негатив |
|---------------------|--------------------|
| Слишком аниме, когда хотите американскую анимацию | `anime, manga, japanese animation` |
| Слишком 3D-рендер | `3D rendered, CGI, pixar, video game` |
| Выглядит дёшево / плоско | `childish, cheap, amateurish` |
| Слишком современно для ретро-стиля | `modern, HDR, digital sharpness` |
| Слишком реалистично для стилизации | `photorealistic, photography, live action` |

**Не** добавляйте:
- "blurry, low quality, jpeg artifacts" — обучение Lightricks не взвешивало их; это конвенции SD.
- "extra fingers, bad hands, deformed" — на видео-моделях это не работает так, как на картиночных.
- Больше ~8 концептов всего — после этого убывающая отдача.

---

## 6.12 Эмфаза без весов

Раз `(word:1.4)` в Gemma не работает, как делать акценты?

**1. Повторение через абзацы.** Упомяните ключевой визуальный элемент в двух разных абзацах. Если он критичен — в трёх.

> "She wears a cherry-red costume." ... (абзацем позже) "The cherry-red fabric catches the rim light."

**2. Конкретность.** Специфичное существительное бьёт акцентированное общее.

> Слабо: `(red costume:1.4)`
> Сильно: "a fitted cherry-red costume with black spider-web pattern stitched across the chest"

**3. Front-loading.** Упомянутое раньше несёт больше веса. Критичные элементы — в ¶2 (subject), не в ¶5.

**4. Плотность прилагательных.** Несколько прилагательных на одно существительное усиливают: "a tall, gaunt, deeply shadowed figure" подчёркивает фигуру сильнее, чем "a tall figure".

---

## 6.13 Протокол итерации — меняйте одно

Когда промпт не даёт того, что вы хотите, крупнейшая ошибка — менять пять вещей сразу. Вы теряете сигнал.

**Протокол**:

1. **Залочьте сид.** Никаких исключений, пока итерируете.
2. **Определите ОДНУ неправильную вещь.** Не вайб — конкретный визуальный элемент.
3. **Отредактируйте ОДИН абзац.** Обычно тот, что «владеет» этой вещью (движение → ¶3, свет → ¶4 и т. д.).
4. **Прогоните на низком разрешении.** Не генерируйте на полном разрешении, пока итерируете.
5. **Сравните с предыдущей версией.** Изменилась ли та самая одна вещь? Всё остальное осталось на месте?
6. **Оставьте или откатите.** Если правка помогла — идём дальше. Если ухудшила — откат и другой правкой.

Типичное время цикла на 4090 при 512×384 с 49 кадрами: ~25 секунд. За вечер можно сделать 30 итераций.

---

## 6.14 Переиспользуемые фрагменты промптов — собирайте библиотеку

Заведите в проекте `prompt_fragments.md`. По мере того как вы находите рабочие формулировки, сохраняйте их по категориям:

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

После первого проекта у вас будет 40–80 переиспользуемых фрагментов. В следующем проекте вы уже больше собираете из них, чем пишете. Так пайплайны становятся мышечной памятью.

---

## 6.15 Особенности Gemma, которые стоит знать

**Gemma умнее CLIP, но и более буквальна.**

- "A woman" → генерирует женщину. "A young woman in her late twenties with observable life experience" → генерирует конкретную женщину. Конкретика вознаграждается.
- Gemma понимает **отрицание внутри позитивного промпта** ("no camera movement", "no dialogue", "without any text on screen"). CLIP часто инвертирует; Gemma уважает.
- Gemma понимает **последовательности** ("first she stands still, then she turns toward the camera"). Используйте для кадров с двумя битами.
- Gemma понимает **счёт** ("three figures in the background"). Используйте цифры, не слова («3» парсится надёжнее, чем «three» в некоторых промптах — протестируйте под свой случай).
- Gemma понимает **сравнительные степени** ("taller than the figure behind her", "quieter than the previous shot's audio"). Полезно для относительных размеров.
- Gemma понимает **причинность** ("because of the heavy rain, the streets are empty"). Добавляет мотивацию, которую модель может выразить.

**Известные слабости:**
- Текст в правильном порядке чтения: рендер конкретного текста на экране не работает. Добавляйте текст в посте.
- Точный счёт выше ~5. "Seven candles" часто рендерится как 4 или 8.
- Руки, совершающие конкретные пальцевые действия в CU.
- Брендовые предметы (конкретные логотипы, конкретные модели машин) — сильный дрейф.

---

## 6.16 Частые ловушки промптинга

| Ловушка | Фикс |
|---------|------|
| Промпт слишком короткий (<3 предложений) | Разверните в пятиабзацную структуру |
| Промпт — каша тегов (только запятые) | Перепишите прозой |
| Использование эмфазы `(word:1.4)` | Уберите; используйте повторение и конкретику |
| Style preamble меняется от кадра к кадру | Копируйте ровно одну и ту же преамбулу в каждый кадр |
| Описание статики | Каждое предложение в ¶3 нуждается в глаголе движения |
| Расплывчатый свет ("dramatic lighting") | Указывайте направление, цвет, качество |
| Нет атмосферного элемента | Добавьте туман/дождь/пыль в каждый кадр с окружением |
| Смешение SD и LTX конвенций | Выберите один стиль — для LTX это проза |
| Сверхдетализированный персонаж (все детали костюма в CU) | В CU выкидывайте невидимые предметы. Экономьте токены на том, что в кадре. |
| Размытый звуковой слой ("ambient sound") | Явно расслаивайте тремя звуковыми элементами |
| Негатив — 40 SD-тегов | Держите ≤8 концептуальных токенов |
| Одновременная смена промпта и сида при итерации | Сначала залочьте сид; изолируйте изменения |

---

## 6.17 До / после — один кадр, три черновика

Возьмите это как рабочий пример того, как промпт созревает.

### Черновик 1 — новичок (каша из тегов)

```
anime, cyberpunk, night, rain, woman, red jacket, rooftop, looking down, neon,
cinematic, 4k, detailed
```

Результат: генерическое AI-аниме, статично, плавучесть. Персонаж выглядит никак. Дождь декоративный.

### Черновик 2 — развёрнутая проза

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

Результат: гораздо лучше. Стиль ясен, движение присутствует, атмосфера работает.

### Черновик 3 — полированный

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

Результат: конкретно, кинематографично, консистентно. Этот промпт будет надёжно воспроизводиться на разных сидах.

---

## 6.18 Чек-лист линтинга промпта

Перед тем как ставить кадр в очередь, прогоните мысленную проверку:

- [ ] Пять абзацев, пустые строки между ними
- [ ] ¶1 скопирован из вашего style bible (без изменений)
- [ ] ¶2 явно называет тип кадра и кадрирование
- [ ] В каждом предложении ¶3 есть глагол движения
- [ ] Поведение камеры описано в ¶3 (соответствует вашей camera-LoRA, если есть)
- [ ] Свет в ¶4 имеет направление и цвет/качество
- [ ] В ¶4 есть хотя бы один атмосферный элемент
- [ ] ¶5 наслаивает минимум два звуковых элемента
- [ ] Нет весов `(word:1.4)`
- [ ] Негатив — ≤8 концептуальных токенов
- [ ] Блок описания персонажа (если применимо) идентичен тому, что в других кадрах с этим персонажем

Если все чекбоксы отмечены — промпт готов.

---

## 6.19 Упражнение

1. Выберите один кадр из своего shot list, который ещё не запромптен.
2. Напишите **Черновик 1** как кашу тегов (≤15 слов).
3. Сгенерируйте на низком разрешении с залоченным сидом. Сохраните видео.
4. Перепишите в **Черновик 2** по пятиабзацной структуре (~8–12 предложений).
5. Сгенерируйте с тем же залоченным сидом. Сохраните и сравните.
6. Перепишите в **Черновик 3** — добавьте кинематографию, детали света, атмосферу, слоистый звук.
7. Сгенерируйте с тем же залоченным сидом. Сравните все три.
8. Отметьте, какие элементы промпта дали наибольшую разницу. Добавьте эти паттерны в свой `prompt_fragments.md`.

Вывод этого упражнения будете использовать как калибровочный эталон для каждого будущего проекта.
