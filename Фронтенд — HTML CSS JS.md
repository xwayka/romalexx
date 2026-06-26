---
date: 2026-06-06
type: technical
project: ROMALEX
tags: [ROMALEX, html, css, javascript, frontend]
---

# Фронтенд — HTML CSS JS

[[ROMALEX]] | [[ROMALEX/Таблица учёта по проектам]]

---

## Архитектура

Фронтенд разбит на три файла (рефакторинг от 05.06.2026 — до этого был монолитный `index.html`):

```
frontend/
├── index.html   ← структура, разметка, модальные окна
├── style.css    ← все стили, CSS-переменные, темы
└── app.js       ← вся логика, API-запросы, рендеринг
```

Стек — чистый HTML/CSS/JS, без фреймворков. SheetJS (xlsx) подключён через CDN для экспорта Excel.

---

## index.html

### Основные блоки разметки

- **`<header>`** — три зоны: кнопка авторизации (слева), логотип + название проекта (центр), переключатель темы (справа)
- **`.project-bar`** — строка с адресом объекта и кнопками (загрузить фото, статистика)
- **`.tabs`** — переключатели вкладок (8 штук)
- **`<table>`** — основная таблица с двухуровневой шапкой
- **`#modal-login`** — модальное окно входа
- **`#modal-operation`** — модальное окно добавления операции
- **`#modal-photo`** — модальное окно OCR (загрузка фото накладной)
- **`#panel-history`** — выезжающая панель истории операций
- **`#modal-download`** — модальное окно после скачивания Excel (с обратным отсчётом)

### Двухуровневая шапка таблицы

```html
<thead>
  <tr class="header-group">
    <th colspan="3"></th>
    <th colspan="2">Проектное</th>
    <th colspan="3">Киржач</th>
    <th colspan="3">ГСК 2</th>
    <th colspan="3">ГСК 3</th>
    <th colspan="2">Итого</th>
  </tr>
  <tr class="header-sub">
    <th>№</th><th>Наименование</th><th>Ед.</th>
    <th>Проектное кол-во</th><th>Сметное кол-во</th>
    <th>Приход</th><th>Расход</th><th>Остаток</th>
    <!-- ...и так для ГСК2, ГСК3 -->
    <th>Выполнение по КС</th><th>Аналитика %</th>
  </tr>
</thead>
```

---

## style.css

### CSS-переменные (темы)

```css
:root {
  --bg: #ffffff;
  --surface: #f5f5f5;
  --card: #efefef;
  --text: #1a1a1a;
  --text-muted: #666;
  --border: #ddd;
  --accent: #217346;        /* корпоративный зелёный Ромалекс */
  --accent-hover: #1a5c38;
  --danger: #c0392b;
}

[data-theme="dark"] {
  --bg: #121212;
  --surface: #1e1e1e;
  --card: #2a2a2a;
  --text: #e8e8e8;
  --text-muted: #999;
  --border: #3a3a3a;
}
```

Тема хранится в `localStorage` под ключом `romalex_theme`. При загрузке страницы читается и применяется через `document.documentElement.setAttribute('data-theme', ...)`.

### Анимация смены темы

Clip-path circle от правого верхнего угла — там расположена кнопка переключателя:

```js
function toggleTheme() {
  const cur = document.documentElement.getAttribute('data-theme');
  const next = cur === 'dark' ? 'light' : 'dark';
  const bg = getComputedStyle(document.documentElement).getPropertyValue('--bg').trim();
  const overlay = document.createElement('div');
  overlay.style.cssText =
    'position:fixed;inset:0;z-index:9999;pointer-events:none;will-change:clip-path;' +
    `background:${bg};clip-path:inset(0 0 0 0)`;
  document.body.appendChild(overlay);
  applyTheme(next);  // новая тема сразу под overlay
  overlay.getBoundingClientRect();  // force reflow
  overlay.style.transition = 'clip-path 0.42s cubic-bezier(0.4,0,0.2,1)';
  overlay.style.clipPath = 'inset(100% 0 0 0)';  // штора уезжает вниз
  overlay.addEventListener('transitionend', () => overlay.remove(), { once: true });
}
```

⚠️ CSS `transition: background 0.2s` с `body` удалён — конфликтовал с JS анимацией.

### Кнопки единого стиля (project-bar)

```css
.project-bar button {
  background: var(--card);
  border: 1px solid var(--border);
  color: var(--text);
  border-radius: 6px;
  padding: 6px 14px;
  cursor: pointer;
  transition: background 0.2s, border-color 0.2s;
}
.project-bar button:hover {
  background: var(--accent);
  border-color: var(--accent);
  color: #fff;
}
```

### Admin-only элементы

```css
.admin-only { display: none; }
.logged-in .admin-only { display: inline-flex; }
```

При входе класс `logged-in` добавляется на `<body>` — все admin-only элементы появляются.

---

## app.js

### Архитектура

