# Supercare (CARENET.BG) — Custom API

Документация за къстъм endpoint-ите на проекта `supercare` от `projects/supercare/api/front/*.php`.
Ядрените endpoint-и са в `core-v4-front.md` и не се дублират тук.

> Засега са документирани `doctors_specialities` (SEO полетата от 2026-10-05) и работният график (периоди от 2026-10-09). Останалите endpoint-и се добавят при следваща промяна по тях.

---

## 1. Общи конвенции

- Base URL: `https://devcore.myexpozy.com/api/` (дев). Проектът се определя от хедъра `Authentication: Basic <saas_key>`, не от домейна.
- Име на PHP метода = path. `doctors_specialities()` → `GET /doctors_specialities`.
- `title`, `meta_title`, `meta_description`, `content` идват от Content системата (i18n, `lang_content`) за текущия език (`?lang=bg`). Винаги присъстват в JSON-а, дори празни.
- Списъците без пагинация връщат плосък масив. Единичен запис (`/<id>`) връща обекта директно.
- Грешка: `{ "status": 0, "error": "..." }`.

---

## 2. Специалности — `doctors_specialities`

Номенклатура на лекарските специалности. Всяка специалност има собствена SEO страница на фронта.

### `GET /doctors_specialities`

Връща **активните** специалности (`status_id = 1`), подредени по `title` DESC. Без пагинация.

| Param | Тип | Задълж. | Описание |
|-------|-----|---------|----------|
| `slug` | string | не | Филтър по slug (`LIKE %v%`). За зареждане на страница по URL: `?slug=kardiologia` → масив с един елемент. |
| `id` | int | не | Филтър по id. За единичен запис ползвай `/<id>`. |
| `lang` | string | не | Език на текстовете (default: езикът по подразбиране). |

Auth: само `saas_key`.

### `GET /doctors_specialities/<id>`

Връща една специалност по id, независимо от `status_id`.

### Схема на обекта `DoctorSpeciality`

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | int | |
| `icon` | string | CSS клас / име на икона |
| `slug` | string | SEO URL сегмент, уникален, latin lowercase с тирета (`kardiologia`). Генерира се от заглавието, ако админът не зададе свой. |
| `status_id` | int | 1 = активна, 0 = неактивна |
| `title` | string | Заглавие (Content) |
| `meta_title` | string | `<title>` на страницата на специалността (Content). Празно → фронтът ползва `title`. |
| `meta_description` | string | `<meta name="description">` (Content). Plain text. |
| `content` | string | Съдържание на страницата — **HTML** от ContentBuilder редактора (Content). Рендира се с `innerHTML`, не се escape-ва. |
| `date_created` | datetime | |
| `date_updated` | datetime | |

### Примерен отговор — `GET /doctors_specialities/<id>`

_(реален отговор от дев ядрото — ще бъде добавен след теста)_

### Препоръчан flow за страница на специалност

1. `GET /doctors_specialities?slug=<slug>` → ако масивът е празен → 404.
2. `<title>` = `meta_title || title`; `<meta description>` = `meta_description`.
3. Тяло: `content` (HTML) + списък лекари: `GET /doctors?speciality_id=<id>`.

### Промени

- **2026-10-05** — добавени `slug`, `meta_title`, `meta_description`, `content`; нов филтър `?slug=`. Не е breaking — старите полета са непроменени.

---

## 3. Работен график на кабинет — периоди

Графикът на кабинета е **седмичен шаблон от периоди**. Всеки период има свои дни, работно време и продължителност на часа (`interval`). Пример: вт+ср 10:00–12:00 по 15 мин и вт+ср 13:00–14:00 по 20 мин. Празнината между периодите е почивка.

> **BREAKING (2026-10-09).** Единичното работно време (`time_start` / `time_end` / `interval` + `day0`–`day6`) вече не се записва. Графикът се задава **само** с `periods` през `POST /doctors_schedules`. Виж „Промени“ най-долу.

### `POST /doctors_schedules`

Записва периодите и/или онлайн часовете на кабинета. Графикът е един на кабинет (няма `id` на графика).

Auth: `saas_key` + `Authorization: Bearer <token>` на **лекар**, който притежава кабинета (или админ).

