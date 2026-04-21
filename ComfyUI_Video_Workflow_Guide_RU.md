# Построение Text/Image-to-Video воркфлоу в ComfyUI
### Исчерпывающее, модель-агностичное руководство

---

## 1. Понимание пайплайна

Каждый text-to-video или image-to-video воркфлоу в ComfyUI следует одному и тому же высокоуровневому пайплайну, независимо от конкретной модели (Wan, HunyuanVideo, CogVideoX, AnimateDiff, LTX-Video, Mochi и др.). Базовая идея отражает генерацию изображений, но добавляет **временное (time/frame) измерение**.

**Универсальный пайплайн выглядит так:**

```
Load Model → Encode Conditioning → Configure Latent Space → Sample (Denoise) → Decode → Export
```

Каждая из этих стадий отображается на одну или несколько групп нод в ComfyUI.

---

## 2. Базовые компоненты воркфлоу

### 2.1 Ноды загрузки модели

Эти ноды загружают предобученные веса в память. Видео-модели обычно большие (10–30+ ГБ), поэтому управление VRAM здесь критично.

**Распространённые типы нод:**

- **Checkpoint Loader** — Загружает один объединённый файл (`.safetensors` / `.ckpt`), содержащий UNet/DiT, VAE и текстовый энкодер(ы). Некоторые видео-модели поставляются как один чекпоинт.
- **Diffusion Model Loader (UNet/DiT Loader)** — Загружает только denoising-бэкбон (UNet или DiT-трансформер). Используется, когда компоненты модели разнесены по разным файлам.
- **VAE Loader** — Загружает Variational Autoencoder отдельно. Видео-VAE 3D-aware (они кодируют/декодируют и по временной оси).
- **CLIP / Text Encoder Loader** — Загружает текстовый энкодер(ы). Многие современные видео-модели используют двойные энкодеры (например, CLIP + T5-XXL, или CLIP-L + CLIP-G).

**Ключевые соображения:**

- **Точность / dtype**: Большинство видео-моделей поддерживают `fp16`, `bf16` или `fp8`. Меньшая точность снижает VRAM при незначительной потере качества. У нод часто есть параметр `dtype` или `weight_dtype`.
- **Device offloading**: Некоторые ноды-лоадеры поддерживают CPU offloading или последовательную загрузку, чтобы вместить большие модели на потребительских GPU (16–24 ГБ VRAM).
- **Квантизация**: Некоторые воркфлоу используют GGUF-квантизованные модели (Q4, Q8 и т.д.) для драматического снижения использования VRAM. Они требуют специальных нод-лоадеров (например, GGUF-based loaders).

---

### 2.2 Текстовое кондиционирование (Промптинг)

Текстовое кондиционирование переводит ваш текстовый промпт в эмбеддинги, которые направляют процесс denoising'а.

**Типичные ноды:**

- **CLIP Text Encode** — Стандартная нода. Принимает текстовую строку и CLIP-модель, выдаёт `CONDITIONING`-объект. Для видео обычно нужны и позитивный (что вы хотите), и негативный (чего избегать) кондиционинги.
- **Dual CLIP Encode** — Для моделей, использующих два текстовых энкодера (часто в современных архитектурах). Кодирует через оба и объединяет выходы.
- **T5 Text Encode** — Некоторые видео-модели используют T5-XXL вместо или вместе с CLIP. Отдельная нода для T5-кодирования.

**Советы по промптингу для видео:**

- Описывайте **движение и временные изменения**, а не только статичную сцену. Например: "A woman walks through a forest, camera slowly panning left, leaves falling."
- Многие видео-модели хорошо реагируют на **кинематографический/операторский язык**: "dolly shot", "tracking shot", "slow motion", "timelapse".
- Негативные промпты часто включают: "blurry, distorted, static, low quality, watermark, text, morphing, flickering."
- Некоторые модели поддерживают **per-frame или временной промптинг** через специальные ноды кондиционирования, позволяющие назначать разные промпты разным временным регионам.

---

### 2.3 Кондиционирование изображением (Image-to-Video)

Для image-to-video воркфлоу вы предоставляете одно или несколько референсных изображений, которые модель использует как стартовый кадр(ы) или стилевой референс.

**Типичные ноды и подходы:**

- **Load Image** — Загружает входное изображение с диска.
- **Image Encode (VAE Encode)** — Кодирует изображение в латентное пространство с помощью видео-VAE. Этот латент затем инжектируется в процесс denoising'а, обычно как первый кадр.
- **Image-to-Video Conditioning Node** — Модель-специфичные ноды, объединяющие закодированное изображение с текстовым кондиционированием в единый сигнал управления. Модель "продолжает" видео от входного изображения.
- **CLIP Vision Encode** — Некоторые модели используют CLIP Vision энкодер для извлечения семантических признаков из референсного изображения (стиль/контент-управление вместо точного попиксельного совпадения первого кадра).

**Распространённые паттерны:**

- **First-frame conditioning**: Входное изображение становится кадром 0; модель генерирует последующие кадры.
- **Last-frame conditioning**: Некоторые модели поддерживают указание конечного кадра, генерируя кадры, переходящие от начала к концу.
- **Start + End frame**: Несколько моделей умеют интерполировать между двумя референсными изображениями.
- **Style/IP-Adapter conditioning**: Использует image-фичи для стилевого управления без жёсткой привязки к точному первому кадру.

**Подготовка изображения:**

- Перед кодированием отресайзьте входное изображение к ожидаемому моделью разрешению. Распространённые видео-разрешения: 512×512, 576×320, 832×480, 848×480, 1280×720.
- Соотношение сторон важно — большинство моделей обучены на специфичных aspect ratio.

---

### 2.4 Конфигурация латентного пространства

Перед сэмплированием нужно определить "холст" в латентном пространстве — включая количество кадров.

**Ключевые ноды:**

- **Empty Latent Video** — Создаёт пустой латентный тензор с размерностями: `[batch, channels, frames, height, width]`. Здесь вы задаёте разрешение и количество кадров.
- **Empty Latent Image (with batch)** — Некоторые старые сетапы используют стандартный empty latent с batch-размерностью, перепрофилированной под кадры.

**Параметры, которые вы контролируете:**

- **Width & Height** — Пространственное разрешение выхода. Должно соответствовать обучающему разрешению модели или поддерживаемому aspect ratio. Распространённые: 512×512, 832×480, 1280×720.
- **Frame Count / Length** — Количество генерируемых кадров. У моделей есть максимальные поддерживаемые длины (часто 16, 24, 49, 81, 97 или 121 кадр). Превышение обученного максимума обычно даёт артефакты.
- **Batch Size** — Обычно 1 для видео-воркфлоу (временное измерение отдельно от batch).

**Важные замечания:**

- Разрешение и количество кадров напрямую влияют на использование VRAM. Удвоение любого измерения примерно учетверяет память.
- Некоторые модели требуют, чтобы количество кадров следовало специфичным формулам (например, `4k+1` или `8k+1`, где k — целое).

---

### 2.5 Сэмплирование (KSampler)

Сэмплер — это движок воркфлоу: он итеративно шумоочищает латентный тензор под управлением вашего кондиционирования.

**Базовые ноды:**

- **KSampler** — Стандартный сэмплер ComfyUI. Работает с видео-моделями, которые подключаются к стандартному пайплайну ComfyUI.
- **KSampler Advanced** — Даёт больше контроля (start/end steps, инжекция шума и т.д.).
- **Model-Specific Samplers** — Некоторые видео-модели поставляются с кастомными нодами-сэмплерами, оптимизированными под их архитектуру (например, flow-matching сэмплеры для rectified flow моделей).

**Ключевые параметры:**

| Параметр | Что делает | Типичные значения |
|-----------|-------------|----------------|
| **Steps** | Количество шагов denoising'а | 20–50 (больше = выше качество, медленнее) |
| **CFG Scale** | Насколько сильно модель следует промпту | 1.0–15.0 (видео-модели часто используют низкий CFG, например 1.0–7.0) |
| **Sampler** | Алгоритм пошагового denoising'а | `euler`, `euler_ancestral`, `dpmpp_2m`, `uni_pc`, `deis` |
| **Scheduler** | Контролирует расписание шума | `normal`, `karras`, `sgm_uniform`, `beta`, `linear_quadratic` |
| **Seed** | Случайный сид для воспроизводимости | Любое целое; одинаковый сид = одинаковый результат |
| **Denoise** | Сколько шума добавить/убрать | 1.0 для полной генерации; <1.0 для img2vid или рефайна |

**Видео-специфичные замечания по сэмплированию:**

- Многие видео-модели используют **flow matching** вместо традиционной диффузии, что означает свою логику scheduler'а и они могут не использовать CFG в традиционном смысле (некоторые используют "guidance scale", встроенный в модель, вместо classifier-free guidance).
- **Denoise strength** критична для image-to-video: меньшие значения (0.5–0.85) сохраняют больше входного изображения; большие дают модели больше творческой свободы.
- **Временная согласованность** частично контролируется сэмплером — некоторые сэмплеры дают более плавное движение. Euler и DPM++ 2M обычно безопасны по умолчанию.

---

### 2.6 VAE-декодирование

После сэмплирования шумоочищенный латент должен быть декодирован обратно в пиксельное пространство.

**Ноды:**

- **VAE Decode** — Декодирует латентный тензор в последовательность изображений (кадров). Для видео-моделей использует видео-aware VAE, понимающий временное измерение.
- **Tiled VAE Decode** — Декодирует пространственными тайлами для снижения VRAM. Незаменим для высоких разрешений на ограниченном железе. Некоторые ноды также тайлят по временной оси.

**Замечания:**

- Видео-VAE декодирование требовательно к памяти. Если на этой стадии заканчивается VRAM — переключайтесь на тайлинг.
- Выход VAE Decode — батч изображений (по одному на кадр), которые вы затем собираете в видео.

---

### 2.7 Видео-вывод / Экспорт

Финальная стадия: объединение декодированных кадров в воспроизводимый видео-файл.

**Распространённые ноды:**

- **Video Combine** (из ComfyUI-VideoHelperSuite) — Самая популярная нода для экспорта видео. Объединяет кадры в MP4, GIF или WEBM.
- **Save Image Sequence** — Сохраняет отдельные кадры как пронумерованные PNG/JPG файлы для внешнего монтажа.
- **Save Animated WebP / APNG** — Для коротких клипов или web-friendly форматов.

**Параметры:**

- **Frame Rate (FPS)** — Обычно 8, 16, 24 или 30 fps. Соответствуйте обучающему FPS модели для естественного движения.
- **Codec** — H.264 (MP4) самый совместимый. H.265 для лучшего сжатия. VP9 для WEBM.
- **Quality / CRF** — Контролирует сжатие. Меньший CRF = выше качество, больший файл.
- **Loop Count** — Для GIF: сколько раз зациклить (0 = бесконечно).

