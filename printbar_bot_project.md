---
title: Printbar Bot — полная документация проекта
tags: [printbar, instagram-bot, n8n, meta-api, automation]
created: 2026-07-06
status: production
version: 1.0
---

# Printbar Bot — Instagram DM AI-ассистент

> **Статус: 🟢 РАБОТАЕТ В ПРОДАКШЕНЕ** (с 5 июля 2026)
> AI-бот, который автоматически отвечает клиентам в Instagram Direct аккаунта @printbar_baku, знает продукцию и цены Printbar, ведёт диалог на русском и азербайджанском.

---

## 1. Что это и зачем

Printbar.az (PG MMC, VÖEN 1305883031) — мастерская кастомной печати на одежде и сувенирке в Баку. Клиенты массово пишут в Instagram Direct: спрашивают цены, тиражи, сроки, присылают дизайны. Бот заменяет ручные ответы и сторонний сервис SMMbot.net.

**Ключевое отличие от конструкторов типа ManyChat:** бот понимает свободный текст через LLM (Claude), а не работает по жёстким кнопочным сценариям. Ноль абонентки за подписчиков, полный контроль, любые интеграции.

---

## 2. Архитектура (общая картина)

```
Клиент пишет в Instagram Direct (@printbar_baku)
        │
        ▼
Meta Webhook ──POST──► https://bot.printbar.az/webhook/instagram
        │                      (Cloudflare → Caddy → n8n)
        ▼
n8n Workflow:
  Webhook → If (верификация/фильтр) → Respond to Webhook (200 за 2ms)
        │
        ▼
  If1 (анти-эхо фильтр, 3 условия AND)
        │
        ▼
  HTTP Request: подтягивает printbar_bot_brain.md с GitHub (raw)
        │
        ▼
  AI Agent (Claude Haiku 4.5 + Simple Memory по sender.id)
        │
        ▼
  HTTP Request ──POST──► graph.instagram.com/v23.0/me/messages
        │
        ▼
Клиент получает ответ от @printbar_baku
```

**Философия двух слоёв:**
- **MD-слой (промпт, `printbar_bot_brain.md`)** — всё «мягкое»: тон, правила диалога, каталог, примеры. Лежит на GitHub, правится без захода в n8n.
- **Код-слой (n8n workflow)** — всё «жёсткое»: маршрутизация, фильтры, API-вызовы. Где ошибка стоит денег — там код, не промпт.

---

## 3. Инфраструктура и ключевые параметры

### Серверная часть
| Параметр | Значение |
|---|---|
| n8n | v2.27.4, self-hosted |
| URL | `https://bot.printbar.az` |
| Reverse proxy | Caddy |
| CDN/DNS | Cloudflare |
| Webhook URL | `https://bot.printbar.az/webhook/instagram` |

### Meta / Instagram
| Параметр | Значение |
|---|---|
| Meta App (бот) | **Printbar Bot**, App ID `1331543191846410` |
| Meta App (сайт, FB Login) | **Printbar.Az**, App ID `817820293679702` — НЕ трогать, обслуживает OpenCart |
| Бизнес-аккаунт | @printbar_baku, IG ID `17841438924734576` |
| Тест-аккаунт | @turalll, IG ID `4348449221915491` |
| Статус приложения | **Published (Live)** — опубликовано без App Review |
| Разрешения | `instagram_business_basic`, `instagram_business_manage_messages`, `instagram_business_manage_comments` |
| Подписка вебхука | поле `messages` для printbar_baku |
| Privacy Policy | `printbar.az/privacy-policy` (RU/EN/AZ, с разделом 2.4 про Meta-данные) |

### Токен и эндпоинт отправки (КРИТИЧНО — главный урок проекта)
| Параметр | Значение |
|---|---|
| Тип токена | **IGAA** (Instagram User Token), НЕ EAA (Page/User token) |
| Где генерится | Meta Dashboard → Use Cases → Instagram API → «Сгенерировать маркер» для printbar_baku |
| Эндпоинт отправки | `POST https://graph.instagram.com/v23.0/me/messages` |
| Авторизация | Header Auth: `Authorization: Bearer <IGAA-токен>` |
| ❌ НЕ работает | `graph.facebook.com` + EAA-токен → возвращает HTML 400 «Sorry, something went wrong» за 24–37ms |

