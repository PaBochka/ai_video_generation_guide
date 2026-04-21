# 10 — Стайл-гайды: SpiderMan 90-х и аниме

Этот модуль практический: шаблоны промптов, LoRA-стеки, словари камеры
и списки шотов для каждой дорожки. Используйте как справочник на
протяжении финального проекта.

---

## Дорожка A — Анимация SpiderMan 90-х

Точки референса: *Spider-Man: The Animated Series* (1994–1998),
*X-Men: The Animated Series* (1992–1997), splash-страницы комиксов
эпохи Todd McFarlane / Mark Bagley.

### Визуальный язык

- **Линии**: толстые чёрные контуры, переменный вес, тяжелее на
  теневой стороне. Проведены чернилами, не карандашом.
- **Шейдинг**: cel-shaded, максимум 2–3 тона. Без градиентов.
  Жёсткие формы теней.
- **Цвет**: насыщенные primaries, особенно красные/синие герои-
  костюмы на серо-синих городских фонах.
- **Свет**: сильные rim-light'ы на героях, выбеленные источники
  солнца/луны, драматическая подсветка снизу на злодеев.
- **Фоны**: писаный вид, более мягкая детализация, чем у
  персонажей. Здания слегка карикатурны — наклонённые, вытянутые,
  угловатые.
- **Движение**: ограниченная анимация — ключевые позы удерживаются,
  in-between'ов мало. Линии скорости, motion-blur-штрихи,
  звёзды удара.

### Словарь камеры

- Crash zoom на сжатый кулак / лицо крупным планом.
- Whip-pan между соперниками.
- Dutch-angle-кадры экшна.
- Кадры «паука» сверху, смотрящие прямо вниз.
- Параллакс крыш: камера быстро трекает, skyline прокручивается.
- Low-angle-стойка героя против неба.

### Скеффолд промпта

```
[TRIGGER: 90s-saturday-morning-animation],
[TRIGGER: spider-man-1994-style],
[SUBJECT + POSE: например, spider-man crouched on a gargoyle, one hand gripping stone, other hand raised mid-web-shot],
[CAMERA: low angle, slight dutch tilt, crash zoom on the mask],
[STYLE: thick black ink outlines, cel-shaded with two-tone shadow,
saturated red and blue, limited animation key-pose,
comic book splash-page composition],
[LIGHTING: moonlit new york skyline behind, warm street-light rim on costume],
[ATMOSPHERE: distant car horns implied, cold night, light fog]
```

Негатив:

```
photorealistic, 3d render, smooth shading, gradient, blurry lines,
modern cgi, cinematic, photo, watermark, text, bad hands
```

### Добавки под движения камеры

- Crash zoom: `"crash zoom forward into extreme close-up, speed lines
  converging, motion blur on frame edges"`
- Whip pan: `"camera whip pans right, intense horizontal motion
  streaks, subject enters frame mid-pan"`
- Rooftop chase: `"camera tracks alongside at high speed, buildings
  stream past in the background, parallax layers"`
- Impact: `"full-frame impact flash, white and yellow, radial speed
  lines, comic onomatopoeia shape"`

### Пейсинг (24 fps, с ощущением 12 fps)

- Удержание ключевых поз 4–8 кадров.
- Удар в кулак in-between'ится за 3–5 кадров.
- Статичный «smear»-кадр на экстремальном движении — один
  вытянутый кадр.
- Удержания реакции 8–16 кадров для пробивки.

### Идеи шотов для 15-секундной SpiderMan-короткометражки

```
00.0 - 02.0   establishing NY skyline, camera slow dolly up, moon visible
02.0 - 04.0   crouch pose on gargoyle, crash zoom into mask, eyes glint
04.0 - 06.5   leap off, whip pan follows, trailing webs
06.5 - 09.0   swinging through buildings, alternating parallax, cuts on each web
09.0 - 11.0   villain reveal, dutch-angle low shot, menacing silhouette
11.0 - 13.0   impact punch, full-frame hit flash, onomatopoeia
13.0 - 15.0   hero landing stance, low angle, logo-friendly composition
```