---

## 3. Опциональные, но распространённые улучшения

### 3.1 ControlNet / Guidance

ControlNet'ы добавляют структурное управление генерацией — карты глубины, детекция краёв, оценка позы и т.д.

- **Apply ControlNet** — Прикрепляет ControlNet-модель к вашему кондиционированию. ControlNet получает guidance-изображение/последовательность и направляет пространственную структуру каждого кадра.
- **Распространённые типы управления**: Depth, Canny edges, OpenPose, Segmentation maps, Line art.
- Для видео можно подавать **последовательность** управляющих изображений (по одному на кадр) для управления движением.

### 3.2 Загрузка LoRA

LoRA (Low-Rank Adaptations) тонко настраивают поведение модели без замены полных весов.

- **Load LoRA** — Загружает LoRA-файл и применяет его к модели и/или CLIP-энкодеру.
- Стек нескольких LoRA даёт комбинированные эффекты (например, style LoRA + motion LoRA).
- Параметр **Strength** контролирует силу влияния LoRA на выход (0.0–1.0+).

### 3.3 Апскейлинг / Super-Resolution

Сгенерированное видео часто в более низком разрешении. Апскейлинг улучшает финальное качество.

- **Latent upscale + второй проход** — Апскейльте латент, затем повторно denoise при частичной силе для добавления деталей.
- **Pixel-space upscaling** — Используйте модели типа RealESRGAN для апскейла декодированных кадров.
- **Покадровый vs. временной апскейлинг** — Покадровый проще, но может вносить мерцание; временно-aware апскейлеры сохраняют согласованность.

### 3.4 Интерполяция кадров

Увеличивает количество кадров для более плавного движения без перегенерации.

- **RIFE / FILM ноды** — Берут существующие кадры и генерируют промежуточные.
- Полезно для перехода с 8 fps выхода модели к 24/30 fps для плавного воспроизведения.

### 3.5 IP-Adapter / Style Transfer

IP-Adapter'ы позволяют image-based стилевое управление без использования изображения как прямого первого кадра.

- Загрузите IP-Adapter модель параллельно с основной моделью.
- Подавайте референсные изображения для управления стилем, композицией или внешностью субъекта.

### 3.6 Оптимизация внимания / памяти

Критично для запуска больших видео-моделей на потребительском железе.

- **Tiled sampling / attention** — Обрабатывает видео пространственными или временными тайлами.
- **SAGE Attention / Flash Attention** — Оптимизированные реализации внимания, снижающие VRAM и увеличивающие скорость.
- **CPU offloading** — Перемещает неиспользуемые компоненты модели в системную RAM.
- **Tea Cache / Temporal Caching** — Кэширует промежуточные вычисления между кадрами для снижения избыточной работы.

---

## 4. Типичная разводка воркфлоу

Вот как ноды обычно соединяются в стандартном text-to-video воркфлоу:

```
┌─────────────────┐     ┌──────────────────┐
│  Load Checkpoint │────▶│  CLIP Text Encode │──── positive conditioning ────┐
│  (or separate    │     │  (Positive)       │                               │
│   loaders)       │     └──────────────────┘                               │
│                  │     ┌──────────────────┐                               │
│                  │────▶│  CLIP Text Encode │──── negative conditioning ────┤
│                  │     │  (Negative)       │                               │
│                  │     └──────────────────┘                               │
│                  │                                                         │
│                  │──── model ──────────────────────────────────────────────┤
│                  │                                                         │
│                  │──── vae ───────────────────────┐                       │
└─────────────────┘                                 │                       │
                                                     │                       │
┌─────────────────┐                                 │     ┌─────────────┐  │
│ Empty Latent     │──── latent ────────────────────────▶│  KSampler   │◀──┘
│ Video            │                                 │     │             │
│ (set resolution  │                                 │     └──────┬──────┘
│  & frame count)  │                                 │            │
└─────────────────┘                                 │         latent
                                                     │            │
                                                     │     ┌──────▼──────┐
                                                     └────▶│  VAE Decode  │
                                                           └──────┬──────┘
                                                                  │
                                                               images
                                                                  │
                                                           ┌──────▼──────┐
                                                           │ Video Combine│
                                                           │ (export MP4) │
                                                           └─────────────┘
```

**Для image-to-video** добавьте это перед сэмплером:

```
┌─────────────┐     ┌──────────────┐     ┌──────────────────────┐
│ Load Image   │────▶│ VAE Encode   │────▶│ Image-to-Video       │
│              │     │              │     │ Conditioning Node    │
└─────────────┘     └──────────────┘     │ (merges with text    │
                                          │  conditioning)       │
                                          └──────────┬───────────┘
                                                     │
                                              conditioning ──▶ KSampler
```

---

## 5. Справка по VRAM и производительности

| Разрешение | Кадры | Прибл. VRAM (fp16) | Заметки |
|-----------|--------|---------------------|-------|
| 512×512 | 16 | 8–12 ГБ | Доступно на большинстве GPU |
| 512×512 | 49 | 12–18 ГБ | Mid-range GPU |
| 832×480 | 81 | 16–24 ГБ | High-end consumer |
| 1280×720 | 97+ | 24–48+ ГБ | Требует offloading или квантизации |

**Советы по снижению VRAM:**

- Используйте `fp8` или GGUF-квантизованные модели.
- Включите тайловый VAE-декодинг.
- Используйте SAGE/Flash attention, если поддерживается.
- Понижайте разрешение и интерполируйте/апскейльте после.
- Уменьшайте количество кадров и используйте интерполяцию (RIFE).
- Включите sequential CPU offloading в нодах-лоадерах.

---

## 6. Необходимые пакеты кастомных нод

Видео-возможности ComfyUI сильно зависят от community-нод. Вот наиболее часто нужные:

| Node Pack | Назначение |
|-----------|---------|
| **ComfyUI-VideoHelperSuite** | Экспорт видео (Video Combine), загрузка кадров, обработка GIF/MP4 |
| **ComfyUI-Manager** | Установка/управление кастомными нодами и моделями из UI |
| **ComfyUI-GGUF** | Загрузка GGUF-квантизованных моделей для меньшей VRAM |
| **ComfyUI-KJNodes** | Утилитарные ноды (батч-операции, помощники кондиционирования) |
| **ComfyUI-Frame-Interpolation** | RIFE/FILM интерполяция кадров |
| **ComfyUI-Advanced-ControlNet** | Поддержка ControlNet с временным батчингом |
| **ComfyUI-Impact-Pack** | Утилитарный пакет (детекция лиц, региональный промптинг и т.д.) |

Модель-специфичные пакеты (например, для Wan, HunyuanVideo, CogVideoX) предоставляют ноды-лоадеры и кондиционирования, заточенные под каждую архитектуру.

---

## 7. Отладка распространённых проблем

| Проблема | Вероятная причина | Решение |
|---------|-------------|-----|
| Чёрный или зашумлённый выход | Неправильный VAE, несовпадающие компоненты модели, слишком высокий CFG | Проверьте, что все компоненты из одного семейства модели; снизьте CFG |
| Статичное / замёрзшее видео | Слишком низкий denoise, в промпте нет движения, неправильное количество кадров | Увеличьте denoise; добавьте слова движения в промпт; проверьте формулу кадров |
| Мерцающие кадры | Нестабильность сэмплера, нет временного внимания | Попробуйте `euler` или `dpmpp_2m`; убедитесь, что у модели загружены временные слои |
| Out of memory (OOM) | Слишком высокое разрешение или количество кадров | Снизьте разрешение/кадры; включите тайлинг; используйте квантизованные модели |
| Color shift / banding | Проблемы точности VAE | Используйте `fp32` для VAE-декодинга; включите тайловый VAE |
| Искажённое движение | Слишком высокий CFG для модели | Многие видео-модели лучше работают при CFG 1.0–6.0; экспериментируйте |
| Первый кадр не совпадает с входом | Слишком высокий denoise в img2vid | Снизьте denoise до 0.6–0.8; проверьте image encoding |
| Размытый выход | Недостаточно шагов, несовпадение разрешения | Увеличьте шаги; соответствуйте нативному разрешению модели |

---

## 8. Общие лучшие практики

1. **Начинайте с малого**: Начните с низкого разрешения и малого количества кадров для теста промпта и настроек, затем масштабируйте.
2. **Соответствуйте спекам модели**: Всегда проверяйте, на каком разрешении, количестве кадров, aspect ratio и FPS обучалась модель. Борьба с обучающими данными даёт плохие результаты.
3. **Фиксация сида**: Держите сид фиксированным при подкручивании параметров, чтобы честно сравнивать изменения.
4. **Сохраняйте воркфлоу**: Используйте встроенный "Save" в ComfyUI для сохранения рабочих конфигураций. Видео-воркфлоу сложные и легко ломаются.
5. **Читайте model card**: У каждой модели свои требования к кондиционированию, scheduling'у и конфигурации. Прочитайте документацию перед построением воркфлоу.
6. **Батч-тестируйте промпты**: Используйте систему очереди ComfyUI для батч-генерации с разными промптами/сидами.
7. **Постобработка снаружи**: Для профессиональных результатов экспортируйте последовательности кадров и используйте специализированные видеоредакторы (DaVinci Resolve, FFmpeg) для финального монтажа, цветокоррекции и звука.

---

## 9. Спотлайт модели: LTX-2 / LTX-2.3

LTX-2 (Lightricks) — это совместный **audio + video** DiT. Поскольку он генерирует обе модальности за один проход denoising'а, латент, входящий в сэмплер, — это *конкатенированный AV-латент*, а не обычный видео-латент. Поэтому общий пайплайн §4 валится на LTX-2 с ошибками shape-mismatch на сэмплере — модель ожидает дополнительное измерение модальности. **LTX-2.3 заменяет LTX-2** (лучше детали, 9:16 портрет, ниже шум аудио, сильнее I2V, обновлённые distilled-веса, формальный двухстадийный base+upscale пайплайн). Все новые воркфлоу должны таргетиться на LTX-2.3.

### 9.1 Необходимый custom node pack

**ComfyUI-LTXVideo** от Lightricks — установите через Manager (поиск "LTXVideo") или:
```
git clone https://github.com/Lightricks/ComfyUI-LTXVideo.git
pip install -r requirements.txt
```
Референсные воркфлоу-JSON'ы поставляются в `custom_nodes/ComfyUI-LTXVideo/example_workflows/2.3/` (Single_Stage, Two_Stage_Distilled, ICLoRA_Motion_Track, FLF2V).

### 9.2 Файлы моделей и их размещение