### Формат webhook payload (актуальный)
```
entry[0].messaging[0].sender.id        ← ID отправителя
entry[0].messaging[0].message.text     ← текст сообщения
entry[0].messaging[0].message.is_echo  ← флаг эха (наши собственные ответы)
```
⚠️ Старый формат `entry[0].changes[0].value.sender.id` в документации/примерах — **устарел**, из-за него recipient.id уходил пустым.

---

## 4. Структура n8n Workflow (нода за нодой)

1. **Webhook (POST)** — принимает события от Meta. Path: `/webhook/instagram`.
2. **Webhook (GET)** — отдельная нода только для верификации Meta (`hub.challenge`). Meta проверяет URL GET-запросом — одна POST-нода не проходит верификацию.
3. **If** — разделяет верификацию и события (`verify_token` OR `body.object = instagram`).
4. **Respond to Webhook** — мгновенный ответ 200 Мете. **Обязательно заголовок `Content-Type: text/plain`** для hub.challenge.
5. **If1 (анти-эхо, 3 условия AND):**
   - `messaging[0].sender.id` **≠** `17841438924734576` — тип **String** (не отвечаем сами себе)
   - `messaging[0].message.is_echo` **≠** `true` — тип **Boolean** (фильтруем эхо наших ответов)
   - `messaging[0].message.text` **не пустой** — тип String (пропускаем реакции/вложения без текста)
   - ⚠️ Типы условий критичны: Boolean вместо String на sender.id ронял все сообщения в false-ветку.
6. **HTTP Request (brain)** — тянет системный промпт с GitHub raw: репозиторий `tural1033/printbar-bot`, файл `printbar_bot_brain.md`. Промпт правится в GitHub без захода в n8n.
7. **AI Agent:**
   - Модель: Anthropic **Claude Haiku 4.5** (было Sonnet 4.6, переехали ради экономии ×5)
   - **Simple Memory**: session key = `sender.id` клиента → у каждого клиента своя память диалога, окно 10 сообщений
   - В промпт агента подставляется текущее время Баку: `{{ $now.setZone('Asia/Baku').toFormat('cccc, dd.MM.yyyy HH:mm') }}` + текст клиента
8. **HTTP Request (отправка ответа):**
   - URL: `https://graph.instagram.com/v23.0/me/messages`
   - Credential: Header Auth (Bearer IGAA). ⚠️ В настройках credential домен `graph.instagram.com` должен быть в allowed domains
   - Body: `recipient.id` = `messaging[0].sender.id`, `message.text` = `{{ JSON.stringify($json.output) }}`

---

## 5. Мозг бота — printbar_bot_brain.md (v2.0)

Живёт на GitHub (`tural1033/printbar-bot`), подтягивается в рантайме. Объём ≈ 9 000–11 000 токенов.

**Раздел 0 (наивысший приоритет) — три критических правила:**
1. **Язык:** никакой кириллицы в азербайджанских/английских ответах (классический баг: «tiraж» вместо «tiraj»). Самопроверка перед отправкой.
2. **Ссылки:** каждая ссылка на отдельной строке, с переносами до и после, без прилепленных эмодзи и знаков препинания — иначе Instagram не делает их кликабельными.
3. **Формат:** никакого markdown в ответах (Instagram его не рендерит).

**Бизнес-правила, зафиксированные в мозге:**
- Тираж 6–19 шт → всегда передаём менеджеру, без оценки цены в чате
- Печать на изделиях клиента: физлицам — нет; бизнесу — от 5 шт
- Проверять график работы по текущему времени, прежде чем звать в офис (был баг: «приходите, мы открыты» в воскресенье)

**Метод развития:** промпт растёт инкрементально — каждый неправильный ответ бота = одно новое правило или один few-shot пример. Раздел с примерами диалогов — самая мощная часть документа.

---

## 6. Экономика (сколько бот кушает)

| Статья | Стоимость |
|---|---|
| Meta API, вебхуки | $0 |
| n8n self-hosted | $0 (свой сервер) |
| LLM (единственная переменная) | см. ниже |