### LoRA-стек (T2I ключевые стилы; база SDXL или Flux)

```
LoRA                              strength
90s-saturday-morning-animation    0.85 / 0.80
comic-book-inking                 0.45 / 0.35
(опц.) spider-man-character       0.60 / 0.50
```

Если LoRA «spider-man-character» нет, описывайте костюм в промпте
явно (red and blue spandex with black webbing pattern, white eye
lenses on red mask) и опирайтесь на LoRA инкинга.

---

## Дорожка B — Аниме

Суб-стили, на которые можно целиться:

- **Madhouse/Gainax 90-х** — плотная линия, писаные фоны, cel-
  шейдинг, часто серьёзный регистр.
- **Современные KyoAni / Shaft** — более мягкий шейдинг, пастельная
  палитра, выразительные лица, выверенный композитинг.
- **Ghibli-esque** — акварельные фоны, землистая палитра, мягкое
  движение, природная деталь.

Выберите один под курс. Блендинг суб-стилей требует аккуратного
баланса LoRA и не является ходом для новичка.

### Визуальный язык (пример: Madhouse 90-х)

- **Линии**: средний вес, чёрные, слегка вольные.
- **Шейдинг**: two-tone cel-shading, местами hatch-тень.
- **Цвет**: приглушённый, но насыщенный, продавленные пурпур и
  циан, плёночное зерно.
- **Фоны**: hand-painted, мягче переднего плана, атмосферная
  перспектива.
- **Лица**: крупные выразительные глаза с множеством catch-light'ов,
  маленькие носы, выраженный подбородок.
- **Движение**: ограниченная анимация с сильными ключевыми позами.
  Иконический «anime running» (руки назад, голова вперёд). Волосы
  и ткань пере-анимированы на драматических битах.

### Словарь камеры (аниме)

- Медленный push-in на эмоциональных битах.
- Панорама по статичному писаному фону (трюк Hanna-Barbera, широко
  принятый в аниме).
- Замена фона speed-line'ами во время экшна (персонаж на переднем
  плане, фон — радиальные/горизонтальные линии).
- Pillar-boxing для драматических крупных планов.
- Rack focus между передним и задним планами.

### Скеффолд промпта