| Папка | Файл | Заметки |
|--------|------|-------|
| `models/checkpoints/` | `ltx-2.3-22b-dev.safetensors` (46 ГБ BF16) | 48 ГБ+ карты, максимальное качество |
| `models/checkpoints/` | `ltx-2.3-22b-dev-fp8.safetensors` (~29 ГБ) | 24–32 ГБ карты, рекомендуемый по умолчанию |
| `models/checkpoints/` | `ltx-2.3-22b-dev-nvfp4.safetensors` (~21 ГБ) | Blackwell, ~16 ГБ карты |
| `models/checkpoints/` | GGUF Q2_K..Q8_0 (8–25 ГБ) | Загружается через `UnetLoaderGGUF` |
| `models/checkpoints/` | `ltx-2.3-22b-distilled-1.1.safetensors` | Меньше шагов, ниже CFG |
| `models/loras/` | `ltx-2.3-22b-distilled-lora-384.safetensors` (7.6 ГБ) | Применяется на dev при strength 0.6–0.8 → быстрое сэмплирование |
| `models/loras/` | IC-LoRAs (Union depth+edges, motion, pose, Canny, camera dolly/jib, detailer) | Опциональное структурное управление |
| `models/text_encoders/` | `gemma_3_12B_it_fp8_scaled.safetensors` (13 ГБ) | Рекомендуемый вариант Gemma |
| `models/text_encoders/` | `gemma_3_12B_it_fp4_mixed.safetensors` (9.5 ГБ) | Меньше VRAM |
| `models/latent_upscale_models/` | `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` | Для stage-2 latent upscale |
| `models/latent_upscale_models/` | `ltx-2.3-temporal-upscaler-x2-1.0.safetensors` | Опциональный временной апскейл |

**Прямые URL для скачивания (проверены):**

| Файл | URL |
|------|-----|
| `gemma_3_12B_it_fp8_scaled.safetensors` (13.2 ГБ) | `huggingface.co/Comfy-Org/ltx-2/blob/main/split_files/text_encoders/gemma_3_12B_it_fp8_scaled.safetensors` |
| `gemma_3_12B_it_fp4_mixed.safetensors` (9.5 ГБ) | тот же репо, та же папка |
| `ltx-2.3-22b-dev-fp8.safetensors` (29.1 ГБ) | `huggingface.co/Lightricks/LTX-2.3-fp8/blob/main/ltx-2.3-22b-dev-fp8.safetensors` |
| `ltx-2.3-22b-distilled-fp8.safetensors` (29.5 ГБ) | `huggingface.co/Lightricks/LTX-2.3-fp8` (тот же репо) |
| `ltx-2.3-22b-distilled-lora-384-1.1.safetensors` (7.6 ГБ) | `huggingface.co/Lightricks/LTX-2.3` |
| `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` (996 МБ) | `huggingface.co/Lightricks/LTX-2.3` |
| `ltx-2.3-temporal-upscaler-x2-1.0.safetensors` (262 МБ) | `huggingface.co/Lightricks/LTX-2.3` |
| Camera-control LoRAs (LTX-2 19B; грузятся и на 2.3) | `huggingface.co/collections/Lightricks/ltx-2` |

Заметка: FP8 чекпоинты живут в **отдельном** `Lightricks/LTX-2.3-fp8` репо (а не BF16-репо `Lightricks/LTX-2.3`).

### 9.3 Текстовый энкодер: Gemma 3 12B Instruct (НЕ generic CLIP)

Gemma 3 — это 12B decoder LLM со своим токенизатором, проекцией и hidden-state размерностями — **стандартный `CLIPLoader` / `DualCLIPLoader` молча провалится или выдаст shape-ошибки на DiT**. Симптомы включают консольные предупреждения вида `clip missing: ['gemma3_12b.logit_scale', ...]`, абракадабру на выходе или пустое кондиционирование.

