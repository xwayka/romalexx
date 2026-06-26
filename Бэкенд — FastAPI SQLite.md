---
date: 2026-06-06
type: technical
project: ROMALEX
tags: [ROMALEX, fastapi, sqlite, jwt, python, backend]
---

# Бэкенд — FastAPI SQLite

[[ROMALEX]] | [[ROMALEX/Таблица учёта по проектам]]

---

## Стек

- **FastAPI** — веб-фреймворк, REST API
- **SQLite** — встроенная БД, файл `romalex.db`
- **JWT** — авторизация, библиотека `python-jose`
- **httpx** — HTTP-клиент для запросов к Ollama
- **pydantic** — валидация данных (встроен в FastAPI)
- **python-dotenv** — загрузка `.env`

---

## Структура файлов

```
backend/
├── main.py          ← FastAPI-приложение, CORS, роуты, статика
├── database.py      ← SQLite init, get_db dependency
├── models.py        ← Pydantic-модели запросов/ответов
├── auth.py          ← JWT login/verify, хеширование паролей
├── seed.py          ← засев 53 позиций (выполнен один раз)
├── .env             ← ключи, пароли (не в git)
└── romalex.db       ← SQLite база данных
```

---

## Схема базы данных

### Таблица `items` (позиции)

| Поле | Тип | Описание |
|---|---|---|
| id | INTEGER PK | Авто-ID |
| tab | TEXT | Вкладка: tube / insulation / automation / electrical / drainage / consumable |
| name | TEXT | Наименование позиции |
| unit | TEXT | Единица измерения |
| project_qty | REAL | Проектное количество (из PDF) |
| estimate_qty | REAL | Сметное количество |

### Таблица `operations` (операции)

| Поле | Тип | Описание |
|---|---|---|
| id | INTEGER PK | Авто-ID |
| item_id | INTEGER FK | Ссылка на items.id |
| warehouse | TEXT | Склад: kirzhach / gsk2 / gsk3 |
| op_type | TEXT | Тип: prikhod / rashod |
| amount | REAL | Количество |
| date | TEXT | Дата операции (ISO) |
| supplier | TEXT | Поставщик / подрядчик (опционально) |
| invoice | TEXT | Номер накладной (опционально) |

### Таблица `chats` (чаты ИИ-аналитики)

| Поле | Тип | Описание |
|---|---|---|
| id | INTEGER PK | Авто-ID |
| username | TEXT | Логин пользователя |
| name | TEXT | Название чата |
| created_at | TEXT | Дата создания (ISO) |

### Таблица `chat_messages` (сообщения чатов)

| Поле | Тип | Описание |
|---|---|---|
| id | INTEGER PK | Авто-ID |
| chat_id | INTEGER FK | Ссылка на chats.id (ON DELETE CASCADE) |
| role | TEXT | user / assistant |
| content | TEXT | Текст сообщения |
| created_at | TEXT | Дата создания (ISO) |

---

## API-роуты

| Метод | Путь | Описание | Авторизация |
|---|---|---|---|
| POST | `/api/auth/login` | Получить JWT-токен | Нет |
| GET | `/api/auth/me` | Проверить текущего пользователя | JWT |
| GET | `/api/items/{tab}` | Позиции вкладки с агрегированными суммами | Нет |
| PUT | `/api/items/{item_id}/estimate` | Обновить сметное кол-во | JWT |
| GET | `/api/operations/{item_id}` | История операций по позиции | Нет |
| POST | `/api/operations` | Добавить операцию | JWT |
| DELETE | `/api/operations/{op_id}` | Удалить операцию | JWT |
| GET | `/api/last-update` | Дата последней операции в БД | Нет |
| POST | `/api/upload-photo` | OCR накладной через Ollama (фото) | JWT |
| GET | `/api/chats` | Список чатов пользователя | JWT |
| POST | `/api/chats` | Создать новый чат | JWT |
| DELETE | `/api/chats/{id}` | Удалить чат + все сообщения | JWT |
| PATCH | `/api/chats/{id}` | Переименовать чат | JWT |
| GET | `/api/chat_messages/{id}` | История сообщений чата | JWT |
| POST | `/api/chat` | Отправить сообщение, получить ответ ИИ | JWT |

---

## Авторизация (auth.py)

- JWT-токены, TTL 7 дней
- Секрет в `.env` (поле `JWT_SECRET`)
- Поддержка нескольких аккаунтов через `.env`:
  - `developer` / пароль из `DEV_PASSWORD`
  - `director` / пароль из `DIR_PASSWORD`
- Токен передаётся в заголовке `Authorization: Bearer <token>`
- При ошибке 401 — фронтенд автоматически разлогинивает пользователя

---

## .env структура

```
JWT_SECRET=<случайная строка>
ADMIN_USER_1=developer
ADMIN_PASS_1=<пароль>
ADMIN_USER_2=director
ADMIN_PASS_2=<пароль>
OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=gemma4:26b
```

---

## Запуск

```bash
cd "D:\vibecoding projects\romalex - таблица учета\backend"
python main.py
```

Сервер поднимается на `http://localhost:8000`. Фронтенд раздаётся как статика.

---

## 06.06.2026 — бэкенд готов

Реализован с нуля в течение одной сессии. Все роуты работают, авторизация протестирована, seed выполнен (53 позиции в базе).

---

## Технические решения и проблемы

### Конфликт статики с API-роутами

**Проблема:** `app.mount("/", StaticFiles(directory="frontend"))` перехватывало ВСЕ запросы, включая `/api/...` → 404 на все API-эндпоинты.

**Решение:**
- Статика монтируется только на `/assets/`: `app.mount("/assets", StaticFiles(directory="frontend/assets"))`
- Главная страница отдаётся явным роутом: `@app.get("/") → FileResponse("frontend/index.html")`
- Порядок в FastAPI важен: сначала все `@app.xxx` роуты, потом `app.mount`

---

## Связанные ноты

- [[ROMALEX/Таблица учёта по проектам]] — общий контекст проекта
- [[ROMALEX/Фронтенд — HTML CSS JS]] — клиентская часть
- [[ROMALEX/Скрипты — парсер PDF и seed]] — seed.py с 53 позициями