```
[TRIGGER: madhouse-90s-anime],
[CHARACTER DESCRIPTION: например, a young woman with shoulder-length
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

Негатив:

```
photorealistic, 3d render, cgi, smooth shading, photo, blurry,
watermark, text, modern digital anime, moe, chibi
```

(Корректируйте негатив в зависимости от того, хотите ли вы moe/chibi.)

### Пейсинг (аниме)

- On 2s (эквивалент 12 fps) для большей части анимации персонажей —
  удерживайте каждую нарисованную позу 2 кадра на 24 fps.
- On 3s (8 fps) для действительно статичных секций с ограниченной
  анимацией.
- On 1s (24 fps) для hero-анимации: большие драки, трансформации,
  фирменный экшн.
- Эмоциональные удержания: 24–48 кадров почти-статичного
  выразительного лица.

### Идеи шотов для 15-секундной аниме-короткометражки

```
00.0 - 03.0   пан по писаному пейзажу, без персонажей, задать настроение
03.0 - 05.0   персонаж входит в кадр, medium-шот, останавливается, смотрит в камеру
05.0 - 06.5   крупный план на глаза, лёгкое движение ветра, эмоциональный бит
06.5 - 09.0   action-триггер: она вытаскивает клинок за три cel-бита
09.0 - 11.5   action wide: motion-линии, дуга клинка, враг за кадром
11.5 - 13.0   impact still: белая вспышка, силуэт, удержан 1 с
13.0 - 15.0   возврат в medium, клинок убран, камера медленно отъезжает
```

### LoRA-стек (T2I ключевые стилы; база SDXL или Flux)

```
LoRA                              strength
madhouse-90s-anime                0.80 / 0.70
cel-shading-painterly             0.30 / 0.25
(опц.) character-lora             0.70 / 0.60
```

### Заметки специфично для аниме-воркфлоу

- **Ощущение «on 2s» — это выбор пейсинга, а не настройка модели.**
  Генерируйте на 24 fps и либо выкидывайте каждый второй кадр в
  редакторе, либо генерируйте на 12 fps нативно (если LTX-2.3
  поддерживает) и дайте «рубленности» дышать.
- **Сопротивляйтесь интерполяции.** RIFE до 60 fps превращает
  аниме в «мыльную оперу». Держите финал 24 fps для 90s-работ, 24
  или 30 для современного аниме.

---

## Дорожка C — Микс-медиа в духе референс-видео

Референс, который вы ссылались, — это микс лайв-экшн + анимационный
оверлей. Без пересмотра по шотам, типичный рецепт такой:

### Подход к продакшну

1. **Снимите или достаньте лайв-экшн-плейт.** Телефонный футаж норм.
   Нужно: субъект на относительно чистом фоне, какое-то движение.
2. **Грейдинг и кроп плейта.** Грейдите под консистентный лук —
   часто зажатые блэки, высокий контраст, лёгкая десатурация.
3. **Ротоскопьте субъект** — руками или моделью сегментации (SAM /
   RVM), чтобы получить покадровую маску.
4. **Сгенерируйте анимационную версию** какого-то участка через
   LTX-2.3 — либо из самого плейта (vid2vid с edge-
   кондиционированием), либо независимо со стилов (I2V).
5. **Композьте**: лайв-экшн-плейт + анимационный оверлей,
   блендящийся по маске, с единым грейдом.

### Визуальный язык

- Гибридный лук опирается на контраст между медиумами: зерно
  лайв-экшн vs. чистая анимационная линия, физика лайв-экшн vs.
  анимационное преувеличение.
- Эффект-слои (частицы, speed-линии, scribble-анимация, комиксовая
  ономатопея) объединяют два медиума.
- Единый грейд на обоих слоях обязателен — без него два медиума
  выглядят как плохой стоковый футаж + плохой клипарт.

### Сторона LTX-2.3

- Используйте edge/depth-кондиционирование (модуль 09) из лайв-
  экшн-плейта, чтобы получить совпадающую анимацию.
- Используйте I2V с писаным стилом субъекта для «альтернативных
  анимационных дублей», заменяющих лайв-экшн-плейт целиком.

### Когда микс-медиа работает

- Как музыкальный клип — движение, ведомое битом, cuts, прикрывающие
  несовершенства.
- Short-form (15–60 с) — лук утомляет на длинных длительностях.
- На стилизованных субъектах — перформеры с характерными силуэтами
  композируются лучше, чем обычные «говорящие головы».

### LoRA-стек

- Лайв-экшн-плейт: без LoRA, только грейдинг.
- Анимационный оверлей (T2I-стадия): сильная стилизованная LoRA —
  та, которая нравится из ваших стилевых дорожек. Подумайте о
  «scribble animation», «sharpie rotoscope», «comic inking».

---

## Глубокое погружение в дорожку A — суб-стили «90s-экшн-мультфильма»

Сваливать всю американскую экшн-анимацию 1990-х в одну кучу —
прятать реальные различия. Знайте, за кем именно вы гонитесь.

### A1 — Spider-Man: The Animated Series (1994)

- **Линии:** толстые, чистые, консистентный вес.
- **Тень:** жёсткие cel-формы, 2 тона, цвет тени сдвинут в холодный
  сине-фиолетовый, а не в чёрный.
- **Палитра:** насыщенные primary красные/синие на героях,
  десатурированные городские фоны (серо-синий, umber).
- **Движение:** умеренно ограниченная анимация; геройский экшн
  получает больше рисунков, чем массовка.
- **Иконические шоты:** крауч-позы на горгульях, параллакс web-
  swing'а, crash-zoom'ы на маску.

Библиотека фрагментов промптов:

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

- **Линии:** чуть грубее, чем Spider-Man TAS, больше вариации
  толщины линии, более текстурированный шейдинг.
- **Палитра:** смелее, больше жёлтых/оранжевых акцентов, тяжелее
  использование пурпура для злодеев и мутантской энергии.
- **Движение:** чаще cel-анимированные эффекты (energy-blast'ы,
  psychic-оверлеи), чем «акробат в городе» Spider-Man'а.
- **Иконические шоты:** hero-шоты с «roster line-up», пиротехнические
  крупные планы на силы, драматический zoom-in на загорающиеся
  глаза.

Библиотека фрагментов промптов:

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

Более тихий регистр. Стоит знать, потому что его техники дают
больше всего настроения на одну генерацию.

- **Линии:** средний вес, исключительно уверенные (минимальные).
- **Тень:** тяжёлые чёрные тени; арт-дирекция знаменитая «Dark
  Deco».
- **Палитра:** десатурированная, noir-склоняющаяся; красные и
  жёлтые появляются редко и тогда выстреливают.
- **Движение:** замечательно сдержанное — иконические тихие
  планерения по крышам, долгие удержания на силуэте.
- **Иконические шоты:** силуэты на крышах, гаргулья-насесты (снова),
  лужи под уличными фонарями под дождём, тень, падающая на лицо
  в два бита.

Библиотека фрагментов промптов:

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

- **Линии:** чище и тоньше, чем Batman TAS, смелее, чем Spider-Man
  TAS.
- **Палитра:** ярче, оптимистичнее, героические primary-цвета
  насыщены даже в дневных сценах.
- **Движение:** более флюидное, чем у Batman; полёт и силовые
  подвиги получают щедрую анимацию.
- **Иконические шоты:** парящий герой на фоне облаков, low-angle
  «статуэтные» портреты, пролёты мимо skyline'а.

Библиотека фрагментов промптов:

```text
POSE: "hovering mid-air, arms at sides, cape billowing upward from
       below, serene expression, looking off to horizon"
