# Startup Studio NSU — Система посещаемости

## Архитектура

### Текущая версия (Phase 1 — GitHub Pages)

Приложение работает полностью на клиентской стороне и не требует сервера. Это позволяет деплоить на GitHub Pages бесплатно.

**Схема работы:**

```
Преподаватель                           Студент
─────────────                           ────────
[Настройка сессии]                      [Сканирует QR]
       │                                      │
[Генерируется токен]                   [Открывает /?t=TOKEN&...]
       │                                      │
[QR → зашифрованный URL]         ┌────[Проверки:]
       │                         │    1. Токен не истёк (timestamp)
[Таймер обновляет QR]            │    2. Геолокация в радиусе (если включена)
       │                         │    3. Device fingerprint не в localStorage
[Google Sheets CSV]              │         │
[polling каждые 5 сек]←──────── │    [Открывается форма в iframe]
       │                         │    (blob: URL — прямая ссылка скрыта)
[Список обновляется]             │         │
                                 └───[После submit → fingerprint записан]
```

**Что защищает:**
- QR-код меняется каждые N секунд → нельзя переслать отсутствующему
- URL формы никогда не показывается студенту (загружается через fetch + blob:)
- Один fingerprint = одна отметка за сессию (localStorage)
- Геолокация с учётом погрешности GPS обоих устройств

**Ограничения текущей версии:**
- Fingerprint хранится в localStorage → обходится через приватный режим браузера
- Нет realtime синхронизации имён (используется polling CSV из Google Sheets)
- Нет серверной валидации токенов

---

### Production версия (Phase 2 — Supabase + Vercel/Netlify)

#### Стек

| Компонент | Технология | Стоимость |
|-----------|-----------|-----------|
| Frontend | Next.js или SvelteKit | Бесплатно (Vercel/Netlify) |
| База данных | Supabase PostgreSQL | Бесплатно (500MB) |
| Realtime | Supabase Realtime (WebSocket) | Включено |
| Auth | Supabase Auth | Включено |
| Edge Functions | Supabase Edge Functions (Deno) | 500K вызовов/месяц бесплатно |
| Hosting | Vercel Hobby | Бесплатно |

**Итого: $0/месяц** для объёма университета (~500 студентов).

---

#### Схема базы данных

```sql
-- Сессии (создаёт преподаватель)
CREATE TABLE sessions (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject     TEXT NOT NULL,
  forms_url   TEXT NOT NULL,
  interval    INT  NOT NULL DEFAULT 20,
  geo_lat     FLOAT,
  geo_lng     FLOAT,
  geo_radius  INT  DEFAULT 80,
  active      BOOL DEFAULT true,
  created_at  TIMESTAMPTZ DEFAULT now(),
  ended_at    TIMESTAMPTZ
);

-- Активные токены QR (валидация на сервере)
CREATE TABLE qr_tokens (
  token       TEXT PRIMARY KEY,
  session_id  UUID REFERENCES sessions(id),
  expires_at  TIMESTAMPTZ NOT NULL,
  used        BOOL DEFAULT false
);

-- Отметки студентов
CREATE TABLE attendance (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id  UUID REFERENCES sessions(id),
  fp_hash     TEXT NOT NULL,    -- SHA-256 от fingerprint (не храним сам ID)
  name        TEXT,
  group_name  TEXT,
  marked_at   TIMESTAMPTZ DEFAULT now(),
  UNIQUE(session_id, fp_hash)   -- один студент = одна запись
);

-- Индексы
CREATE INDEX ON qr_tokens(session_id, expires_at);
CREATE INDEX ON attendance(session_id);
```

---

#### Серверные функции (Supabase Edge Functions)

**1. `POST /functions/v1/validate-token`**
```typescript
// Студент отправляет: { token, fingerprint, geo? }
// Сервер проверяет: токен валиден, не истёк, fingerprint не использован
// Возвращает: { valid: true, sessionId, formsUrl (зашифрован) }
// Или: { valid: false, reason: 'expired' | 'already_marked' | 'geo_fail' }
```