**Оптимизация (сессия 5 июля):**
- Sonnet 4.6 → **Haiku 4.5** (≈ ×5 дешевле, качества хватает)
- **Prompt caching** через HTTP Request ноду (стандартная Anthropic-нода в n8n не умеет `cache_control`):
  - Статичный блок: весь brain.md с `"cache_control": {"type": "ephemeral"}`
  - Динамический блок (дата/время) — ПОСЛЕ точки кеширования
  - Минимум для кеша Haiku — 4 096 токенов; наш brain (9–11k) проходит с запасом
  - Проверка работы кеша: поле `cache_read_input_tokens` в `usage` ответа API (0 при повторных запросах = кеш не бьётся)
- **Ориентиры:** диалог 4–5 сообщений ≈ $0.02–0.03 с кешем (против $0.05–0.06 без). 1000 сообщений/мес ≈ $5–7.
- ⚠️ Включить auto-recharge с лимитом на балансе Anthropic — чтобы бот не умер посреди диалога.

---

## 7. Хронология проекта и уроки (это самое ценное)

### Этап 0 — концепция (≈ 24 июня)
Исследование Replicate-моделей для Printbar Combiner (мокапы), архитектура WhatsApp-бота, создан стартовый шаблон `printbar_bot_brain.md` v0.1. Принцип двух слоёв (MD vs код).

### Этап 1 — вебхук (≈ 29–30 июня)
Проблема: реальные DM не создавали executions, хотя тест-кнопка Meta работала.
Исправлено послойно:
- Добавлена **GET-нода** (Meta верифицирует URL через GET, была только POST)
- `Content-Type: text/plain` в Respond to Webhook
- Workflow нужно **Publish** (в новом UI n8n это заменило Active-тумблер)
- В Meta вписан **production URL**, а не test URL
- **Главное открытие:** в статусе Development Meta доставляет живые messaging-вебхуки ТОЛЬКО от принятых тестировщиков. Тест-кнопка это ограничение обходит — поэтому она работала, а живые сообщения нет.

### Этап 2 — токен-сага и HTML 400 (30 июня – 1 июля)
Нода отправки возвращала **HTML 400** («Sorry, something went wrong») вместо JSON-ошибки, на любых endpoint (graph.facebook.com v21/v23, graph.instagram.com). Быстрый отказ за 24–37ms = отсечка на edge.
- Наняты фрилансеры на Kwork (Darkline_404, затем zmt08) — **не решили**, заказ отправлен на доработку
- Пост на Meta Developer Forum (тред 2540806379761642) — без ответов
- **Урок: HTML 400 от Graph API = неправильный ТИП токена, а не неправильный запрос.**

### Этап 3 — прорыв (5 июля)
Системная диагностика через Graph API Explorer нашла корень:
- ✅ Правильная связка: **`graph.instagram.com` + IGAA-токен + `/v23.0/me/messages`**
- ❌ Была: EAA Page-токен + graph.facebook.com
- Второй фикс: путь `sender.id` — payload приходит в `entry[0].messaging[0]`, а не `entry[0].changes[0].value` (пустой recipient.id)
- Бот впервые ответил живому сообщению ✅

### Этап 4 — публикация и AI-слой (5 июля)
- Приложение **опубликовано** (privacy policy URL, категория, GDPR-контакт, Азербайджан) — **Meta приняла публикацию БЕЗ App Review**. Живые DM от любых людей сразу пошли в n8n.
- Починен If1 (пути + типы условий), собран трёхусловный анти-эхо фильтр
- В credential добавлен `graph.instagram.com` в allowed domains
- Подключён AI Agent + Simple Memory + brain с GitHub
- brain.md переписан до v2.0 (раздел 0, правила языка/ссылок/формата, время Баку)

### Этап 5 — оптимизация затрат (5–6 июля)
Haiku 4.5 + prompt caching (раздел 6).

### Ключевые уроки одним списком
1. Тест-кнопка Meta ≠ реальность: она обходит ограничение Development-режима.
2. Индикаторы в Meta Dashboard врут: зелёная галочка шага 2 не сохраняется после refresh — это UI-баг, не статус.
3. HTML 400 вместо JSON-ошибки = проблема токена/домена, не тела запроса.
4. Формат payload у Meta меняется без предупреждения (`changes[]` → `messaging[]`) — не доверять старым примерам.
5. Типы данных в n8n If-нодах (String vs Boolean) — источник тихих багов.
6. Webhook-верификация Meta требует GET + text/plain.
7. Instagram не кликает ссылки, слепленные с текстом/эмодзи — каждая на своей строке.
8. Публикация приложения может пройти без App Review — стоило попробовать раньше, чем 3 дня дебажить Dev-режим.