CAMERA: "tracking alongside flight, metropolis skyline receding
         into distance, clouds streaming past"
LIGHTING: "bright midday sun, minimal shadow, saturated blue sky,
           white highlights on costume"
```

## Подводные камни дорожки A — что пойдёт не так

- **Перефотореализм.** LTX-2.3 и Flux оба по умолчанию дают
  фотореалистичную текстуру кожи; 90s TV-лук хочет плоскую,
  cel-shaded кожу. Сильно опирайтесь на inking- и cel-LoRA;
  усиливайте негативом (`photoreal, 3d render, skin texture pores`).
- **Современный грейд supehero-фильмов.** Вывод дрейфует в teal-
  and-orange cinematic. Якорьте промптом: «saturated primary
  colours, Saturday morning palette, not cinematic».
- **Над-анимация.** Если RIFE-интерполируете вывод до 60 fps, он
  перестанет выглядеть как 90s TV. Держите 24 fps. Рассмотрите
  падение до 12 fps (on 2s) на секциях.
- **Пропорции персонажа.** Пропорции TV-анимации стилизованы
  (головы крупнее, ноги короче реальных). Без character-LoRA
  промпты дрейфуют в реалистичные пропорции. Явно промптите:
  «stylised cartoon proportions, slightly exaggerated musculature,
  large expressive eyes behind mask».

## Дорожка A — ещё пять идей шотов

```text
1. Cold open: пустой городской skyline, медленный pan слева направо, одно
   окно горит, внутри виден силуэт. 3 с. Героя пока нет.

2. Стойка с газетами вертится, заголовки мелькают («BANK ROBBERY»,
   «HERO SPOTTED»), классический комиксовый переход. 2 с, hard-cuts.

3. Монолог злодея крупным планом, свет снизу, вентилятор раздувает
   ткань, глаза сужаются на ключевой фразе. 4 с. Удержание на глазах
   в конце.

