# InkStory Bot

Telegram-бот для организации литературного конкурса на [inkstory.net](https://inkstory.net).  
Жюри проверяют посты участников — считают слова и ошибки. Бот автоматически собирает посты, распределяет их между жюри и ведёт статистику.

---

## Возможности

- **Два режима очереди** — общая (первый пришёл — первый взял) или распределение по наименьшей загрузке
- **Автопарсер** — собирает новые посты по расписанию и уведомляет жюри
- **AI-проверка** — отправка текста в Groq API для поиска ошибок
- **Просроченные посты** — если жюри не проверил за `EXPIRE_MINUTES` минут, пост автоматически освобождается
- **Админ-панель** — статистика, очереди, верификация жюри, смена режима, логи
- **Экспорт в Excel** — итоги конкурса по дням с рейтингом участников
- **CLI** — консольное управление БД

---

## Установка

```bash
# 1. Клонируй репозиторий
git clone https://github.com/your/inkstory-bot.git
cd inkstory-bot

# 2. Создай виртуальное окружение
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. Установи зависимости
pip install -r requirements.txt

# 4. Создай .env из примера
cp .env.example .env
# Заполни BOT_TOKEN и при необходимости GROQ_API_KEY

# 5. Инициализируй БД
python scripts/init_db.py

# 6. Запусти бота
python bot.py
```

---

## Миграция со старой версии

```bash
# Создай новую пустую БД
python scripts/init_db.py --path data/new.db

# Перенеси данные
python scripts/migrate.py --old data/old.db --new data/new.db

# Замени старую БД
mv data/new.db data/main.db
```

---

## Режимы очереди

Переключаются через переменную `QUEUE_MODE` в `.env`:

| Режим | Описание |
|---|---|
| `distributed` | Пост сразу назначается жюри с наименьшей очередью |
| `open` | Все посты в общей очереди, первый кто нажмёт /next — тот и берёт |

В обоих режимах: если жюри взял пост но не проверил за `EXPIRE_MINUTES` минут — пост освобождается.

---

## Структура проекта

```
├── bot.py               — запуск бота
├── main.py              — запуск парсера вручную
├── bot/
│   ├── handlers/
│   │   ├── user.py      — команды жюри (/next, /register, /stats...)
│   │   └── admin.py     — команды администратора (/admin)
│   └── keyboards.py     — inline клавиатуры
├── parser/
│   ├── links.py         — сбор ссылок через API
│   ├── posts.py         — парсинг постов
│   └── queue_manager.py — управление очередью
├── utils/
│   ├── database.py      — подключение к БД
│   ├── config.py        — все настройки из .env
│   ├── logger.py        — логгер
│   ├── word_counter.py  — подсчёт слов
│   └── ai_utils.py      — проверка через Groq API
├── scripts/
│   ├── init_db.py            — инициализация БД
│   ├── migrate.py            — миграция со старой схемы
│   ├── export_results.py     — экспорт в Excel
│   └── fix_pending_queue.py  — добавляет pending-посты без очереди
├── tests/               — тесты
├── data/                — БД (в .gitignore)
├── logs/                — логи (в .gitignore)
└── results/             — экспорты Excel (в .gitignore)
```

---

## Схема БД

| Таблица | Описание |
|---|---|
| `reviewers` | Жюри (TGID, верификация, права админа) |
| `authors` | Авторы постов |
| `days` | Дни конкурса |
| `links` | Ссылки на посты (Parsed 0/1) |
| `blacklist` | Заблокированные ссылки |
| `posts_info` | Посты со статусом (pending/checking/done/rejected/reviewer_post) |
| `queue` | Очередь — кто за каким постом, когда взял |
| `results` | Результаты проверки (слова, ошибки, reviewer) |

> Все настройки (режим очереди, таймауты и т.д.) хранятся в `.env`, не в БД.

---

## Утилиты

### fix_pending_queue.py

Если посты попали в `posts_info` со статусом `pending`, но запись в `queue` для них не создалась — этот скрипт исправляет ситуацию с учётом текущего `QUEUE_MODE`.

```bash
# Посмотреть что будет сделано (БД не трогает)
python scripts/fix_pending_queue.py --dry-run

# Применить
python scripts/fix_pending_queue.py

# Указать другую БД
python scripts/fix_pending_queue.py --db data/backup.db
```

---

## Тесты

```bash
python tests/test_word_counter.py
python tests/test_database.py
python tests/test_queue_manager.py
```

---

## Переменные окружения

| Переменная | По умолчанию | Описание |
|---|---|---|
| `BOT_TOKEN` | — | Токен Telegram бота от @BotFather (**обязательно**) |
| `PARSER_INTERVAL` | `30` | Интервал автопарсера в минутах |
| `PAGE_SIZE` | `20` | Постов на страницу при обходе API |
| `PAGE_PAUSE_LINKS` | `1.0` | Пауза между запросами при сборе ссылок (сек) |
| `PAGE_PAUSE_POSTS` | `1.5` | Пауза между запросами при парсинге постов (сек) |
| `QUEUE_MODE` | `distributed` | Режим очереди: `distributed` или `open` |
| `EXPIRE_MINUTES` | `30` | Через сколько минут незавершённый пост возвращается в очередь |
| `EXPIRE_CHECK_INTERVAL` | `5` | Как часто проверять просроченные посты (минуты) |
| `FINAL_PARSER_HOUR` | `23` | Час финального парсинга (по Москве) |
| `FINAL_PARSER_MINUTE` | `55` | Минута финального парсинга |
| `NEW_DAY_HOUR` | `0` | Час смены дня (по Москве) |
| `NEW_DAY_MINUTE` | `1` | Минута смены дня |
| `MAX_WORDS` | `100000` | Максимум слов при вводе жюри |
| `MAX_ERRORS` | `10000` | Максимум ошибок при вводе жюри |
| `GROQ_API_KEY` | — | API-ключ Groq для AI-проверки (console.groq.com) |
| `GROQ_MODEL` | `llama-3.3-70b-versatile` | Модель Groq |
| `AI_CHUNK_SIZE` | `3000` | Максимум символов на один запрос к AI |