**2. `POST /functions/v1/mark-attendance`**
```typescript
// Вызывается после отправки формы (через Google Apps Script webhook)
// Записывает: fp_hash, name, group_name в таблицу attendance
// Supabase Realtime рассылает событие всем подключённым клиентам
```

**3. Google Apps Script webhook** (в Google Sheets)
```javascript
function onFormSubmit(e) {
  const name  = e.values[1]; // Имя из формы
  const group = e.values[2]; // Группа из формы
  const fp    = e.values[3]; // Fingerprint (скрытое поле формы, auto-заполненное)
  
  UrlFetchApp.fetch('https://your-project.supabase.co/functions/v1/mark-attendance', {
    method: 'POST',
    contentType: 'application/json',
    payload: JSON.stringify({ fp, name, group, sessionId: ... })
  });
}
```

---

#### Realtime — обновление списка у преподавателя

```typescript
// В компоненте Teacher dashboard
const channel = supabase
  .channel('attendance-updates')
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'attendance',
    filter: `session_id=eq.${sessionId}`
  }, (payload) => {
    // Мгновенно добавляем студента в список без перезагрузки
    addStudentToList(payload.new);
  })
  .subscribe();
```

---

## Деплой текущей версии (Phase 1)

### GitHub Pages

```bash
# 1. Создайте репозиторий на GitHub
git init
git add index.html
git commit -m "Initial attendance app"
git remote add origin https://github.com/YOUR/attendance.git
git push -u origin main

# 2. Settings → Pages → Source: main branch / root
# 3. Приложение доступно по адресу:
#    https://YOUR.github.io/attendance/
```

### Настройка Google Sheets polling (для имён студентов)

1. Откройте вашу Google Form → вкладка "Ответы" → нажмите значок Google Sheets
2. В открытой таблице: **Файл → Поделиться → Опубликовать в интернете**
3. Выберите: "Весь документ" → "Значения, разделённые запятыми (.csv)"
4. Скопируйте ссылку, она выглядит так:
   ```
   https://docs.google.com/spreadsheets/d/XXXX/pub?output=csv
   ```
5. В `index.html` найдите строку:
   ```javascript
   // startSheetsPolling('YOUR_SHEETS_CSV_URL_HERE');
   ```
   Раскомментируйте и вставьте вашу ссылку.

6. В вашей Google Form убедитесь, что первые вопросы — это:
   - Вопрос 1: Временная метка (автоматически)
   - Вопрос 2: Имя и Фамилия
   - Вопрос 3: Номер группы

---

## Структура URL QR-кода

```
https://your-site/?t=TOKEN&f=ENCRYPTED_FORMS_URL&e=EXPIRY_TS&s=SESSION_ID
                   &gl=GEO_LAT&gn=GEO_LNG&gr=RADIUS&ga=TEACHER_ACCURACY
```

| Параметр | Описание |
|----------|----------|
| `t` | Одноразовый токен (14 символов, меняется с каждым QR) |
| `f` | URL формы, зашифрованный XOR+base64 |
| `e` | Unix timestamp истечения (ms) |
| `s` | ID сессии (привязывает fingerprint к конкретному занятию) |
| `gl`, `gn` | Координаты аудитории, обфусцированные XOR |
| `gr` | Допустимый радиус в метрах |
| `ga` | Точность GPS преподавателя (для компенсации погрешности) |

---

## Сравнение Phase 1 vs Phase 2

| Критерий | Phase 1 (текущая) | Phase 2 (Supabase) |
|----------|-------------------|-------------------|
| Серверная валидация токенов | ❌ | ✅ |
| Защита от приватного режима | ⚠️ Частичная | ✅ (IP + UA комбо) |
| Realtime обновление имён | ⚠️ CSV polling | ✅ WebSocket |
| Требует сервера | ❌ | ✅ |
| Стоимость | $0 | $0 |
| Сложность деплоя | Минимальная | Средняя (~2 часа) |
| Надёжность fingerprint | ~85% | ~95% |