4. Web-swing проходит между двумя зданиями, камера зафиксирована на
   статичном окне — герой проносится мимо, отражение в окне
   вспыхивает, исчезает. 1,5 с.

5. End-shot: герой в крауче на крыше, подтягивает маску от уровня
   подбородка, раскрывая гражданскую идентичность на рассвете. 5 с.
   Медленный zoom in на глаза.
```

## Дорожка A — шаблон дизайна персонажа (для T2I ключевого стила)

Используйте, чтобы сгенерировать консистентный референс персонажа.
Заполняйте пропуски.

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

## Глубокое погружение в дорожку B — суб-стили «аниме»

«Аниме» — зонтик над совершенно разными визуальными регистрами.
Выбирайте один. Блендинг — ловушка.

### B1 — Madhouse 90-х (тёмный драматический регистр)

Референс: *Perfect Blue*, *Ninja Scroll*, ранний Satoshi Kon.

- **Линии:** чёткие, уверенные, плотнее, чем у дневного аниме.
- **Шейдинг:** cel-шейдинг с местами писаной деталью; частые
  hatch-тени на лицах.
- **Палитра:** приглушённая, но насыщенная, продавленные cyan и
  magenta, видимое плёночное зерно.
- **Персонажи:** реалистичные пропорции (7–8 голов в росте),
  тонкая геометрия лица, иногда «Kon tells» — плотные крупные
  планы с неотводимым eye-contact'ом.
- **Фоны:** hand-painted, атмосферные, часто с одним выразительным
  цветовым источником (неоновая вывеска, далёкий огонь).

Библиотека фрагментов промптов:

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

### B2 — Ghibli 90-х (натуралистичная пастораль)

- **Линии:** мягкие, текучие, менее жёсткие, чем у Madhouse.
- **Шейдинг:** акварельные смывки, смешанные с cel, много воздуха
  и полупрозрачности.
- **Палитра:** землистые зелёные, небесные синие, тёплые жёлтые,
  натуральные тона дерева/камня.
- **Персонажи:** округлённые черты, крупные выразительные глаза,
  менее стилизованы, чем moe, но больше, чем реализм.
- **Фоны:** писаные с видимой текстурой мазка, landscape-ведущие.

Библиотека фрагментов промптов:

```text
POSE: "young character standing at a hillside fence, hands resting
       on the wood, hair lifting in breeze, looking into a valley"
CAMERA: "medium wide, eye-level, stable, slight backlight"
LIGHTING: "golden hour sun, warm light on grass, long shadows, soft
           rim on hair"
BACKGROUND: "rolling hills, scattered farmhouses with red roofs,
             distant mountains hazy, clouds hand-painted"
```

### B3 — Современные KyoAni / Shaft (contemporary clean)

- **Линии:** тоньше, чем у 90s, очень точные.
- **Шейдинг:** cel с местами мягкими градиентными акцентами,
  цифрово-чистые highlight'ы.
- **Палитра:** пастель, продавленные синие и розовые,
  редкие неон-акценты.
- **Персонажи:** крупнее глаза, выразительные микро-движения,
  детальные волосы.
- **Фоны:** в композите с персонажами клонятся к фотореализму.

Библиотека фрагментов промптов:

```text
POSE: "seated on a window sill, one leg tucked, chin resting on
       knee, looking out at city"
FACE: "large expressive eyes with multiple catch-lights, small nose,
       soft mouth, subtle blush on cheeks"
LIGHTING: "soft sunlight through window, bloom on highlights,
           ambient pastel, slight chromatic aberration at edges"
```

### B4 — Shonen-экшн (регистр Trigger / Studio Bones)

- **Линии:** кинетичные, толще на action-кадрах, тоньше на обычных.
- **Шейдинг:** смелый cel с частыми «impact-frame»-оверами.
- **Палитра:** высокая насыщенность, сильный цветовой контраст,
  lens-flare'ы.
- **Движение:** самое анимированное из аниме-суб-стилей, часто
  on-1s для hero-экшна, дикие smear-кадры.

Библиотека фрагментов промптов:

```text
POSE: "mid-air in full action pose, fist cocked back, body fully
       torqued, energy aura crackling, hair whipping from motion"