**Правильный лоадер:** `LTXAVTextEncoderLoader` (или `DualCLIPLoaderGGUF` с `type="ltxv"` для GGUF). Положите единый консолидированный `.safetensors` прямо в `models/text_encoders/` — **не делайте** `git clone` с HuggingFace; LFS pointer-файлы оставляют tokenizer-ассеты отсутствующими и "тихо ломают производительность" (LTX-2 issue #106).

**Оптимизация VRAM-стоимости Gemma:**
- `LTXVSaveConditioning` пишет закодированное кондиционирование в `models/embeddings/` как `.safetensors`; `LTXVLoadConditioning` загружает его мгновенно. Закодируйте Gemma один раз, переиспользуйте между многими генерациями.
- `GemmaAPITextEncode` оффлоадит кодирование на бесплатное API Lightricks — нулевая локальная VRAM Gemma.

### 9.4 Цепочка нод для text-to-video (канонический two-stage)

```
CheckpointLoaderSimple (ltx-2.3-22b-dev-fp8)
   ├── MODEL ──▶ (опционально) LoraLoader (distilled-lora-384, str 0.5 distilled / 0.2 dev)
   └── VAE   ──▶ to decode stage

LTXAVTextEncoderLoader (gemma_3_12B_it_fp8_scaled)
   └── CLIP ──▶ CLIPTextEncode (positive)
             └▶ CLIPTextEncode (negative)
                  └─▶ LTXVConditioning (frame_rate=25)
                         └── pos / neg conditioning

EmptyLTXVLatentVideo (W, H, length=97, batch=1) ─┐
LTXVEmptyLatentAudio (length ≥ video length)    ─┴─▶ LTXVConcatAVLatent ─▶ AV latent

LTXVScheduler (steps=20, max_shift=2.05, base_shift=0.95, stretch=true, terminal=0.1) ─▶ SIGMAS
KSamplerSelect (euler / euler_ancestral)                                                ─▶ SAMPLER
RandomNoise (seed)                                                                      ─▶ NOISE
MultimodalGuider (model, pos, neg, scale=28, skip_blocks=[...])                         ─▶ GUIDER
   (или CFGGuider с cfg=1.0 при использовании distilled sigmas)

SamplerCustomAdvanced (NOISE, GUIDER, SAMPLER, SIGMAS, AV latent)
   └── output ──▶ LTXVSeparateAVLatent
                     ├── video latent ──▶ LTXVTiledVAEDecode  ──▶ frames
                     └── audio latent ──▶ LTXVAudioVAEDecode  ──▶ AUDIO

frames + AUDIO + fps ──▶ VHS_VideoCombine  (или CreateVideo)
```

**Stage 2 (рекомендуется для полного 2× качества):** возьмите stage-1 video latent → `LatentUpscaleModelLoader` (загружает `ltx-2.3-spatial-upscaler-x2-1.1`) → `LTXVLatentUpsampler` → второй `SamplerCustomAdvanced` (steps=3–4, CFG=1.0, distilled LoRA активна) → `LTXVTiledVAEDecode`. Апскейлер принимает **не до конца дешумленный** латент (поэтому в §9.7 параметр `LTXVScheduler.terminal=0.1` важен — он оставляет запас для второго прохода).

**`CFGGuider` vs `MultimodalGuider`:**
- `CFGGuider` — единый CFG-скаляр, только видео (или distilled, где CFG отключён). Используется в two-stage distilled detail-проходах (обычно `cfg=1`).
- `MultimodalGuider` — нужен при генерации audio+video вместе. Экспозит per-modality scales, **STG (Spatio-Temporal Guidance)** с `skip_block_indices` / `stg_scale` / `rescale` и кросс-модальную синхронизацию. Опубликованные 2.3 single-stage воркфлоу используют `scale=28` на этом гайдере; STG defaults официально не задокументированы (начните со значений из поставленного JSON-шаблона).

Полезные обёртки:
- **`LTXVCropGuides`** — обрезает IC-LoRA / conditioning guides обратно к выходному количеству кадров после апсэмплинга.
- **`LTXVNormalizingSampler`** — оборачивает сэмплер для предотвращения latent overbake при высоком CFG. Используйте только на stage 1; пропускайте для inpaint/extend.

**Примеры промптов (дословно из `LTX-2.3_T2V_I2V_Single_Stage_Distilled_Full.json`):**

> **Positive (T2V + audio):** *"A traditional Japanese tea ceremony takes place in a tatami room as a host carefully prepares matcha. Soft traditional koto music plays in the background, adding to the serene atmosphere. The bamboo whisk taps rhythmically against the ceramic bowl while water simmers in an iron kettle. Guests kneel in formal seiza position, watching in respectful silence. The host bows and presents the tea bowl, turning it precisely before offering it to the first guest with soft-spoken words."*
>
> **Negative:** *"pc game, console game, video game, cartoon, childish, ugly"*

Обратите внимание на конвенцию Lightricks для негативного промпта: короткие, **concept-level** токены (категории медиа + качественные прилагательные), **а не** длинный SD-style список "low quality, blurry, distorted, extra fingers...". Поставленные шаблоны буквально используют только эти шесть токенов.

### 9.5 Цепочка image-to-video

То же, что §9.4, но перед `LTXVConcatAVLatent`:

```
LoadImage ──▶ LTXVPreprocess ──▶ LTXVImgToVideoInplace (strength=1.0)
                                       └── инжектирует изображение как кадр 1 EmptyLTXVLatentVideo
```

**Критично:**
- `LTXVPreprocess` деградирует входное изображение, чтобы соответствовать тренировочному сжатию. **Пропуск даёт почти-статичный / призрачный выход.**
- Ресайзьте вход к **половине** целевого разрешения для stage 1, к полному — для stage 2.
- Промпт должен описывать **движение**, а не сцену (модель уже видит сцену).
- Расширение мультиклипа: `GetImageRangeFromBatch` берёт последние N кадров предыдущего клипа → используйте как connector, обрежьте overlap перед concat.
- FLF2V (first / first-middle-last) воркфлоу живут в `example_workflows/2.3/`.

### 9.6 Ограничения разрешения и кадров

- **Правило 32-stride**: `width % 32 == 0` и `height % 32 == 0`. Полно-resолюционный потолок: 1280×720 до 2560×1440.
- **Формула кадров**: `frames = 8n + 1` → валидные: 9, 17, 25, 33, 41, 49, 57, 65, 73, 81, 97, 105, 121, 129, 161, 257 (макс ~257).
- **FPS**: обучение на 24 / 25 / 30. 50 fps работает, но удваивает стоимость. Держите `LTXVConditioning.frame_rate` и `VideoCombine.fps` идентичными.
- **Длина audio latent должна быть ≥ длины video latent** даже для тихого выхода, иначе сэмплирование упадёт. Для drives audio-track'а `TrimAudioDuration` точно к длине видео перед кодированием.

**Практичные (W, H, frames) пресеты:**

| Пресет | Stage-1 размер | Stage-2 размер (после x2) | Кадры | Длительность @ 25fps | Aspect |
|--------|--------------|-------------------------|--------|------------------|--------|
| Square preview | 640×640 | 1280×1280 | 49 | 2.0s | 1:1 |
| Landscape short | 768×512 | 1536×1024 | 97 | 3.9s | 3:2 |
| Cinematic 16:9 | 832×480 | 1664×960 | 121 | 4.8s | ~16:9 |
| Portrait 9:16 (только 2.3) | 480×832 | 960×1664 | 121 | 4.8s | 9:16 |
| Long single-shot | 768×512 | 1536×1024 | 257 | 10.3s | 3:2 |

Для длиннее ~10s используйте `LTXVLoopingSampler` (§10.5) — VRAM растёт примерно линейно с количеством кадров.

### 9.7 Sampler / scheduler пресеты

| Стадия | Шаги | CFG | Sampler | Заметки |
|-------|-------|-----|---------|-------|
| Dev, stage 1 | 20 (до 25–35) | 4.0 (диапазон 2–5) на `MultimodalGuider` (или `scale=28` по шаблону) | `euler` / `euler_ancestral` | `LTXVScheduler` + `LTXVNormalizingSampler` |
| Dev, stage 2 (upsample) | 3–4 | 1.0 (`CFGGuider`) | `euler` | `LTXVScheduler` |
| Distilled (single stage) | 8 | 1.0 (`CFGGuider`) | `euler` | `ManualSigmas` (см. ниже), без `LTXVScheduler` |

**Значение параметров `LTXVScheduler`:**

| Параметр | По умолчанию | Эффект при увеличении |
|-------|---------|---------------------|
| `max_shift` | 2.05 | Сдвигает расписание к более шумным sigmas в начале → сильнее глобальное движение, рыхлее структура |
| `base_shift` | 0.95 | Наклоняет всю кривую к большему шуму; снижайте для более линейной flow-matching кривой |
| `stretch` | True | Если True, перешкаливает sigmas так, чтобы расписание заканчивалось на `terminal` вместо ~0 |
| `terminal` | 0.1 | Sigma отсечения; denoising останавливается здесь, оставляя запас для stage-2. Ниже = чище stage-1; выше = больше запаса деталей |

**Distilled-воркфлоу использует `ManualSigmas` вместо `LTXVScheduler`.** Поставленный 2.3 single-stage distilled JSON хардкодит:

```
sigmas = "1.0, 0.99375, 0.9875, 0.98125, 0.975, 0.909375, 0.725, 0.421875, 0.0"
```

(8 эффективных шагов + terminal 0.) Поменяйте ноду, если хотите другое distilled-поведение; расписание **не** запечено в модель.

### 9.8 VRAM-уровни (LTX-2.3 22B)

| VRAM | Рекомендуемая сборка |
|------|-------------------|
| 48+ ГБ | BF16 dev + Gemma BF16; полный two-stage; CFG до 5 |
| 24–32 ГБ | FP8 dev + Gemma fp8_scaled; two-stage; `--reserve-vram 5` |
| 16 ГБ | NVFP4 или GGUF Q4_K_M + Gemma fp4_mixed; distilled LoRA; tiled VAE; кэш кондиционирования |
| ≤12 ГБ | GGUF Q2/Q3 + `GemmaAPITextEncode`; distilled LoRA; только single stage |

Универсально: `VAEDecodeTiled` / `LTXVTiledVAEDecode`, batch=1, low-VRAM варианты лоадеров из пакета, запуск с `--reserve-vram 5`.

**Параметры VAE-тайлов:**

- Generic `VAEDecodeTiled` виджеты `(tile_size, overlap, temporal_size, temporal_overlap)` — pixel-space spatial tile, spatial overlap, кадров на временной тайл, overlap кадров. По умолчанию для LTX-2: `(512, 64, 4096, 8)`. **Безопасные значения для 12 ГБ:** `(256, 32, 32, 4)`.
- LTX-специфичный `LTXVTiledVAEDecode` виджеты `(tile_height, tile_width, overlap, force_input_latent, decode_method, upscale_method)` — размеры тайлов в **латентных тайлах** (не пикселях). Two-stage distilled JSON использует `(2, 2, 6, false, "auto", "auto")`. **Безопасно для 12 ГБ:** `(1, 1, 4, false, "auto", "auto")`.

### 9.9 Таблица LTX-2-специфичных ошибок

| Симптом | Причина | Решение |
|---------|-------|-----|
| Sampler shape mismatch (например, "4 vs 3") | Обычный video latent скормлен AV-aware DiT | Используйте `EmptyLTXVLatentVideo` + `LTXVEmptyLatentAudio` + `LTXVConcatAVLatent` |
| Sampler shape mismatch (другие оси) | W/H не 32-stride или frames ≠ `8n+1`; mismatch stage-1↔stage-2 | Подгоните к валидным размерам |
| Абракадабра / тихий CLIP-фейл | Generic `CLIPLoader` на Gemma, или HF Git-LFS pointers | Используйте `LTXAVTextEncoderLoader` + единый safetensors |
| OOM во время text encode | Footprint Gemma | fp4_mixed / fp8_scaled; CPU-offload Gemma; `GemmaAPITextEncode`; кэш через `LTXVSaveConditioning` |
| OOM во время VAE decode | Высокое разрешение / много кадров | `VAEDecodeTiled (512, 64, 4096, 8)`; FP8 чекпоинт |
| Статичный / призрачный I2V | Отсутствует препроцессинг | Вставьте `LTXVPreprocess` перед `LTXVImgToVideoInplace` |
| Audio desync / silent track mismatch | Audio latent короче видео или неверный fps | Сравняйте длину audio latent; выровняйте `LTXVConditioning.frame_rate` с `VideoCombine.fps` |
| Перенасыщение при высоком CFG | Latent overbake | Оберните stage-1 сэмплер `LTXVNormalizingSampler` |
| "Missing node" ошибки | Устаревший `ComfyUI-LTXVideo` | Обновите через Manager, перезапустите |

### 9.10 Чек-лист быстрого старта

1. Установите **ComfyUI-LTXVideo** + **ComfyUI-VideoHelperSuite** + (опционально) **ComfyUI-GGUF**.
2. Положите `ltx-2.3-22b-dev-fp8.safetensors` в `models/checkpoints/`.
3. Положите `gemma_3_12B_it_fp8_scaled.safetensors` в `models/text_encoders/` (один файл, без HF clone).
4. Положите `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` в `models/latent_upscale_models/`.
5. Откройте `example_workflows/2.3/Two_Stage_Distilled.json` из node pack'а.
6. Проверьте, что W/H кратны 32 и frames = `8n+1` перед очередью.
7. Первый прогон: закэшируйте закодированное кондиционирование через `LTXVSaveConditioning`, чтобы пропускать Gemma на последующих генерациях.

### 9.11 Аудио-промптинг

LTX-2.3 использует **единый общий текстовый промпт** для видео и аудио — нет отдельного поля audio conditioning, и **нет задокументированного синтаксиса bracket-tag** (`[sound: ...]` и т.п.). Звуковые подсказки пишутся как естественный язык внутри того же positive-промпта.

**Что работает (из поставленных шаблонов):**
- Назвать источник: *"Soft traditional koto music plays in the background."*
- Описать ритм/текстуру: *"The bamboo whisk taps rhythmically against the ceramic bowl while water simmers in an iron kettle."*
- Речь: *"...with soft-spoken words"* или *"a child's laughter echoes briefly"* — генерирует speech-like аудио (без lip-sync к конкретным словам; относитесь к диалогу как к ambience, а не к скрипту).
- Слоение: комбинируйте ambient bed + foreground sound + случайный акцент в одном предложении.

**Чего избегать:**
- Негативные промпты, нацеленные на аудио, не имеют установленной конвенции. Держитесь визуально-концептных негативов (*"pc game, console game, video game, cartoon, childish, ugly"*).
- Не пытайтесь использовать теги `[silent]` или `[no audio]` — они не парсятся. Для тихого выхода просто опустите упоминания звука и дайте audio latent сгенерировать ambient quiet.
- Не ждите лирики или конкретных музыкальных ключей — audio-модель ambient/cinematic, а не generative-music в смысле Suno.

**Per-modality balance** контролируется на сэмплере через STG / cross-modal scales `MultimodalGuider`'а, **а не** на уровне промпта.

### 9.12 Рабочий пример: 4-секундный кинематографический T2V с аудио (24 ГБ карта)

Конкретные настройки для подстановки в свежий `Two_Stage_Distilled.json`:

| Компонент | Значение |
|-----------|-------|
| Чекпоинт | `ltx-2.3-22b-dev-fp8.safetensors` |
| Текстовый энкодер | `gemma_3_12B_it_fp8_scaled.safetensors` через `LTXAVTextEncoderLoader` |
| Speed LoRA | `ltx-2.3-22b-distilled-lora-384-1.1.safetensors` @ str=0.2 |
| Разрешение (stage 1) | 768 × 512 |
| Кадры | 97 (= 8·12 + 1) |
| Длина audio latent | ≥ 97 |
| FPS | 25 (`LTXVConditioning.frame_rate=25`, `VideoCombine.fps=25`) |
| Stage-1 sampler | `euler`, steps=20, `LTXVScheduler` defaults, `MultimodalGuider scale=28` |
| Spatial upscaler | `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` → 1536×1024 |
| Stage-2 sampler | `euler`, steps=4, `CFGGuider cfg=1.0` |
| VAE decode | `LTXVTiledVAEDecode (2, 2, 6, false, auto, auto)` |
| Positive prompt | (ваша сцена + описание звука, см. §9.11) |
| Negative prompt | `pc game, console game, video game, cartoon, childish, ugly` |

Ожидаемое время stage-1 + stage-2 генерации на 4090: ~80–120s. Пиковая VRAM: ~20 ГБ.

---

## 10. LTX-2.3 продвинутый уровень: стэкинг LoRA, управление камерой, мульти-keyframe анкоры

Этот раздел покрывает то, что нужно подключить поверх базового пайплайна §9, когда вы хотите **направленное движение камеры**, **структурное управление** (depth/pose/Canny) или **несколько якорных кадров** (first+last, first+middle+last, произвольные keyframes).

### 10.1 LoRA-лоадеры — две разные ноды, не взаимозаменяемые

| Лоадер | Использовать для | Почему |
|--------|---------|-----|
| `LoraLoaderModelOnly` | Distilled-speed LoRA, style LoRAs, **camera-control LoRAs** | Стандартная нода ComfyUI. "Model-only" потому что у LTX-2.3 нет CLIP-ветки на стороне LoRA (Gemma сидит снаружи) |
| `LTXICLoRALoaderModelOnly` | **IC-LoRAs** (Union, Motion-Track, Detailer, Pose, Canny) | LTX-специфична. Извлекает `ref_downscale_factor` (`ref0.5` в имени файла) и экспозит его как второй выход, который должен идти в `LTXAddVideoICLoRAGuide` |

**Стэкинг линеен** — пропускайте MODEL через каждый лоадер последовательно. Тестированный практический предел: ~3 LoRA одновременно.

```
CheckpointLoaderSimple
   └── MODEL ──▶ LoraLoaderModelOnly (distilled-lora-384-1.1, str=0.5)
                    └── MODEL ──▶ LoraLoaderModelOnly (camera or style LoRA, str=0.5–0.8)
                                     └── MODEL ──▶ LTXICLoRALoaderModelOnly (ic-lora-union, str=1.0)
                                                      ├── MODEL ──▶ CFGGuider / SamplerCustomAdvanced
                                                      └── downscale_factor ──▶ LTXAddVideoICLoRAGuide
```

Ориентиры по силе:
- Distilled-speed LoRA: **0.5** с distilled-чекпоинтом, **0.2** с полным dev-чекпоинтом.
- Camera LoRA: **0.5–0.8**. Выше 0.8 → артефакты накопления движения.
- Style LoRA: **≤0.6** в комбинации с IC-LoRA, чтобы геометрия оставалась за IC-веткой.

### 10.2 Camera-control LoRAs

Все camera-motion LoRAs **обучены на LTX-2 (19B)**, но загружаются на LTX-2.3 через `LoraLoaderModelOnly`. 2.3-нативных camera LoRAs пока нет. Положите их в `models/loras/ltxv/ltx2/`:

| Файл | Эффект |
|------|--------|
| `ltx-2-19b-lora-camera-control-dolly-in.safetensors` | Камера движется к субъекту |
| `ltx-2-19b-lora-camera-control-dolly-out.safetensors` | Камера отъезжает назад |
| `ltx-2-19b-lora-camera-control-dolly-left.safetensors` | Боковая прокатка влево |
| `ltx-2-19b-lora-camera-control-dolly-right.safetensors` | Боковая прокатка вправо |
| `ltx-2-19b-lora-camera-control-jib-up.safetensors` | Камера поднимается (crane up) |
| `ltx-2-19b-lora-camera-control-jib-down.safetensors` | Камера опускается |
| `ltx-2-19b-lora-camera-control-static.safetensors` | Фиксирует камеру (подавляет дрейф) |

Pan / tilt LoRAs **не опубликованы** на момент написания.

**Mid-clip переключение камеры нативно не поддерживается.** Нет `LoraScheduler`, `LoraTimeMask` или temporal-mask входа на `LoraLoaderModelOnly`. Конвенция:

1. Отрендерьте clip A с camera LoRA #1 (например, dolly-in).
2. Отрендерьте clip B, начиная с последних кадров clip A, с camera LoRA #2 (например, jib-up). См. §10.5.
3. Сшейте в `VideoCombine` или внешнем редакторе.

**Усильте LoRA в промпте.** Специальных trigger-токенов для camera LoRA нет — Lightricks не публикует trigger-фразы. Конвенция — описать движение на простом английском, чтобы bias LoRA и промпт смотрели в одну сторону:

| LoRA | Предлагаемая промпт-фраза |
|------|--------------------------|
| dolly-in | *"the camera slowly dollies in toward the subject"* |
| dolly-out | *"the camera pulls back, revealing more of the scene"* |
| dolly-left / right | *"camera tracks left/right alongside the subject"* |
| jib-up | *"the camera rises smoothly on a crane, revealing the landscape"* |
| jib-down | *"the camera descends from above"* |
| static | *"static locked-off camera, no pan or tilt, no camera movement"* |

Если промпт и LoRA расходятся — результат месиво. Dolly-in LoRA + промпт "the camera holds still" — самая частая foot-gun.

### 10.3 IC-LoRAs и пайплайн guide'ов (depth / pose / Canny / motion track)

IC-LoRAs ("in-context" LoRAs) берут **референсное видео** структурных подсказок — depth-карты, edges, pose-скелеты или sparse motion tracks — и используют их для ограничения генерации.

Доступные IC-LoRAs (положите в `models/loras/ltxv/ltx2/`):

| Файл | Подсказки |
|------|------|
| `ltx-2.3-22b-ic-lora-union-control-ref0.5.safetensors` | Canny + Depth + Pose объединены в одной LoRA |
| `ltx-2.3-22b-ic-lora-motion-track-control-ref0.5.safetensors` | Sparse motion vectors (нарисованные tracks) |
| `ltx-2-19b-ic-lora-detailer.safetensors` | V2V refiner / detail enhancer |
| `ltx-2-19b-ic-lora-pose-control.safetensors` | Только Pose (LTX-2 эра; для 2.3 заменена Union) |
| `ltx-2-19b-ic-lora-canny-control.safetensors` | Только Canny (заменена Union) |

**Разводка (пример Union-Control):**

```
LoadVideo ──▶ GetVideoComponents ──▶ ResizeImageMaskNode ─┬─▶ VideoDepthAnythingProcess ──┐
                                                          ├─▶ CannyEdgePreprocessor      ─┤
                                                          └─▶ DWPreprocessor              ─┤
                                                                                           ▼
                                                                            LTXAddVideoICLoRAGuide
                                                                            (downscale_factor вход
                                                                             от LTXICLoRALoaderModelOnly)
                                                                                  │
                                  (только после stage-2 spatial upsample:)        ▼
                                                                            LTXVCropGuides
                                                                                  │
                                                                                  ▼
                                                                          positive conditioning
                                                                          ──▶ CFGGuider ──▶ Sampler
```

Заметки:
- Препроцессоры depth/Canny/pose — это **обычные ControlNet-aux ноды** — нет `LTXVPrepareDepth` / `LTXVPrepareCanny` обёрток. Для depth используйте `VideoDepthAnythingProcess` (с `LoadVideoDepthAnythingModel`) → `VideoDepthAnythingOutput`.
- Для motion-track IC-LoRA замените препроцессоры на **`LTXVDrawTracks` + `LTXVSparseTrackEditor`** (вы буквально рисуете motion-векторы в редакторе).
- **`LTXVCropGuides`** нужен только после `LTXVLatentUpsampler` (stage 2). Он обрезает guide-латент под апсэмплированный пространственный размер — IC-LoRAs обучены на `ref_downscale=0.5`.
- **`LTX Add Video IC-LoRA Guide Advanced`** добавляет `attention_strength` и `attention_mask` для пространственно-локализованного управления (например, применить depth только внутри маски региона).

Референсные воркфлоу (в `custom_nodes/ComfyUI-LTXVideo/example_workflows/2.3/`):
- `LTX-2.3_ICLoRA_Union_Control_Distilled.json`
- `LTX-2.3_ICLoRA_Motion_Track_Distilled.json`

### 10.4 Multi-keyframe / anchor-frame I2V

Три ноды делают тяжёлую работу:

| Нода | Роль | Лучше всего для |
|------|------|----------|
| `LTXVImgToVideoConditionOnly` | Single first-frame seed через кондиционирование | I2V-шаблон по умолчанию |
| `LTXVImgToVideoInplace` | VAE-кодирует изображение и **смешивает** в латент-кадры с заданной силой (rigid) | Жёсткие first/last анкоры, должны точно совпадать |
| `LTXVAddGuide` | **Soft** анкор — модель ведёт переговоры вокруг него. Принимает `frame_idx` (`-1` = последний кадр) | Множественные keyframes, не-граничные анкоры |

**Критично:** каждое anchor-изображение требует `LTXVPreprocess` сначала (соответствие тренировочному сжатию — пропустите и выход станет статичным/призрачным), и должно быть размером с латент (используйте `ResizeImageMaskNode`).

#### FLF2V — First + Last Frame to Video

**Нет** выделенной ноды `LTXVFirstLastFrame` и нет поставленного `FLF2V.json` в 2.3 примерах. Постройте из двух вызовов `LTXVAddGuide`:

```
LoadImage(first) ─▶ LTXVPreprocess ─▶ LTXVAddGuide(frame_idx=0,  strength=1.0) ─┐
LoadImage(last ) ─▶ LTXVPreprocess ─▶ LTXVAddGuide(frame_idx=-1, strength=1.0) ─┴─▶ positive conditioning
```

#### FMLF2V — First + Middle + Last

Добавьте третий вызов `LTXVAddGuide` в средней точке:

```
LTXVAddGuide(frame_idx = total_frames // 2, strength = 0.5)   ◀── middle anchor (lower strength)
```

Рекомендуемая стартовая сила для среднего анкора — **~0.5** (по Kijai). Поднимайте, если средний дрейфует; снижайте, если создаёт видимый "snap".

#### Произвольные anchor-кадры на любом frame index

Два маршрута:

1. **Официальный:** соедините цепочкой N копий `LTXVAddGuide`, по одной на анкор, каждая со своим `frame_idx` и `strength`.
2. **Kijai fork:** `LTXVAddGuideMulti` принимает список `(frame_index, image, strength)` кортежей в одной ноде. Живёт в `Kijai/LTX2.3_comfy` (HuggingFace), не в официальном Lightricks-репо.

**Нет per-anchor denoise schedule** — `strength` per call единственный per-anchor рычаг. Sampler sigmas (`ManualSigmas` / `LTXVScheduler`) глобальны.

### 10.5 Расширение, продолжение и зацикливание видео

Для длинных видео генерируйте чанками и дайте модели перекрывать каждый чанк хвостом предыдущего.

**Рекомендуется (официально):** используйте `LTXVLoopingSampler` вместо `SamplerCustomAdvanced`. **Заметка:** требует `STGGuiderAdvanced`, не `CFGGuider` или `MultimodalGuider`.

Полная таблица параметров (из исходника ноды):

| Параметр | По умолчанию | Диапазон | Значение |
|-------|---------|-------|---------|
| `temporal_tile_size` | 80 | 24–1000 | Кадров на тайл |
| `temporal_overlap` | 24 | 16–80 | Кадров переиспользуется из предыдущего тайла (каждый тайл продвигается на `tile_size − overlap`) |
| `temporal_overlap_cond_strength` | 0.5 | 0–1 | Насколько сильно хвост предыдущего тайла ограничивает новый (поднимите до 0.7 для сильнее непрерывности, снизьте до 0.3, если швы выглядят замёрзшими) |
| `guiding_strength` | 1.0 | 0–1 | Сила IC-LoRA / guiding latents |
| `cond_image_strength` | 1.0 | 0–1 | Сила conditioning image (I2V) |
| `horizontal_tiles` / `vertical_tiles` | 1 | 1–6 | Пространственный тайлинг (для очень высокого res) |
| `spatial_overlap` | 1 | 1–8 | Пространственный tile overlap |

**Конкретный рецепт — 250 кадров @ 25 fps (10 секунд):**

```
temporal_tile_size = 80
temporal_overlap   = 24
temporal_overlap_cond_strength = 0.5
```

→ `ceil((250 − 24) / (80 − 24)) ≈ 5 тайлов`. Каждый тайл генерирует 80 кадров; 24 перекрываются с предыдущим. Пиковая VRAM примерно `temporal_tile_size / total_frames` от single-shot пика (≈ 32% здесь), ценой per-tile overhead и небольшой мягкости на швах около граничных кадров.

**Manual stitch fallback (без looping sampler'а):** декодируйте clip A → возьмите его последние N кадров как батч → подайте в свежий run с `LTXVAddGuide` на `frame_idx=0..N-1`. Удалите перекрывающиеся кадры перед конкатенацией в `VideoCombine`.

**Looping (первый кадр == последний):** нет first-class фичи. Установите одно изображение и как первый, и как последний анкор (`LTXVAddGuide` на `frame_idx=0` и `frame_idx=-1`, оба strength 1.0), и сэмплер сходится к лупу. Качество варьируется — простые сцены (медленная камера, повторяющееся движение) лупаются чище, чем сложное действие.

### 10.6 Комбинирование camera LoRA с keyframe-анкорами

Механически совместимы — `LoraLoaderModelOnly` модифицирует MODEL до того, как `LTXVAddGuide` оперирует на кондиционировании, поэтому на уровне нод конфликта нет. Поведенческие нюансы:

- **Anchor-кадры выигрывают на границах.** "dolly-in" LoRA + last-frame anchor, противоречащий направлению зума → будут драться; результат выглядит сломанным.
- **IC-LoRA + camera LoRA**: предпочитайте IC-LoRA для траектории и либо отключайте camera LoRA, либо держите её на **≤0.5**. Обе хотят управлять движением; IC-LoRA конкретнее.
- **FLF2V + camera LoRA**: задокументировано как работающее. **FMLF2V + camera LoRA**: не задокументировано в обе стороны — тестируйте на коротких клипах сначала.

### 10.7 Что НЕ поддерживается (не тратьте время на поиск)

- Mid-clip переключение LoRA (нет `LoraScheduler` или temporal-mask LoRA лоадера для LTX).
- Pan / tilt camera LoRAs (не выпущены).
- Per-anchor denoise schedule.
- Нативный loop / first==last шаблон.
- Pan/tilt или roll, выведенные из static LoRA — `static` только подавляет движение.

Для неподдерживаемых поведений стандартный обход — **рендерить сегментами и сшивать** (§10.5).

### 10.8 Рабочий пример: FLF2V с промежуточным анкором на кадре 49

Цель: сгенерировать 97-кадровый клип, где кадр 0 = портрет A, кадр 48 = портрет B (mid-pose), кадр 96 = портрет C, со slow dolly-in throughout.

**Настройки:**

| Компонент | Значение |
|-----------|-------|
| Всего кадров | 97 |
| Anchor 1 | `LoadImage(A)` → `LTXVPreprocess` → `LTXVAddGuide(frame_idx=0,  strength=1.0)` |
| Anchor 2 | `LoadImage(B)` → `LTXVPreprocess` → `LTXVAddGuide(frame_idx=48, strength=0.5)` |
| Anchor 3 | `LoadImage(C)` → `LTXVPreprocess` → `LTXVAddGuide(frame_idx=-1, strength=1.0)` |
| Camera LoRA | `dolly-in` @ str=0.5 (ниже обычного, чтобы не драться с анкорами) |
| Speed LoRA | `distilled-lora-384-1.1` @ str=0.5 |
| Разрешение | 768×512 (stage 1) → 1536×1024 (stage 2) |
| Sampler | distilled `ManualSigmas` (8 шагов, см. §9.7) + `CFGGuider cfg=1.0` |
| Promp | *"the camera slowly dollies in toward the subject; subtle ambient room tone"* |

**Эскиз разводки:**

```
CheckpointLoaderSimple ─▶ LoraLoaderModelOnly(distilled, 0.5) ─▶ LoraLoaderModelOnly(dolly-in, 0.5) ─▶ MODEL ─┐
                                                                                                              │
LTXAVTextEncoderLoader ─▶ CLIPTextEncode(pos)+(neg) ─▶ LTXVConditioning(fps=25) ─┐                            │
                                                                                  │                           │
LoadImage(A) ─▶ LTXVPreprocess ─▶ LTXVAddGuide(idx=0,  str=1.0) ──┐               │                           │
LoadImage(B) ─▶ LTXVPreprocess ─▶ LTXVAddGuide(idx=48, str=0.5) ──┼─▶ positive ───┤                           │
LoadImage(C) ─▶ LTXVPreprocess ─▶ LTXVAddGuide(idx=-1, str=1.0) ──┘               │                           │
                                                                                   │                          │
EmptyLTXVLatentVideo(768, 512, 97) ─┐                                              │                          │
LTXVEmptyLatentAudio(≥97)           ─┴─▶ LTXVConcatAVLatent ─▶ AV latent ──────────┴───▶ CFGGuider ◀──────────┘
                                                                                              │
                                                              ManualSigmas + KSamplerSelect ──▶ SamplerCustomAdvanced ─▶ ...
```

**Тюнинг, если выглядит неправильно:**
- Средний анкор хлопает/snap'ает → снизьте `strength` с 0.5 до 0.3.
- Средний анкор дрейфует → поднимите до 0.7.
- Камера не движется → поднимите camera-LoRA strength до 0.7 или перефразируйте промпт явнее ("a clear, smooth dolly-in over the full duration").
- Камера дерётся с end anchor → снизьте camera LoRA до 0.3 или уберите её; пусть промпт + позиции анкоров намекают на движение.

### 10.9 Quick-reference совместимости

| Комбинация | Работает? | Заметки |
|-------------|--------|-------|
| Distilled LoRA + camera LoRA | Да | Обе через `LoraLoaderModelOnly`, цепочкой последовательно |
| Camera LoRA + IC-LoRA | Caveat | Держите camera LoRA ≤0.5; IC-LoRA хочет траекторию |
| FLF2V + camera LoRA | Да | Camera LoRA формирует внутреннюю траекторию; анкоры выигрывают на границах |
| FMLF2V + camera LoRA | Не тестировано | Тестируйте на коротких клипах сначала |
| Несколько IC-LoRAs одновременно | Не тестировано | Один IC-LoRA на цепь в поставленных шаблонах; Union уже объединяет depth+canny+pose |
| Camera LoRA + audio | Да | Аудио независимо от model-side LoRA |
| `LTXVLoopingSampler` + camera LoRA | Да | LoRA модифицирует MODEL, looping sampler использует MODEL — конфликта нет |
| `LTXVLoopingSampler` + IC-LoRA | Да | Регулятор `guiding_strength` контролирует вес IC-LoRA per tile |

---

## 11. Промптинг Gemma для LTX-2.3 (vs. CLIP)

Текстовый энкодер LTX-2.3 — это **Gemma 3 12B Instruct**, а не CLIP. Большинство онлайн-советов по промптингу (SD, SDXL, Flux, Midjourney) предполагают CLIP и не переносятся. Этот раздел фиксирует различия.

### 11.1 Что меняется vs. CLIP-конвенции

| SD / CLIP конвенция | Gemma / LTX-2.3 реальность |
|----------------------|--------------------------|
| Comma-separated токены (`red dress, long hair, forest, night`) | Работает, но **хуже** прозы. Gemma понимает полные предложения. |
| Синтаксис эмфазы `(word:1.4)` | **Не парсится как взвешивание.** Скобки читаются как пунктуация. Подчёркивайте через повторение + конкретику + front-loading. |
| Предел контекста 77 токенов | Не проблема. Промпты длиной с абзац — это нормально. |
| Короткие промпты (`anime girl, blue eyes, sword`) | Недоопределены — Gemma спроектирована под длинный контекст. |
| Длинный SD-style negative laundry list | Конвенция Lightricks: ≤8 concept-level токенов (см. §9.4). |
| Порядок слов почти не важен | Порядок слов **важен**. Первое предложение задаёт стиль; последующие наслаивают конкретику. |

**Суть:** пишите как оператор, описывающий кадр съёмочной группе, а не как SD power-user, тегирующий изображение.

### 11.2 Пятипараграфный shot prompt

Gemma читает пустые строки как семантические разделители. Эта структура превосходит сплошную стену текста с тем же содержанием:

```
¶1  STYLE + MEDIUM          (копипастится через весь проект)
¶2  SUBJECT + FRAMING       (кто/что, в каком shot type)
¶3  ACTION + CAMERA         (что происходит + как ведёт себя камера)
¶4  LIGHTING + ATMOSPHERE   (look сцены)
¶5  SOUND                   (audio-слои — см. §9.11)
```

Параграф 1 — это ваша **style preamble**. Копипастьте дословно через каждый шот проекта. Перепись для каждого шота — главная причина cross-shot стилевого дрейфа.

### 11.3 Эмфаза без весов

Поскольку `(word:1.4)` ничего не делает, рабочие эквиваленты:

1. **Повторение между параграфами.** Упомяните критический визуальный элемент в двух разных параграфах.
2. **Конкретика.** `"a fitted cherry-red costume with black spider-web pattern stitched across the chest"` бьёт `(red costume:1.4)`.
3. **Front-loading.** Критические элементы идут в ¶2, не в ¶5.
4. **Плотность прилагательных.** `"a tall, gaunt, deeply shadowed figure"` подчёркивает больше, чем `"a tall figure"`.

### 11.4 Глаголы движения — фикс №1 для статичного выхода

Слабые глаголы дают призрачный/замёрзший выход. Сильные — движение. Каждое предложение в ¶3 должно содержать конкретный motion verb.

| Слабо | Сильно |
|------|--------|
| "The character is in the forest" | "The character walks through the forest" |
| "She looks sad" | "She lowers her gaze and exhales slowly" |
| "Rain falls" | "Heavy rain streaks diagonally across the frame" |
| "The car moves" | "The car accelerates hard, tires spinning, toward camera-right" |

Pacing-глаголы (`slowly`, `suddenly`, `gradually`, `smoothly`, `abruptly`) влияют на темп сгенерированного движения.

### 11.5 Названные цвета бьют hex-коды

Gemma знает `"cherry red"`, `"crimson"`, `"oxblood"` — каждый со своими визуальными ассоциациями. Она не знает `#B22222`.

| Generic | Конкретно |
|---------|----------|
| "blue" | navy, cobalt, electric blue, periwinkle, teal, sky blue, denim |
| "red" | cherry red, crimson, scarlet, brick, oxblood, coral, rose |
| "green" | emerald, forest, sage, chartreuse, jade, olive |
| "yellow" | golden, mustard, cream, lemon, amber |

### 11.6 Gemma-специфичные возможности, которые стоит знать

Gemma — полноценная LLM; фичи, которых у CLIP нет:

- **Negation в positive prompt работает**: `"no camera movement"`, `"without any text on screen"` — CLIP часто инвертирует; Gemma уважает.
- **Sequences**: `"first she stands still, then she turns toward the camera"` — шот с двумя битами.
- **Counting**: `"three figures in the background"` — надёжно до ~5; деградирует выше.
- **Comparatives**: `"taller than the figure behind her"`, `"quieter than the previous shot"`.
- **Causality**: `"because of the heavy rain, the streets are empty"` — добавляет мотивацию, которую модель может выразить визуально.

### 11.7 Известные слабости

- **Рендеринг текста на экране** — фейлит; добавляйте текст в посте.
- **Точный счёт выше ~5** — `"seven candles"` часто рендерится как 4 или 8.
- **Руки, выполняющие конкретные действия пальцами в CU**.
- **Brand-name предметы** (конкретные логотипы, конкретные модели машин) — сильно дрейфуют.

### 11.8 Когда расширять негативный промпт

Начните с дефолта Lightricks (`pc game, console game, video game, cartoon, childish, ugly`). Добавляйте concept-level термины **только** когда видите конкретную проблему:

| Проблема | Добавить |
|---------|-----|
| Слишком anime, когда нужна американская анимация | `anime, manga, japanese animation` |
| Слишком 3D-rendered | `3D rendered, CGI, pixar, video game` |
| Слишком modern для retro-таргета | `modern, HDR, digital sharpness` |
| Слишком realistic для стилизованного таргета | `photorealistic, photography, live action` |

**Не** добавляйте SD-эра токены вроде `blurry, low quality, jpeg artifacts, extra fingers, bad hands, deformed`. Тренировка Lightricks их не взвешивала, и они вытесняют полезные негативные токены. Cap total на ~8 концептов.

### 11.9 Протокол итерации — меняй одно

1. **Зафиксируйте сид** в `RandomNoise`.
2. **Идентифицируйте ОДНО, что не так** (конкретный визуальный элемент, не вайб).
3. **Отредактируйте ОДИН параграф** (тот, который владеет этой штукой).
4. **Перезапустите на низком res** (512×384, 49 кадров → ~25s на 4090).
5. **Сравните с предыдущей версией.** Изменилось ли одно? Остальные остались?
6. **Оставьте или откатите.**

Изменение промпта и сида одновременно во время итерации теряет весь сигнал.

---

## 12. Справочник нод

Этот раздел — компактный лукап по нодам, которые встречаются в §§2–10. Каждая строка таблицы даёт назначение ноды, её основные входы/выходы и виджеты, которые вы реально трогаете. За исчерпывающими per-node страницами (особенно по edge-case виджетам) идите по ссылкам в конце раздела.

**Как читать раздел:**
- Имена виджетов в нижнем регистре (`cfg`, `sampler_name`) — они совпадают с лейблами ComfyUI UI.
- «↑» означает «повышение значения →»; «↓» — наоборот. Направления описывают *типичные* эффекты; несколько нод инвертируют их около крайних значений.
- Значения по умолчанию / рекомендованные предполагают работу с LTX-2.3, если не указано иное.

---

### 12.1 Core loader-ноды

| Нода | Назначение | Ключевые виджеты |
|------|------------|-------------------|
| `CheckpointLoaderSimple` | Грузит объединённый чекпойнт (`MODEL + CLIP + VAE`) из `models/checkpoints/`. | `ckpt_name` — файл для загрузки. Других виджетов нет; precision выводится из файла. |
| `UNETLoader` / `DiffusionModelLoader` | Грузит только denoising backbone. Используется, когда модель поставляется раздельными файлами (LTX-2.3 dev/fp8/nvfp4 чекпойнты работают с этой нодой, если вы отдельно грузите VAE + text encoder). | `unet_name` — файл. `weight_dtype` — `default` / `fp8_e4m3fn` / `fp8_e5m2`. ↓ precision = ↓ VRAM, ↑ риск color shift / бэндинга. |
| `VAELoader` | Грузит отдельный VAE. | `vae_name` — файл. Видео-VAE включают temporal-ось; не смешивайте с image-only VAE. |
| `CLIPLoader` / `DualCLIPLoader` | Грузит CLIP / T5-XXL text encoders для SD-family / Wan / Hunyuan / CogVideoX. | `clip_name(_1/_2)`, `type` — должен соответствовать семейству модели (`sdxl`, `flux`, `wan`, и т. д.). **Не использовать для LTX-2.3** — см. `LTXAVTextEncoderLoader` в §12.5. |
| `UnetLoaderGGUF` (ComfyUI-GGUF) | Грузит GGUF-квантизованные backbones (Q2_K…Q8_0). | `unet_name` — `.gguf` файл. Уровень квантизации запечён внутри; ↓ квант = ↓ VRAM, ↓ точность. |
| `DualCLIPLoaderGGUF` (ComfyUI-GGUF) | GGUF-квантизованные text encoders, включая Gemma через `type="ltxv"`. | `clip_name_1/2`, `type` — должен быть `ltxv` для LTX-2.3. |
| `LoraLoader` | Применяет LoRA к `MODEL` + `CLIP`. | `lora_name`, `strength_model` (0–1+), `strength_clip` (0–1+). ↑ strength = сильнее стилистический pull, ↑ риск оверфита/артефактов выше ~1.2. |
| `LoraLoaderModelOnly` | Применяет LoRA только к `MODEL` (используется LTX — Gemma вне графа LoRA). | `lora_name`, `strength_model`. См. §10.1 по типичным strengths. |

---

### 12.2 Ноды text / image кондиционирования

| Нода | Назначение | Ключевые виджеты |
|------|------------|-------------------|
| `CLIPTextEncode` | Кодирует промпт в тензор `CONDITIONING`. | `text` — строка промпта. Всё. Синтаксис усиления (`(word:1.4)`) работает на CLIP, **не на Gemma** — см. §11.1. |
| `LoadImage` | Грузит картинку из `input/`. | `image` — file picker, кнопка `upload`. Опционально выход `mask`. |
| `VAEEncode` | Кодирует картинку в латентное пространство. | `pixels`, `vae` — только входы; виджетов нет. Необходимо перед image→latent инъекцией в I2V-графах. |
| `VAEDecode` | Декодирует latent обратно в пиксели (все кадры за один проход). | `samples`, `vae` — только входы. Высокий VRAM на длинных/HD видео — используйте tiled-вариант ниже. |
| `VAEDecodeTiled` | Декодирует пространственными + временными тайлами. | `tile_size` (px, пространственный тайл), `overlap` (px, пространственный overlap), `temporal_size` (кадров на тайл), `temporal_overlap` (кадры). LTX-значения см. в §9.8. |
| `CLIPVisionEncode` | Извлекает semantic image features для IP-Adapter-style гайдинга. | `clip_vision`, `image` — виджетов нет. Выход — *не* первокадровый latent; это style/content эмбеддинг. |

---

### 12.3 Ноды latent и сэмплинга

| Нода | Назначение | Ключевые виджеты |
|------|------------|-------------------|
| `EmptyLatentVideo` | Generic empty video latent (используется Wan, Hunyuan, Cog). | `width`, `height`, `length` (кадров), `batch_size`. У каждой оси есть model-specific stride-ограничение. |
| `KSampler` | Стандартный ComfyUI sampler. Денойзит latent. | `seed`, `steps` (↑ = ↑ качество, ↑ время), `cfg` (prompt adherence; видео-модели часто 1–7), `sampler_name` (алгоритм), `scheduler` (sigma-расписание), `denoise` (0–1; <1 = частичный renoise для I2V/refine). |
| `KSamplerAdvanced` | KSampler с `start_at_step` / `end_at_step` / `add_noise` — включает two-pass refine, hi-res fix, инъекцию шума. | То же плюс `start_at_step`, `end_at_step`, `return_with_leftover_noise`. |
| `SamplerCustomAdvanced` | Модульный сэмплер: подаёте `NOISE`, `GUIDER`, `SAMPLER`, `SIGMAS`, `LATENT` отдельно. Используется всеми LTX-2.3 workflow. | Скалярных виджетов нет; конфигурация — это проводка. |
| `RandomNoise` | Выдаёт noise-тензор для `SamplerCustomAdvanced`. | `noise_seed` — целочисленный сид. Лочьте его во время итераций (§11.9). |
| `KSamplerSelect` | Выбирает алгоритм шага (выдаёт `SAMPLER`). | `sampler_name` — `euler`, `euler_ancestral`, `dpmpp_2m`, `uni_pc`, `deis`, и т. д. Euler / dpmpp_2m самые безопасные для временной согласованности. |
| `BasicScheduler` | Выдаёт sigma-расписание. | `scheduler` (`normal`, `karras`, `sgm_uniform`, `beta`, `linear_quadratic`), `steps`, `denoise`. Для LTX замещён `LTXVScheduler`. |
| `CFGGuider` | Оборачивает модель + pos/neg conditioning в `GUIDER` с одним скаляром `cfg`. | `cfg` — при 1.0 classifier-free guidance отсутствует (используется distilled LTX); поднимайте для более сильной prompt adherence ценой пересыщения. |

---

### 12.4 Ноды вывода / сборки видео

| Нода | Назначение | Ключевые виджеты |
|------|------------|-------------------|
| `SaveImage` | Пишет отдельные кадры в `output/`. | `filename_prefix`. |
| `VHS_VideoCombine` (VideoHelperSuite) | Собирает кадры → MP4 / WEBM / GIF / WebP. | `frame_rate` (должен совпадать с `LTXVConditioning.frame_rate`), `format` (`video/h264-mp4`, `video/h265-mp4`, `image/gif`, …), `pix_fmt` (`yuv420p` для максимальной совместимости), `crf` (↓ = ↑ качество, ↑ размер; 17–23 типично), `save_metadata`, `audio` (опциональный AUDIO вход). |
| `CreateVideo` | Новая встроенная ComfyUI-альтернатива `VHS_VideoCombine`. | Та же идея; чуть проще набор виджетов. |
| `VHS_LoadVideo` (VideoHelperSuite) | Грузит видео → image batch + audio. | `video` (файл), `force_rate` (ресемпл fps), `force_size` (ресайз), `frame_load_cap` (макс кадров), `skip_first_frames`, `select_every_nth`. |
| `GetImageRangeFromBatch` (KJNodes) | Срезает кадры из batch (используется для сшивки / extension в §10.5). | `start_index`, `num_frames`. Отрицательный `start_index` отсчитывает от конца. |
| `GetVideoComponents` | Разделяет загруженное видео на `IMAGE`, `AUDIO`, `fps`. | Виджетов нет. |

---

### 12.5 LTX-2.3 loaders и кондиционирование

Все требуют `ComfyUI-LTXVideo`.

> **Авторитетный референс для §§12.5–12.8.** За полным актуальным списком виджетов по каждой LTX-2.3 ноде (включая виджеты, опущенные здесь для краткости) идите в официальный репозиторий и reference-воркфлоу:
> - [`github.com/Lightricks/ComfyUI-LTXVideo`](https://github.com/Lightricks/ComfyUI-LTXVideo) — README ноды-пака.
> - `custom_nodes/ComfyUI-LTXVideo/example_workflows/2.3/` — поставляемые JSON (`Two_Stage_Distilled`, `Single_Stage_Distilled_Full`, `ICLoRA_Union_Control`, `ICLoRA_Motion_Track`). Эти файлы — ground truth по поводу того, какие виджеты соединены, а какие оставлены в дефолтах.
> - [`huggingface.co/Lightricks/LTX-2.3`](https://huggingface.co/Lightricks/LTX-2.3) — model card с заметками по conditioning / scheduler.

| Нода | Назначение | Ключевые виджеты |
|------|------------|-------------------|
| `LTXAVTextEncoderLoader` | Грузит Gemma 3 12B Instruct (AV-aware text encoder). **Обязательна** вместо `CLIPLoader` для LTX-2.3. | `text_encoder_name` — single consolidated safetensors из `models/text_encoders/`. Не подавайте Git-LFS pointer-файлы (§9.3). |
| `GemmaAPITextEncode` | Офлоадит кодирование Gemma на API Lightricks — ноль локального Gemma VRAM. | `api_key` (если требуется), `text`. Зависит от сети; не воспроизводимо офлайн. |
| `LTXVSaveConditioning` | Сериализует закодированный conditioning в `models/embeddings/`. | `filename_prefix`. Запускайте один раз на промпт, дальше пропускаете Gemma на rerun. |
| `LTXVLoadConditioning` | Перезагружает сохранённый conditioning. | `conditioning_name`. |
| `EmptyLTXVLatentVideo` | LTX-aware empty video latent (корректный latent-shape для AV DiT). | `width`, `height` (обязаны быть 32-кратны), `length` (обязан быть `8n+1`), `batch_size` (держите 1). |
| `LTXVEmptyLatentAudio` | Пустой audio-latent в паре с видео-латентом. | `length` — должен быть **≥** длины видео-латента, иначе сэмплинг падает. |
| `LTXVConcatAVLatent` | Конкатит видео + audio латенты по modality-оси в единый тензор, который ждёт DiT. | Виджетов нет. |
| `LTXVSeparateAVLatent` | Обратная операция — разбивает выход сэмплера обратно на видео и audio латенты. | Виджетов нет. |
| `LTXVConditioning` | Штампует frame-rate метадату на conditioning-тензор. | `frame_rate` — должен совпадать с `VHS_VideoCombine.fps` (25 — дефолт из шаблонов). |

---

### 12.6 LTX-2.3 schedulers, guiders, samplers

| Нода | Назначение | Ключевые виджеты |
|------|------------|-------------------|
| `LTXVScheduler` | Подтюненное LTX sigma-расписание для полного dev-чекпойнта. | `steps` (20 stage-1, 3–4 stage-2), `max_shift` (2.05; ↑ = больше глобального движения, слабее структура), `base_shift` (0.95; ↑ = более шумная кривая в целом), `stretch` (True оставляет `terminal` как конечную сигму), `terminal` (0.1 оставляет запас для stage-2; ↓ = чище stage-1). Полную таблицу см. в §9.7. |
| `ManualSigmas` | Хардкодит список сигм. Используется distilled 8-step воркфлоу. | `sigmas` — список float через запятую. Шаблонное distilled-расписание в §9.7. |
| `MultimodalGuider` | AV-aware guider с per-modality шкалами и STG. **Обязателен** для генерации video+audio. | `scale` (28 в поставляемом шаблоне, трактуется как «CFG-эквивалент»), `skip_block_indices` (список STG-слоёв), `stg_scale`, `rescale`. Дефолты официально не задокументированы — стартуйте от JSON-шаблона. |
| `STGGuiderAdvanced` | Вариант `MultimodalGuider`, необходимый для `LTXVLoopingSampler`. | Та же идея плюс продвинутые STG-крутилки. |
| `LTXVNormalizingSampler` | Оборачивает другой `SAMPLER`, предотвращая overbake латента на высоком CFG. | `sampler` (вход). Используйте только на stage-1; пропускайте на inpaint/extend. |
| `LTXVLoopingSampler` | Temporal-tiling сэмплер для длинных видео. См. §10.5. | `temporal_tile_size` (80), `temporal_overlap` (24), `temporal_overlap_cond_strength` (0.5; ↑ = сильнее непрерывность, ↓ = больше вариативности), `guiding_strength` (вес IC-LoRA per tile), `cond_image_strength`, `horizontal_tiles`, `vertical_tiles`, `spatial_overlap`. |

---

### 12.7 LTX-2.3 image / keyframe кондиционирование

| Нода | Назначение | Ключевые виджеты |
|------|------------|-------------------|
| `LTXVPreprocess` | Деградирует входную картинку под training-компрессию. **Без неё выход получается статичный/призрачный** (§9.5). | Виджетов нет. |
| `LTXVImgToVideoConditionOnly` | First-frame I2V через conditioning-инъекцию (soft anchor). | `conditioning`, `image`, `vae` — скалярных виджетов нет. |
| `LTXVImgToVideoInplace` | Кодирует картинку и вбивает её в первый латент-кадр (hard anchor). | `strength` (1.0 = полный лок, ↓ = больше drift разрешено). |
| `LTXVAddGuide` | Soft anchor на произвольном кадре. Строительный блок для FLF2V / FMLF2V. | `frame_idx` (`0` = первый, `-1` = последний), `strength` (1.0 для границ, ~0.5 для внутренних якорей — см. §10.4). |
| `LTXVAddGuideMulti` (Kijai fork) | Принимает список `(frame_index, image, strength)` кортежей. | Одна строка на якорь; семантика идентична `LTXVAddGuide`. |
| `LTXVLatentUpsampler` | Stage-2 latent upsampler (нужен чекпойнт spatial-апскейлера). | `upscaler` (вход модели), `scale` (2×). |
| `LatentUpscaleModelLoader` | Грузит `ltx-2.3-spatial-upscaler-x2-*.safetensors` / temporal upscaler. | `model_name`. |
| `LTXVCropGuides` | Подрезает guide-латенты обратно до целевого frame count после апскейла. | Виджетов нет (шейпы читает автоматически). |
| `LTXVTiledVAEDecode` | LTX-специфичный tiled VAE decode. Виджеты в **latent tiles**, не в пикселях (отличается от generic `VAEDecodeTiled`). | `tile_height`, `tile_width`, `overlap`, `force_input_latent`, `decode_method` (`auto`), `upscale_method` (`auto`). Безопасные 12 GB значения в §9.8. |
| `LTXVAudioVAEDecode` | Декодирует audio-латент в стрим `AUDIO`. | Виджетов нет. |

---

### 12.8 LTX-2.3 IC-LoRA и структурный контроль

| Нода | Назначение | Ключевые виджеты |
|------|------------|-------------------|
| `LTXICLoRALoaderModelOnly` | Грузит IC-LoRA и экспонирует её `ref_downscale_factor`. | `lora_name`, `strength_model`. Выходы: `MODEL` и `downscale_factor` — второй подключайте к guide-ноде ниже. |
| `LTXAddVideoICLoRAGuide` | Присоединяет препроцессенное guide-видео (depth / Canny / pose / motion-track) к conditioning. | `positive` / `negative` (conditioning), `vae`, `guide_video`, `downscale_factor`. |
| `LTX Add Video IC-LoRA Guide Advanced` | То же плюс пространственное / attention маскирование. | Добавляет `attention_strength`, `attention_mask`. |
| `LTXVDrawTracks` | Холст-редактор для рисования разрежённых motion-векторов. | Интерактивный виджет. |
| `LTXVSparseTrackEditor` | Компаньон к `LTXVDrawTracks` — редактирует/уточняет motion-треки. | Интерактивный виджет. |

**Препроцессоры (generic, из `comfyui_controlnet_aux` / `ComfyUI-VideoDepthAnything`):**

| Нода | Производит | Ключевые виджеты |
|------|------------|-------------------|
| `VideoDepthAnythingProcess` + `LoadVideoDepthAnythingModel` | Depth-видео для IC-LoRA Union. | Вариант модели (`vits`, `vitb`, `vitl`), входное разрешение. |
| `CannyEdgePreprocessor` | Canny-edge видео. | `low_threshold` (↓ = больше рёбер, больше шума), `high_threshold`. |
| `DWPreprocessor` | OpenPose skeleton-видео. | `detect_hand`, `detect_body`, `detect_face`. |
| `ResizeImageMaskNode` (KJNodes) | Ресайзит image/mask под latent. | `width`, `height`, `upscale_method`. |

---

### 12.9 Куда смотреть, когда этой таблицы недостаточно

Для полной per-widget документации, нюансов взаимодействия и новых нод, не покрытых здесь:

- **Core ComfyUI ноды**: встроенный список нод на `http://127.0.0.1:8188/` → ПКМ → `Help`, или ComfyUI wiki (`docs.comfy.org`).
- **LTX-2.3 ноды**: `custom_nodes/ComfyUI-LTXVideo/README.md` и поставляемые workflow JSON в `example_workflows/2.3/` — эти JSON самые авторитетные по поводу того, какие виджеты ожидаются подключёнными, а какие оставлены в дефолтах.
- **VideoHelperSuite**: README `github.com/Kosinkadink/ComfyUI-VideoHelperSuite`.
- **ComfyUI-GGUF**: `github.com/city96/ComfyUI-GGUF`.
- **KJNodes**: `github.com/kijai/ComfyUI-KJNodes`.
- **controlnet_aux препроцессоры**: `github.com/Fannovel16/comfyui_controlnet_aux`.
- **RIFE / frame interpolation**: `github.com/Fannovel16/ComfyUI-Frame-Interpolation`.
- **Advanced-ControlNet**: `github.com/Kosinkadink/ComfyUI-Advanced-ControlNet`.

Когда эффект виджета не очевиден из имени, быстрее всего **зафиксировать сид, изменить один виджет крупным шагом, перерендерить на 512×384 / 49 кадров** (§11.9). Большинство виджетов раскрывают своё направление за две итерации.

---

*Это руководство покрывает универсальные строительные блоки. У каждой конкретной модели будут свои loader-ноды, нюансы кондиционирования и оптимальные настройки — но общая структура остаётся той же. §9–§11 демонстрируют, как пайплайн отображается на реальную audio+video модель (LTX-2.3), включая её продвинутые LoRA и keyframe-управления, и её Gemma-based грамматику промптинга.*

Для паттернов сторителлинга и продакшна поверх этого пайплайна (shot lists, репликация стиля, character consistency, рецепты по типам шотов, постпродакшн и более глубокая прокачка промптов) см. сопутствующую серию `animation_course_RU/`.