---

## 8. Известные проблемы (открытые)

- [ ] Ссылки в одном сообщении пачкой — некликабельны. План: слать ссылки и WhatsApp-номер **отдельными сообщениями** (отдельные HTTP Request ноды).
- [ ] Редкие проскоки кириллицы в AZ-ответах (правило в разделе 0 снизило частоту, следить).
- [ ] Нет og:image на странице конструктора `/ru/create-ru/` (превью ссылок в чате хромает; twitter:image есть, og:image нет).
- [ ] Нестабильные превью картинок у ссылок в Direct.

---

## 9. Роадмап (приоритезированный)

| # | Фича | Суть решения | Оценка |
|---|---|---|---|
| 1 | **Пауза для оператора** | Ловить `is_echo: true` от ручных ответов → ставить флаг паузы на sender.id (12–24ч). Плюс ключевое слово «оператор» → пауза + уведомление в Telegram | 0.5–1 день |
| 2 | **Кнопки и карусели** | Quick replies (до 13 кнопок) и Generic template (карточки: фото + до 3 кнопок) — это тот же Graph API, другое поле `message.attachment` | 1 день |
| 3 | **CRM-запись** | Нода после диалога → Google Sheets / amoCRM: имя, запрос, статус, дата | ~1 час |
| 4 | **Трекинг кликов** | UTM + свой редирект `printbar.az/go/xyz` с логированием | 0.5 дня |
| 5 | **Vision (фото)** | URL картинки из payload → Claude vision → бот понимает присланные логотипы | 1 день |
| 6 | **Голосовые** | URL аудио → Whisper API (~$0.006/мин) → текст в LLM | вечер |
| 7 | **Простой мокап** | rembg на Replicate + программный оверлей (ImageMagick/sharp) на плоский шаблон | 2–4 дня |
| 8 | **Красивый мокап (Combiner)** | Генеративные модели Replicate (SDXL+ControlNet и т.п.), ~$0.01–0.05/генерация. AI — для промо-мокапов; точный оверлей — для производственного пруфа (AI искажает буквы/лого!) | 1–2 недели |
| 9 | **Оплата в чате** | API Epoint / Payriff / Kapital Bank → платёжная ссылка прямо в Direct | 2–3 дня |
| 10 | **PDF/AI-файлы** | Только через WhatsApp (Instagram DM не пропускает произвольные файлы). PDF → рендер 1-й страницы → пайплайн мокапа | после WhatsApp-бота |

**Конечная цель:** AI-продавец — клиент прислал логотип голосовым «хочу на 20 футболок», получил мокап, цену и ссылку на оплату за минуту, ночью, без человека.

---

## 10. Смежные проекты и контекст

- **Printbar Combiner** — image-пайплайн мокапов на Replicate (vision, rembg, upscale, ControlNet, сегментация, NSFW-фильтр). Задуман как независимый актив, не зависящий от роадмапа Meta.
- **WhatsApp-бот** — следующая платформа; архитектура переносится почти 1:1. Зарезервировать username `printbar_baku` (Wave 1 для Азербайджана).
- **Сайт printbar.az** — OpenCart; Facebook Login чинился отдельно (deprecated scope `user_about_me` в модуле Social Login), это App ID 817820293679702.

---

## 11. Глоссарий / быстрые ссылки

| Что | Где |
|---|---|
| Meta Dashboard (бот) | `developers.facebook.com/apps/1331543191846410` |
| Настройка Instagram API | Dashboard → Use Cases → Customize → API Setup |
| Graph API Explorer | `developers.facebook.com/tools/explorer` |
| n8n | `bot.printbar.az` |
| Мозг бота | GitHub `tural1033/printbar-bot` → `printbar_bot_brain.md` |
| Webhook | `https://bot.printbar.az/webhook/instagram` |
| Endpoint отправки | `POST https://graph.instagram.com/v23.0/me/messages` |

---

*Документ собран 06.07.2026 из истории разработки. При изменениях архитектуры — обновлять разделы 3, 4 и 8.*