| Param | Тип | Задълж. | Описание |
|-------|-----|---------|----------|
| `cabinet_id` | int | да | Кабинетът. |
| `periods` | array | не | **Пълното състояние** на периодите (не се добавя към съществуващите). Липсващ ключ = периодите не се пипат. Празен масив / празен стринг = изтрива всички (кабинетът остава без работно време). |
| `periods[].days` | int[] или `"2,3"` | да | Дни от седмицата: `0` = неделя … `6` = събота. Поне един. |
| `periods[].time_start` | string `HH:MM` | да | Начало. |
| `periods[].time_end` | string `HH:MM` | да | Край — трябва да е след началото. |
| `periods[].interval` | int | да | Продължителност на един час в минути, `5`–`240`. |
| `online_slots` | object | не | Онлайн часовете: `{ "<ден>": ["HH:MM", ...] }`. Пълно състояние. Валидни са само начални часове, които графикът реално генерира за този ден. Липсващ ключ = не се пипат. |

JSON пример:
```json
{
  "cabinet_id": 60,
  "periods": [
    { "days": [2, 3], "time_start": "10:00", "time_end": "12:00", "interval": 15 },
    { "days": [2, 3], "time_start": "13:00", "time_end": "14:00", "interval": 20 }
  ],
  "online_slots": { "2": ["10:00", "13:20"] }
}
```

Form-encoded: `periods[0][days][]=2&periods[0][days][]=3&periods[0][time_start]=10:00&periods[0][time_end]=12:00&periods[0][interval]=15&...`

Правила:
- Последният час, който не се побира до `time_end`, отпада без грешка (13:00–14:00 по 25 мин → 13:00, 13:25).
- Онлайн часове, които след промяна на периодите вече не съществуват (махнат ден или разместени часове), се изтриват автоматично.
- Вече записаните часове (`doctors_appointment`) **не се пипат** при промяна на графика.

Отговор: `{ "status": 1, "obj": <DoctorSchedule>, "msg": "..." }`.

Грешки (`status: 0`, `error`):

| Кога | `error` |
|------|---------|
| няма `cabinet_id` / чужд кабинет | `Cabinet required` / `Cabinet access error` |
| `periods` не е масив | `periods трябва да е масив` |
| период без дни | `Период N: изберете поне един ден` |
| ден извън 0–6 | `Период N: невалиден ден (X)` |
| час не е `HH:MM` | `Период N: невалиден час (очаква се HH:MM)` |
| край ≤ начало | `Период N: краят трябва да е след началото` |
| `interval` извън 5–240 | `Период N: продължителността трябва да е между 5 и 240 мин.` |
| два периода се застъпват в общ ден | `Периоди A и B се застъпват (10:00-12:00 и 11:00-13:00)` |
| онлайн часове при кабинет без работно време | `Кабинетът няма конфигурирано работно време` |
| нито един подаден онлайн час не съществува в графика | `Нито един от подадените часове не съвпада с графика на кабинета` |

При всяка от грешките по-горе **нищо не се записва** — нито периодите, нито онлайн часовете. Ако част от онлайн часовете са валидни, а част не, записват се валидните, а невалидните се пропускат без грешка.

### Обектът `DoctorSchedule`

Идва като `cabinet.schedule` в `GET /doctors`, `GET /doctors/<id>` и като `obj` на `POST /doctors_schedules`.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | int | |
| `cabinet_id` | int | |
| `periods` | `DoctorSchedulePeriod[]` | **Ново.** Периодите, подредени по `time_start`. |
| `day0`–`day6` | bool | Изчислени: денят е работен, ако поне един период го включва. |
| `dayOfWeek` | array[7] | `{id, status_id, title}` за всеки ден; `status_id` = 1 ако е работен. |
| `online_slots` | array[7] | Онлайн часовете по ден: `[[], [], ["10:00","13:20"], ...]`. |
| `time_start`, `time_end`, `interval` | string, string, int | **DEPRECATED** — стойностите на първия период. Ще бъдат премахнати от отговора; не ги ползвай. |
| `date_created`, `date_updated` | datetime | |

### Обектът `DoctorSchedulePeriod`

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | int | |
| `schedule_id` | int | |
| `time_start` | string `HH:MM` | |
| `time_end` | string `HH:MM` | |
| `interval` | int | Минути на един час. |
| `days` | int[] | Дните на периода, `[2, 3, 4]`. |
| `day0`–`day6` | bool | Същото като `days`, по флагове. |
| `dayOfWeek` | array[7] | `{id, status_id, title}` — за показване („Вторник, Сряда“). |
| `date_created` | datetime | |

