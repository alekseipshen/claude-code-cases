# Skill: генерация и локализация видео-рекламы AI-агентом

> Это инструкция-скилл. Скормите её своему AI-агенту (Claude Code / любой coding-agent) как контекст — и он сможет собрать или локализовать рекламный ролик без монтажёра, диктора и студии. Технические термины оставлены на английском намеренно.

## Что делает этот скилл
Две задачи:
- **A. Ролик с нуля** — из сценария собрать вертикальный рекламный ролик (B-roll или talking-head) с озвучкой, музыкой и субтитрами.
- **B. Локализация** — взять готовый ролик и пересобрать его на другом языке с переозвучкой и синхронизацией губ (lipsync).

## Стек и роль каждого инструмента

| Инструмент | Роль |
|---|---|
| **Varg SDK** (`vargai`, Bun-рантайм) | Оркестратор. JSX/React-подход: сцены описываются декларативно (`<Render>`, `<Clip>`, `<Image>`, `<Video>`, `<Music>`, `<Speech>`, `<Captions>`), SDK сам зовёт провайдеров, кэширует и собирает финал через встроенный FFmpeg-композитор. Auto-ducking музыки под голос, transitions, word-by-word captions. |
| **fal.ai** | Генерация картинок и видео + lipsync. Image: `flux-pro` (финал), `flux-schnell` (тест), `nano-banana-pro` (консистентность персонажа/продукта). Video (image-to-video): `kling-v2.5` / `kling-v3` (финал), `wan-2.5` (драфт). Lipsync: endpoint `fal-ai/sync-lipsync/v2`, модель `lipsync-2` или `lipsync-2-pro` (движок sync.so), редактирует только область рта. |
| **ElevenLabs** | Голос, музыка, дубляж. TTS-модели `eleven_multilingual_v2` (спокойная наррация) и `eleven_v3` (выразительнее). **Dubbing v2** (`POST /v1/dubbing`) — переозвучка готового ролика на 90+ языках с сохранением голоса спикера. Музыка — генерация по текстовому промпту. |
| **Groq** | Транскрипция для субтитров (бесплатно). Whisper `whisper-large-v3`, `response_format=verbose_json`, word-level timestamps. |
| **FFmpeg / ffprobe + ASS + PIL** | Низкоуровневая сборка вне Varg (особенно в локализации): probe структуры, нарезка, перекодирование HEVC→H264, concat, overlay плашек, прожиг ASS-субтитров, remux аудио. |

## Где здесь Heygen
Heygen в этом пайплайне **не используется**, но логично закрывает соседнюю нишу: **генерация talking-head аватара с нуля**, когда нет реального видео спикера. В нашем стеке тот же результат достигается иначе:
- **Talking-head из портрета:** `fal kling` анимирует мимику портрета → `fal sync-lipsync` синхронизирует губы под озвучку ElevenLabs.
- **Перевод реального видео спикера:** ElevenLabs Dubbing v2 (голос) → fal sync-lipsync (губы).

Если нужен именно синтетический ведущий-аватар «из текста» — это территория Heygen / Higgsfield, и его выход можно вставлять как ещё один `<Video>`-клип в Varg-сцену или накладывать поверх B-roll через FFmpeg.

## Что нужно (аккаунты и env-переменные — только имена, без значений)
- `FAL_API_KEY` (он же `FAL_KEY`) — fal.ai: image, video, lipsync
- `ELEVENLABS_API_KEY` — голос, музыка, Dubbing v2
- `GROQ_API_KEY` — Whisper-транскрипция субтитров
- `GOOGLE_GENERATIVE_AI_API_KEY` — опционально, если используете Google-видеомодели через Varg (`veo-3.1`/`veo-2`)
- (Heygen-ключ — нужен только если добавляете аватары Heygen)

Ключи держать в `.env` проекта (gitignored). Рантайм — **Bun**.

---

## Workflow A — ролик с нуля

1. **Сценарий.** Распишите сцены: текст озвучки + визуальное описание каждого кадра.
2. **Озвучка отдельно** (не через `<Speech>`, а прямым REST к ElevenLabs — потому что `<Speech>` не умеет `speed` и кастомные voice ID): TTS → `media/voiceover.mp3`. Затем Groq Whisper транскрибирует mp3 → слова с таймкодами → чистый `media/voiceover.srt`.
3. **Проверка длительности:** голос должен быть на **1–3 сек короче** суммы длительностей клипов (иначе обрезка/тишина).
4. **JSX-рендер** (`.tsx`): `<Render width={1080} height={1920} fps={30} cache=".cache/ai">` → набор `<Clip duration transition>`, в каждом `<Video prompt={{text, images:[img]}} model={fal.videoModel("kling-v2.5")}>`, где `img = Image({prompt, model: fal.imageModel("flux-pro"), aspectRatio:"9:16"})`. Музыка `<Music prompt=... ducking>`, голос подключается как `<Music src="media/voiceover.mp3">`, субтитры `<Captions srt="media/voiceover.srt" style="tiktok">`.
5. **Первый рендер:** fal генерит картинки → анимирует в видео → ElevenLabs музыка → всё в `.cache/ai` → FFmpeg собирает `output/*.mp4`.
6. **Итерации:** меняете что нужно — закэшированные ассеты переиспользуются, повторный платный re-gen не происходит. Драфты на `wan-2.5`, финал — переключить на `kling-v3`.