CAMERA: "dutch-angle wide, low-angle, speed-line background
         replacement"
EFFECTS: "energy particles trailing from limbs, lens flare from
          background, impact frame overlay"
```

## Подводные камни дорожки B

- **Коллизия стиля с пропорцией.** «Modern clean» style-LoRA +
  90s-character-LoRA = несовпадение пропорций. Используйте
  персонажей с современными пропорциями со «modern clean»-стилем;
  используйте более коренастых — с 90s-стилями. Смешивать можно,
  но вы должны знать, с чем боретесь.
- **Catch-light'ы глаз дрейфуют.** Без character-LoRA дизайн глаз
  варьируется от шота к шоту. Будьте явны: «circular highlight
  upper-left of each eye, smaller highlight lower-right, no other
  reflections».
- **Интерполяция волос в I2V.** Аниме-волосы полу-жёсткие, но
  выразительные. LTX-2.3 склонна перегонять их в физическую
  симуляцию. Промптите: «anime hair physics, stylised not
  realistic, moves in clumps not strands».
- **Фоны уходят в фотореализм.** Аниме-фоны писаные; Flux/LTX-2.3
  дефолтит в фотографию. Промптите: «hand-painted background,
  visible brushwork, not photographic».

## Дорожка B — ещё пять идей шотов

```text
1. Лепестки сакуры падают мимо статичного крупного плана лица
   персонажа, фокус rack'ится с лепестков на глаза, 4 с, медленно.

2. POV из окна поезда, сельская местность проносится мимо, отражение
   персонажа на стекле призраком ложится на пейзаж, 5 с, непрерывно.

3. Вытаскивание меча в три чистых бита: рука на рукояти / клинок
   полу-вытащен с вспышкой света / полностью вытащен над головой,
   поза удержана. Итого 1,2 с, без интерполяции, биты на кадрах 0,
   8, 24.

4. Сцена за обеденным столом, мягкий свет лампы, пан по трём мискам
   еды перед остановкой на задумчивом лице персонажа. 4 с.

5. Rooftop-at-dawn: два персонажа сидят, один говорит лицом вперёд,
   другой слушает в профиль, небо циклится от индиго к розовому
   сзади. 6 с.
