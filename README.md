# 🤖 MerchHelperBot

> Telegram-юзербот для автоматических ответов мерчантам в техническом чате во время интеграции.

---

## 📋 Проблематика

При интеграции мерчанты часто задают в техническом чате вопросы, ответы на которые уже есть в публичной документации. Дополнительно периодически возникают вопросы о бизнес-логике каскада — и в обоих случаях поддержка тратит время на ответы, которые можно автоматизировать.

**Ключевые боли:**
- Повторяющиеся вопросы по интеграции, которые задокументированы, но мерчанты не всегда их находят самостоятельно
- Нагрузка на поддержку по стандартным вопросам
- Вопросы по бизнес-логике каскада, которые требуют экспертных знаний, но встречаются регулярно

---

## 💡 Решение

**MerchHelperBot** — Telegram-юзербот на базе LLM, который:

1. **Приглашается в технические чаты** с мерчантами как обычный участник
2. **Реагирует только на упоминания** (`@username`) — не мешает органическому общению
3. **Отвечает на вопросы по документации** — с помощью RAG (Retrieval-Augmented Generation) бот ищет релевантный контекст в базе знаний и формирует точный ответ
4. **Покрывает вопросы по каскаду** — бизнес-логика каскада описана в отдельных документах, доступных боту
5. **Кэширует частые вопросы** — семантический кэш (embedding-based) позволяет переиспользовать ответы на похожие вопросы, сокращая задержку и стоимость API-вызовов

---

## 🏗️ Архитектура

```
┌─────────────────────────────────────────────────────┐
│                  Telegram Chat                       │
│   Merchant: "@helperbot как подключить webhook?"    │
└──────────────────────┬──────────────────────────────┘
                       │ mention event
                       ▼
┌─────────────────────────────────────────────────────┐
│                  Userbot (Telethon)                  │
│              bot/handlers.py                         │
└──────────────────────┬──────────────────────────────┘
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
┌─────────────────┐      ┌──────────────────────┐
│   FAQ Cache     │      │    AI Agent (RAG)     │
│  (SQLite +      │      │  ai/agent.py          │
│  embeddings)    │      │                       │
│  cache/store.py │      │  1. Embed question    │
│                 │      │  2. Find docs chunks  │
│  Hit → return   │      │  3. Call OpenAI LLM   │
│  cached answer  │      │  4. Return answer     │
└─────────────────┘      └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │    Knowledge Base     │
                         │    docs/              │
                         │  ├── integration.md  │
                         │  └── cascade.md      │
                         └──────────────────────┘
```

### Компоненты

| Модуль | Описание |
|---|---|
| `bot/handlers.py` | Telethon-хэндлеры: слушает упоминания, управляет диалогом |
| `ai/agent.py` | RAG-агент: разбивает документы на чанки, ищет релевантный контекст, генерирует ответ через OpenAI |
| `cache/store.py` | Семантический кэш: хранит вопрос → embedding → ответ в SQLite; при cosine similarity > порога возвращает кэшированный ответ |
| `config.py` | Централизованная конфигурация через `.env` |
| `docs/` | Markdown-файлы с документацией (интеграция, каскад и т.д.) |
| `setup_session.py` | Утилита для первичной авторизации и получения session string |
| `main.py` | Точка входа |

---

## ⚙️ Как работает кэш

Для снижения нагрузки на OpenAI API реализован **семантический FAQ-кэш**:

1. При поступлении вопроса вычисляется его **embedding** (`text-embedding-3-small`)
2. Вычисляется **cosine similarity** с эмбеддингами всех кэшированных вопросов
3. Если схожесть превышает порог (`CACHE_SIMILARITY_THRESHOLD`, по умолчанию `0.92`), возвращается сохранённый ответ **без вызова LLM**
4. Новый вопрос + ответ сохраняются в кэш с TTL (`CACHE_TTL_HOURS`)
5. При превышении лимита (`CACHE_MAX_ENTRIES`) вытесняются самые старые записи

---

## 🚀 Быстрый старт

### 1. Клонировать репозиторий

```bash
git clone https://github.com/iLnamzdat/merchHelperBot.git
cd merchHelperBot
```

### 2. Установить зависимости

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Настроить переменные окружения

```bash
cp .env.example .env
```

Заполнить `.env`:

| Переменная | Где получить | Описание |
|---|---|---|
| `TELEGRAM_API_ID` | [my.telegram.org](https://my.telegram.org) → API development tools | MTProto API ID |
| `TELEGRAM_API_HASH` | [my.telegram.org](https://my.telegram.org) | MTProto API Hash |
| `TELEGRAM_SESSION_STRING` | `python setup_session.py` | Сериализованная сессия |
| `OPENAI_API_KEY` | [platform.openai.com](https://platform.openai.com) | Ключ OpenAI API |

### 4. Получить session string

```bash
python setup_session.py
```

Скрипт запросит номер телефона и код подтверждения, после чего выведет `SESSION_STRING` — его нужно вставить в `.env`.

### 5. Наполнить базу знаний

Добавить/отредактировать Markdown-файлы в папке `docs/`:

```
docs/
├── integration.md   # Инструкция по интеграции (webhook, API-ключи, форматы и т.д.)
└── cascade.md       # Бизнес-логика каскада
```

### 6. Запустить бота

```bash
python main.py
```

Пригласить аккаунт в нужные Telegram-чаты. Мерчанты смогут задавать вопросы, упоминая юзербота.

---

## 🔧 Конфигурация

Все параметры задаются через `.env` (см. `.env.example`):

```env
# Telegram
TELEGRAM_API_ID=...
TELEGRAM_API_HASH=...
TELEGRAM_SESSION_STRING=...

# OpenAI
OPENAI_API_KEY=...
OPENAI_MODEL=gpt-4o-mini
OPENAI_EMBEDDING_MODEL=text-embedding-3-small

# FAQ-кэш
CACHE_SIMILARITY_THRESHOLD=0.92   # порог схожести для попадания в кэш
CACHE_TTL_HOURS=24                # время жизни записи в кэше (0 = бессрочно)
CACHE_MAX_ENTRIES=500             # максимальное число записей

# Документация
DOCS_DIR=docs
MAX_CONTEXT_CHARS=12000
```

---

## 🗂️ Структура проекта

```
merchHelperBot/
├── main.py                # Точка входа
├── config.py              # Конфигурация (env)
├── setup_session.py       # Утилита первичной авторизации
├── requirements.txt
├── .env.example
├── bot/
│   ├── __init__.py
│   └── handlers.py        # Telethon event handlers
├── ai/
│   ├── __init__.py
│   └── agent.py           # RAG-агент (документы → LLM → ответ)
├── cache/
│   ├── __init__.py
│   └── store.py           # Семантический FAQ-кэш (SQLite)
└── docs/
    ├── integration.md     # Документация по интеграции
    └── cascade.md         # Документация по каскаду
```

---

## 🛡️ Безопасность

- `.env` добавлен в `.gitignore` — **никогда не коммитить** реальные ключи
- `*.session` файлы также в `.gitignore`
- Session string даёт полный доступ к Telegram-аккаунту — хранить безопасно (например, в секретах CI/CD или vault)

---

## 📝 Лицензия

MIT