Глобальное состояние в объекте `state`:
```js
const state = {
  currentTab: 'tube',
  items: [],
  token: localStorage.getItem('romalex_token'),
  user: localStorage.getItem('romalex_user'),
};
```

### Ключевые функции

| Функция | Описание |
|---|---|
| `apiFetch(path, options)` | Все запросы к API, автоматически добавляет Bearer-токен, при 401 — разлогин |
| `loadItems()` | GET /api/items, сохраняет в state.items, вызывает renderTable |
| `renderTable()` | Рендерит таблицу по state.currentTab + state.items |
| `openHistory(itemId, warehouse, type)` | Открывает панель истории, загружает операции |
| `addOperation(data)` | POST /api/operations |
| `deleteOperation(id)` | DELETE /api/operations/{id} |
| `exportExcel()` | GET /api/export, формирует xlsx через SheetJS |
| `uploadPhoto(file)` | POST /api/upload-photo, показывает результат OCR |
| `toggleTheme()` | Меняет data-theme, сохраняет в localStorage, запускает анимацию |

### Excel-экспорт

Имя файла формируется динамически:
```js
const now = new Date();
const dd = String(now.getDate()).padStart(2, '0');
const mm = String(now.getMonth()+1).padStart(2, '0');
const yyyy = now.getFullYear();
const hh = String(now.getHours()).padStart(2, '0');
const min = String(now.getMinutes()).padStart(2, '0');
const fname = `Ромалекс_таблица учета_ЖК Рязанский пр вл.26-1 ГСК 2-3_актуально на_${dd}.${mm}.${yyyy}_${hh}.${min}.xlsx`;
```

После скачивания показывается `#modal-download` с обратным отсчётом **2 секунды** на кнопке закрытия. Пока идёт отсчёт: кнопка disabled, выглядит как белая пилюля с числом (не как ×), клик мимо модалки заблокирован. Показывается: «Проект: ЖК Рязанский пр.» и «Актуально на: ДД.ММ.ГГГГ&nbsp;ЧЧ:ММ».

---

## UI-компоненты

### Панель истории (slide-in)

Выезжает снизу. Два режима:
1. **По ячейке** — показывает историю конкретного склада + типа (Приход/Расход) по одной позиции
2. **По названию** — показывает ВСЕ операции позиции по всем складам

Внутри — таблица: `Дата | Склад | Тип | Кол-во | Поставщик | Накладная`. В режиме фильтрации колонки Склад и Тип скрыты (они известны из заголовка).

### Модал добавления операции

Появляется кнопкой `+` (admin-only) при hover на строке таблицы. Поля: склад, тип (приход/расход), количество, дата, поставщик/подрядчик, номер накладной.

---

## Связанные ноты
- [[ROMALEX/Таблица учёта по проектам]] — общий контекст
- [[ROMALEX/Бэкенд — FastAPI SQLite]] — API к которому обращается фронтенд
- [[ROMALEX/Дизайн — таблица учёта]] — дизайн-система и CSS-решения

---

## Редизайн сайдбара облачного хранилища (23.06.2026)

[[ROMALEX/Облачное хранилище — UI и масштабирование]] | [[ROMALEX/Облачное хранилище ROMALEX — концепт]]

Полный редизайн сайдбара `cloud-ui-concepts.html` / `frontend/index.html` — новый glassmorphism стиль с pill-навигацией и акцентными цветами по разделам.

### Что изменено в style.css

**Glassmorphism сайдбар:**
```css
.sidebar {
  background: rgba(12, 12, 14, 0.35);
  backdrop-filter: blur(80px) saturate(180%);
}
body::before {
  /* белые radial-gradient блобы, opacity 0.12–0.18 */
}
```

**Pill-навигация `.sb-pill`:**
- Пассивное состояние: `border: 1px solid rgba(255,255,255,0.09)`
- Активное: цветная обводка + `box-shadow: 0 0 8px <accent-color>` glow
- Иконки без квадратного блока: `border: none; background: none`

**Акцентные цвета по разделам:**
| Раздел | Цвет |
|---|---|
| Дашборд | blue |
| Документы | amber |
| Граф | purple |
| Учёт проектов | orange |
| Личное | green |
| Заметки | pink |
| Избранное | lime |
| Недавние | cyan |
| Корзина | red |

### Что изменено в index.html

- Раздел «Теги» удалён полностью из nav и из dashboard
- «Учёт проектов» — убрана выпадашка (`<details>`/стрелка), теперь просто пункт
- Группы сайдбара ГЛАВНОЕ / ЛИЧНОЕ / ПРОЧЕЕ — сворачиваются по клику

### Что изменено в app.js

- Логика `show(n)` обновляет `.sb-pill.on` синхронно с активным экраном
- Каждый пункт сайдбара имеет свой экран (нет переброса «Недавние» → Дашборд и т.д.)

**Статус:** завершено ✓