Примерен `schedule` (реален отговор, `GET /doctors/64`, кабинет 60, `dayOfWeek` съкратен):
```json
{
  "id": 59,
  "cabinet_id": 60,
  "time_start": "15:00",
  "time_end": "18:45",
  "interval": 45,
  "day0": false, "day1": false, "day2": true, "day3": true, "day4": true, "day5": true, "day6": true,
  "periods": [
    {
      "id": 41, "schedule_id": 59,
      "time_start": "15:00", "time_end": "18:45", "interval": 45,
      "day0": false, "day1": false, "day2": true, "day3": true, "day4": true, "day5": true, "day6": true,
      "days": [2, 3, 4, 5, 6],
      "dayOfWeek": [
        { "id": 0, "status_id": 0, "title": "Неделя" },
        { "id": 1, "status_id": 0, "title": "Понеделник" },
        { "id": 2, "status_id": 1, "title": "Вторник" }
      ],
      "date_created": "2026-10-09 09:57:28"
    }
  ],
  "online_slots": [[], [], [], [], [], [], []]
}
```

### `GET /doctors_timeslots?cabinet_id=<id>&date_from=YYYY-MM-DD`

Свободните/заетите часове за 3 дни от `date_from`. **Часовете се генерират на бекенда** — фронтът не трябва да ги смята сам от `time_start`/`time_end`/`interval`: с периодите те са различни по дни и по продължителност.

Auth: само `saas_key`.

Отговор — масив от дни:

| Поле | Тип | Описание |
|------|-----|----------|
| `date` | `YYYY-MM-DD` | |
| `dayOfWeek` | string | `0`–`6` |
| `title` | string | „Днес“ / „Утре“ / име на деня |
| `is_working` | bool | Работен ден без изключение. |
| `is_exception` | bool | Целият ден е блокиран. |
| `time_slots[]` | array | `{time_start, time_end, interval, status_id, online}`; `status_id`: 1 свободен, 0 зает, 2 блокиран. **`interval` е ново** — продължителността на конкретния час. |

Пример (реален, кабинет 60, съкратен):
```json
[
  { "date": "2026-10-12", "dayOfWeek": "1", "title": "Понеделник", "is_working": false, "is_exception": false, "time_slots": [] },
  { "date": "2026-10-13", "dayOfWeek": "2", "title": "Вторник", "is_working": true, "is_exception": false,
    "time_slots": [
      { "time_start": "15:00", "time_end": "15:45", "interval": 45, "status_id": 1, "online": false },
      { "time_start": "15:45", "time_end": "16:30", "interval": 45, "status_id": 1, "online": false }
    ] }
]
```

### Записване на час / блокиране

`POST /doctors_appointment`, `/doctors_appointment_series`, `/doctors_timeslot_block_series` и блокирането на час валидират `time_start` спрямо часовете **на конкретния ден от седмицата** (от периодите). Час, който съществува във вторник, но не и в сряда, се отказва за сряда с `Invalid time slot`. Записаният час получава `time_end` и `interval` от своя слот.

### Промени

- **2026-10-09 — BREAKING**
  - Нов масив `schedule.periods[]` и нов параметър `periods` в `POST /doctors_schedules`.
  - `POST /doctors_schedules` **вече не приема** `time_start`, `time_end`, `interval`, `day0`–`day6` (игнорират се).
  - `POST /doctors_cabinets` **вече не записва графика** — `time_start`, `time_end`, `interval`, `day0`–`day6` се игнорират. Нов кабинет няма график, докато не се извика `POST /doctors_schedules` с `periods`.
  - `schedule.time_start` / `time_end` / `interval` са DEPRECATED и ще бъдат премахнати от отговора, щом фронтът мине на `periods`.
  - `time_slots[].interval` — ново поле в `GET /doctors_timeslots`.
  - Фронтът трябва да спре да генерира часове в JS от `schedule.time_start/time_end/interval` (record.html, userpage.html, модалът за онлайн часове) — да ползва `GET /doctors_timeslots` или `schedule.periods`.

---

## 4. Профил на лекар — `POST /doctors`

Лекарят записва собствения си профил (записът винаги е върху профила на логнатия лекар; `id` от заявката се игнорира).

Auth: `saas_key` + `Authorization: Bearer <token>` на потребител с ниво лекар.

Параметрите са полетата на профила (`speciality_ids`, `tag_ids`, `nzok`, `children`, `home_visits`, `video_url`, `show_phone`, `first_name`, `last_name`, `phone`, …).

**Не се приемат** (тихо се игнорират): `status_id`, `payment_status_id`, `user_id` — менят се само от админа.

### Промени

- **2026-10-09** — `status_id`, `payment_status_id` и `user_id` вече се игнорират (преди лекар можеше сам да се активира или да се маркира като платил). Не е breaking за легитимен фронт.
