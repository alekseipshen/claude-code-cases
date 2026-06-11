# Skill: SEO-конвейер программных страниц AI-агентом

> Это инструкция-скилл. Скормите её своему AI-агенту как контекст — и он сможет собрать конвейер генерации сотен/тысяч уникальных локальных SEO-страниц (`город × услуга × бренд`) с автоматическим контролем качества, публикацией и проверкой индексации. Технические термины — на английском намеренно.

## Что делает этот скилл
Фабрика контента для программного SEO: на каждое сочетание `город × услуга` (опционально `× бренд`) генерируется уникальная страница, проходит **17 авто-проверок качества**, публикуется на сайт и форсится в индекс Google. Цель — закрыть long-tail и «near me» запросы динамическими локальными посадочными страницами.

## Архитектура (5 шагов)
```
[enrich_cities] → [generate_content] → [quality_check] → [publish_content] → [check_indexing + submit_to_indexing_api]
  Census+Perplexity   Claude (CLI/API)    17 авто-проверок    YAML→git→deploy     GSC URL Inspection + Google Indexing API
```
Хранилище — PostgreSQL-схема с таблицами `city_profiles`, `page_content`, `content_quality_log`. Контент разных сайтов разделён колонкой `site`. Статусная машина страницы: `draft → reviewed → published` (+ `rejected` на регенерацию).

## Что нужно (аккаунты и env-переменные — только имена)
- **Anthropic**: подписка Claude (OAuth, для CLI `claude -p` = $0) **или** `ANTHROPIC_API_KEY` (API-режим)
- `CENSUS_API_KEY` — US Census API (с ~2026 ключ обязателен)
- `PERPLEXITY_API_KEY` — качественные локальные данные
- PostgreSQL connection string (**в env, не в коде**)
- Google Search Console OAuth token (refresh-token, JSON) — для URL Inspection
- Google Service Account (JSON, scope Indexing API; SA добавлен владельцем property) — для submit
- GitHub repo сайта + webhook на CI/CD; деплой-платформа (напр. Vercel)

---

## Шаги детально

**1. enrich_cities.py** — наполняет `city_profiles`:
- Парсит список городов из исходника сайта.
- **US Census API** (ACS 5-year): население (`B01003_001E`) и median income (`B19013_001E`) по штатам (через FIPS). Матчинг Census→город: точное имя → slug → подстрока. UPSERT (COALESCE сохраняет существующее).
- **Perplexity API**: районы, ориентиры, тип застройки, локальный характер. Батчами по 15 городов на вызов (чтобы JSON не ломался).

**2. generate_content.py** — генерация через Claude:
- Два режима: **CLI** (`claude -p` по подписке, $0 — основной для nightly) и API (`anthropic.Anthropic`).
- `max_tokens=4000`, `temperature=0.7`. System-промпт per-site, user-промпт общий (`city_service`).
- Авто-скип уже сгенерированного. Режим **improve**: берёт `rejected`, перегенерирует, инкрементит `improve_attempts` (с лимитом попыток).

**3. quality_check.py** — 17 авто-проверок (ниже). Проходные → `reviewed`.

**4. publish_content.py** — `reviewed` → YAML-файлы в `content/` → git → деплой → `published`.

**5. Индексация** — `check_indexing.py` (проверка) + `submit_to_indexing_api.py` (форс).

---

## 17 проверок качества (CHECK_WEIGHTS)
Итоговый `score = (Σ(score_i × weight_i) / Σ weight_i) × 100` (0–100).

| # | Проверка | Вес | Назначение |
|---|----------|-----|------------|
| 1 | `yaml_valid` | **1.0** | must-pass — валидный YAML, есть `intro:` |
| 2 | `word_count` | 0.8 | объём в пределах фазы (1500–2000 слов) |
| 3 | `city_mentions` | 0.7 | город упомянут ≥ 3 раз |
| 4 | `banned_phrases` | **1.0** | critical — 0 AI-филлерных фраз |
| 5 | `local_details` | 0.8 | локальные детали (районы / zip / тип жилья) |
| 6 | `similarity` | 0.9 | уникальность intro внутри сервиса |
| 7 | `readability` | 0.6 | читабельность |
| 8 | `burstiness` | 0.7 | AI-детект: вариативность длины предложений (CV ≥ 0.30) |
| 9 | `lexical_diversity` | 0.7 | AI-детект: лексическое разнообразие |
| 10 | `keyword_density` | 0.6 | плотность city+service без переспама |
| 11 | `sentence_starters` | 0.5 | разнообразие начал предложений |
| 12 | `paragraph_variance` | 0.4 | вариативность длины абзацев |
| 13 | `expertise_signals` | 0.9 | E-E-A-T: бренды + технические термины |
| 14 | `cta_presence` | 0.8 | наличие CTA |
| 15 | `body_similarity` | 0.9 | **doorway-детект**: 4-gram Jaccard по всему телу |
| 16 | `title_uniqueness` | 0.8 | штраф за дублирующиеся title |
| 17 | `local_depth` | 0.7 | качество `localContext`, а не просто наличие |

