# Supercare (CARENET.BG) — Custom API

Документация за къстъм endpoint-ите на проекта `supercare` от `projects/supercare/api/front/*.php`.
Ядрените endpoint-и са в `core-v4-front.md` и не се дублират тук.

> Засега е документиран само `doctors_specialities` (SEO полетата от 2026-10-05). Останалите endpoint-и се добавят при следваща промяна по тях.

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