```

## Дорожка B — шаблон дизайна персонажа

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

## Глубокое погружение в дорожку C — микс-медиа

Видео-референс, который вы ссылались, принадлежит широкому семейству
микс-медиа-работ: live-action-plus-scribble, rotoscope-over-
photography, scribble-анимационный оверлей на фотоплейтах. Несколько
конкретных суб-стилей внутри этого зонтика.

### C1 — Sharpie-ротоскоп (scribble-over-footage)

Референс: клипы в родословной A-ha «Take On Me», многие инди-
артисты с hand-drawn-over-film.

- **Линии:** одноцветная чернильная поверх футажа, покадровое
  дрожание.
- **Заливка:** в основном контур, иногда вольная заливка.
- **Цвет:** чёрное на белом или один акцентный цвет.
- **Движение:** «кипящее» дрожание от перерисованных линий каждый
  кадр — намеренная часть эстетики.

Пайплайн:

1. Лайв-экшн-плейт, грейд в высокий контраст.
2. Покадровое извлечение краёв (Canny / HED).
3. Стилизация через SDXL img2img с sharpie-rotoscope-LoRA,
   denoise 0,6–0,75 для поощрения вольной перерисовки.
4. Композьте стилизованный выход обратно поверх обработанного
   плейта или используйте как есть.

### C2 — Painted-over-live-action (стиль Linklater)

Референс: *A Scanner Darkly*, *Waking Life* Ричарда Линклейтера.

- **Линии:** painterly, толще, меньше, чем в sharpie.
- **Заливка:** полноцветные писаные заливки, текстура от мазка.
- **Движение:** слегка «плавающее», цвета сдвигаются от кадра к
  кадру.

Пайплайн:

1. Лайв-экшн-плейт.
2. LTX-2.3 vid2vid с painterly-style-LoRA на denoise 0,5.
3. Опциональное временное сглаживание для уменьшения цветового
   дрожания, если слишком суетливо.

### C3 — Scribble-оверлей с лайв-экшн-ядром

Анимация живёт *поверх* лайв-экшн, не заменяя его. Классика для
музыкальных клипов и титров.

- **Плейт:** лайв-экшн, минимально тронутый.
- **Оверлей:** вручную анимированные (или диффузионно-
  сгенерированные) зацикленные элементы — дудлы, эффекты, текст,
  ротоскоп-акценты вокруг субъекта.
- **Композит:** оверлей поверх плейта с screen/add-бленд-режимами
  или alpha-масками.

Пайплайн:

1. Снимите плейт.
2. Отдельно сгенерируйте элементы оверлея — либо рисуя вручную,
   либо диффузией на чёрном фоне.
3. Композьте в Resolve Fusion или After Effects, с motion-
   трекингом, если оверлеи должны следовать за субъектами.

## Подводные камни дорожки C

- **Перегрузка мерцанием.** Покадровая стилизация по природе более
  «кипящая», чем ожидает современная диффузия. Немного мерцания
  — это эстетика; много — тошнота. Контролируйте нодами временного
  сглаживания или генерируя каждый *второй* кадр и интерполируя.
- **Несоответствие грейда.** Плейт и оверлей грейдятся по-разному,
  из-за чего композит выглядит как два слоя, склеенных кое-как.
  Применяйте объединяющий грейд *после* композита.
- **Лаг оверлея.** Вручную анимированные оверлеи, не отслеживающие
  субъект, читаются как любительщина. Либо коммититесь к
  трекнутым оверлеям (больше работы), либо к независимо-парящим
  оверлеям (другая эстетика, тоже легитимная).
- **Ограничения референс-футажа.** Дёрганый, слабосветный
  телефонный видос даст дёрганый, слабосветный стилизованный
  результат. Каждый раз фиксите вход первым.

## Дорожка C — пять идей шотов

```text
1. Лайв-экшн-субъект идёт, sharpie-ротоскоп покадрово накладывается
   на него, а фон остаётся фотографическим. 4 с.

2. На весь экран — painted-over сцена с фотографии, медленный
   push-in, движение ведёт камера (субъект статичен). 3 с.

3. Лайв-экшн-крупный план лица; анимированные thought-bubble-
   дудлы орбитят вокруг головы, субъект остаётся фотореалом. 5 с.

4. Быстрый cross-fade между лайв-экшн и писаной версией того же
   кадра каждые 8 кадров, ритмично. 2 с.

5. Фотографический wide-шот, анимированный нарисованный субъект
   входит в кадр справа, останавливается в центре, фотография
   «зумит» кропом. 3 с.
```

## Межтрековая справочная карта камеры-шота

Сохраните эту карту. Переиспользуйте свободно.

| Эффект | Фраза промпта | Заметки |
|---|---|---|
| Crash zoom | `crash zoom into [subject], speed lines converging` | Бьёт сильно; экономно |
| Whip pan | `camera whip pans [left/right], motion streaks` | Хорошо прикрывает cut |
| Dolly in | `slow dolly forward, subject centred, lens slightly compresses` | Строит напряжение |
| Dolly out | `slow pull back, subject becomes smaller against [environment]` | Эмоциональная дистанция |
| Overhead | `top-down birds-eye shot, subject below` | Хорошо дезориентирует |
| Dutch angle | `dutch angle tilted right, horizon slanted` | Action / неуют |
| Rack focus | `rack focus from foreground to background, shallow depth of field` | Трудно для LTX-2.3 без сильного промпта |
| Static / locked | `static locked-off camera, no movement` | Даёт движению субъекта дышать |

Продолжайте с [11-final-project.md](11-final-project.md).