**Порог публикации:** страница проходит, если (а) **все 6 критических** проверок прошли — `yaml_valid, word_count, city_mentions, banned_phrases, local_details, similarity` (hard-fail) — **И** (б) `score ≥ 85`. Остальные 11 — только штраф к score.

## Модели и фазы
- Sonnet `claude-sonnet-4-6`, Haiku `claude-haiku-4-5`. В CLI-режиме — алиас `--model sonnet` (НЕ `--bare`, иначе отключается OAuth-подписка).
- Логика фаз (приоритет покрытия): Phase 1 = топ-50 городов × топ-4 сервиса (~200 стр., высшее качество) → Phase 2 = топ-150 × все (~1500 стр.) → Phase 3 = все × все (~4200 стр.). Каждая фаза авто-скипает готовое.
- Себестоимость: ~$0.05/страница на Sonnet через API (~1800 слов); **через CLI `claude -p` по подписке = $0** (основной режим, nightly cron).

## Публикация
- Только `reviewed` → YAML. Путь: `content/cities/{city_slug}/services/{service}.yaml` (варианты для brand-страниц). Поля: `intro`, `localContext`, `commonProblems[]`, `faq[]`, `title`, `whyChooseUs`.
- Git: `add` → `commit` → **`git pull --rebase`** (защита от параллельных деплоев несколькими агентами) → `push` (3 ретрая) → статус `published`, `published_at=NOW()`.
- Деплой-чейн: push → webhook → CI → билд. На Next.js: YAML читается через `fs+js-yaml`, рендерится компонентом внизу страницы. **Критично:** `outputFileTracingIncludes` должен включать `./content/**/*.yaml`, иначе serverless не зальёт файлы.

## Индексация
- **check_indexing.py** — GSC URL Inspection API (`/v1/urlInspection/index:inspect`). Лимит **2000 инспекций/день на property**. OAuth-токен с авто-рефрешем внутри батча. Проверяет страницы старше `min_age_days`, на 429 останавливает сайт, пишет verdict (`indexed/not_indexed/partial`).
- **submit_to_indexing_api.py** — Google Indexing API (`/v3/urlNotifications:publish`, `URL_UPDATED`), service-account auth. Квота **200 URL/день на проект** (напр. 6 сайтов × 30). Ускоряет индексацию молодых доменов с малым crawl budget.

---

## Ключевые готчи
1. **`claude -p` ДОЛЖЕН снимать API-ключи из env**, иначе CLI молча биллит по API вместо подписки (реальный инцидент — ~$70/ночь). Перед `subprocess.run`:
   ```python
   env = os.environ.copy()
   env.pop("CLAUDECODE", None)          # иначе nested-session error
   env.pop("ANTHROPIC_API_KEY", None)   # форсит OAuth-подписку
   env.pop("ANTHROPIC_AUTH_TOKEN", None)
   ```
   Плюс: `cwd="/tmp"` (пропустить авто-загрузку проектного CLAUDE.md, который может конфликтовать с system-промптом генерации), `--system-prompt` отдельным флагом, `timeout=600`.
2. **Banned phrases** — 6 категорий AI-филлера (filler / buzzwords / transitions / manipulation / faux-expertise / weak-phrasing). Примеры: «plays a crucial role», «a wide range of», «a plethora of», «tailored to your», «satisfaction guaranteed», «we have the expertise», «no matter the issue, big or small». Любое вхождение = критический фейл.
3. **Doorway-детект** (`body_similarity`): 4-gram Jaccard по полному телу страниц внутри одного сервиса. **>30% = doorway risk** (Google штрафует near-duplicate city-страницы), >20% = warning.
4. **Очистка CLI-вывода**: срезать преамбулу до первого YAML-ключа, markdown-фенсы, footer-строки, trailing `---` и бэктики.
5. **QA только `draft`** — `published` не перепроверяется (уже задеплоен).
6. **Census-ключ обязателен** (~2026) — без него enrichment молча скипается.
7. **Паузы сайтов** — исключайте из autonomous batch сайты с 0% индексации (расследовать sitemap/robots/deploy), ручной прогон по сайту всё равно работает.

## Анти-thin-content дисциплина
Не гнать максимум страниц сразу. Если контент тонкий или похожий — Google наказывает как doorway/thin content. Ограничивайте охват приоритетными городами × сервисами, держите `body_similarity` низким, наполняйте реальной локалью (районы, ZIP, ориентиры), и форсите индексацию батчами в пределах квот.