**Talking-head вариант:** портрет → `kling-v3` анимирует мимику → `<Speech>` (ElevenLabs) → `fal.videoModel("sync-v2-pro")` с `prompt={{video, audio}}` синхронизирует губы. Каждый клип рендерить **отдельным** `render()` (sync-v2-pro ломает multi-clip concat), потом склейка FFmpeg.

## Workflow B — локализация (дубляж + lipsync)

1. **ElevenLabs Dubbing v2** (в веб-приложении или API): исходный ролик → задублированный MP4 на целевом языке (голос сохранён). Внимание: экспорт может содержать **вшитые** субтитры исходного языка.
2. **Probe** (ffprobe): понять структуру; найти `FACE_START` (момент появления лица), замерить полосу вшитых субтитров.
3. **Нарезка сегментов** (FFmpeg): `talk_vid.mp4` (сегмент с лицом, **обязательно H264/yuv420p — sync.so не принимает HEVC**, без аудио), `talk_aud.m4a` (аудио сегмента → в lipsync), `hook.mp4` (без-лицевой кусок остаётся как есть), `full_audio.m4a` (всё целевое аудио для финального remux).
4. **Lipsync** (`fal-ai/sync-lipsync/v2`): загрузить video+audio в `fal.storage`, `fal.subscribe(..., {video_url, audio_url, model:"lipsync-2-pro", sync_mode:"cut_off"})`. **Сначала 5-сек тест**, потом полный сегмент.
5. **Субтитры целевого языка** (Groq Whisper на `full_audio`, `language=<код>`, word-timestamps) → чанкинг в строки (≤26 симв / ≤5 слов) + кламп overlap → `cap.srt`.
6. **Плашка** (PIL, только если в источнике вшиты субтитры): непрозрачный rounded-rect PNG под полосу субтитров (translucent/blur протекают — нужен opaque).
7. **ASS-субтитры:** `PlayResX/PlayResY` **обязательно** = размеру видео (1080×1920); центрирование `{\an5\pos(cx,cy)}`, не MarginV.
8. **Сборка** (FFmpeg): concat hook+lipsynced (нормализовать SAR/fps), overlay плашек с time-gated `enable`, прожиг `ass=combined.ass`, remux полного целевого аудио → финал.

---

## Ключевые правила и готчи
1. **`cache: ".cache/ai"` обязательно** в `render()` — дефолтный кэш только in-memory, теряется между запусками → каждый ран жжёт кредиты заново. **Никогда не удаляйте `.cache/ai/`** (там оплаченные генерации, $0.50–2.00/клип).
2. **Озвучка отдельно от Varg** — `<Speech>` не поддерживает `speed`/кастомный voice ID; генерируйте REST-ом, подключайте как `<Music src>` + `<Captions srt>`.
3. **Кламп overlap в SRT** — Whisper отдаёт пересекающиеся таймкоды слов → субтитры стакаются. Правило: `end = min(word.end, nextStart)`, минимум 50 ms показа.
4. **ElevenLabs Music — без имён реальных людей** в промпте («в стиле X» = отказ по ToS). Описывайте жанр обобщённо.
5. **Cinematic image prompts** — добавляйте «camera anchors» (35mm film, anamorphic lens, shallow DoF, film grain), избегайте крупных лиц/рук (силуэты/wide shots) — анти-«AI look».
6. **Draft→Production** — итерации на `wan-2.5` (~$0.15), финал один раз на `kling-v3` (~$0.50): экономия ~70%.
7. **HEVC→H264** перед отправкой в fal/sync.so (`-c:v libx264 -pix_fmt yuv420p`).
8. **`-ss` до `-i` сбрасывает PTS** → timed ASS-субтитры не рендерятся при извлечении превью-кадров; ставьте `-ss` после `-i` или `-copyts`.
9. **Lipsync трогает только рот** — графику/оверлеи над ртом не нужно вырезать, держите сегмент лица целым.

## Команды и структура
```bash
cd <project>            # Bun: export PATH="$HOME/.bun/bin:$PATH"
bun run generate-voiceover.ts                 # озвучка + SRT (ElevenLabs + Groq)
bun run my-reel.tsx                            # рендер ролика
bun run lipsync.ts <video> <audio> <out.mp4> lipsync-2-pro   # standalone lipsync (нужен @fal-ai/client)
```
`tsconfig.json` ключевое: `"jsx": "react-jsx"`, **`"jsxImportSource": "vargai"`**, `moduleResolution: "bundler"`.
```
project/
├── *.tsx                 # рендер-сцены в JSX
├── generate-voiceover.ts # ElevenLabs TTS + Groq SRT
├── lipsync.ts            # standalone fal sync-lipsync
├── tsconfig.json
├── .env                  # ключи (gitignored)
├── media/                # voiceover.mp3/.srt, портреты
├── output/               # финалы .mp4
└── .cache/ai/            # дисковый кэш платных генераций — НЕ удалять
```
Доп. для локализации: `ffmpeg`, `ffprobe`, `python3` + `Pillow (PIL)`, шрифты Liberation/DejaVu Sans Bold.
