# autochinagz — Custom API (front)

Документация за **всички къстъм front endpoint-и** на проекта —
`projects/autochinagz/api/front/{Get,Post,Delete,Put}.php` в devcore.
Ядрените endpoint-и (вход, профил, адреси, блог и т.н.) са в `core-v4-front.md` и тук не се повтарят.
Админските (`api/admin/*`) не са част от този документ.

Сайт: `https://autochinagz.com` · език на текстовете в отговорите: **английски**.

> Проектът е отделен от autochinabg — кодът е сходен, но **не е еднакъв**. Разликите са отбелязани с **[gz]**.

---

## Съдържание

1. [Общи конвенции](#1-общи-конвенции)
2. [Каталог — автомобили](#2-каталог--автомобили)
3. [Каталог — мотоциклети](#3-каталог--мотоциклети)
4. [Каталог — камиони](#4-каталог--камиони)
5. [Обяви — публично](#5-обяви--публично)
6. [Регистрация и профил](#6-регистрация-и-профил)
7. [Моите обяви (продавач)](#7-моите-обяви-продавач)
8. [Заявки — резервация, покупка, лизинг](#8-заявки--резервация-покупка-лизинг)
9. [Огледи](#9-огледи)
10. [Аукцион — наддаване, депозити, наблюдавани](#10-аукцион--наддаване-депозити-наблюдавани)
11. [Сделки](#11-сделки)
12. [Калкулатори и справочни данни](#12-калкулатори-и-справочни-данни)
12a. [Репорти](#12a-репорти-gz)
13. [Въпроси и отговори (Q&A)](#13-въпроси-и-отговори-qa)
14. [Служебни — не за фронта](#14-служебни--не-за-фронта)
15. [Схеми](#15-схеми)
16. [Известни проблеми](#16-известни-проблеми)

---

## 1. Общи конвенции

### 1.1. URL и хедъри

**Base URL:** `https://devcore.myexpozy.com/api` (дев). Името на PHP метода = пътят:
`my_cars_inspections()` → `/my_cars_inspections`. Единичен запис: `/<endpoint>/<id>` или `?id=<id>`.

| Хедър | Кога | Описание |
|---|---|---|
| `Authentication: Basic <saas_key>` | винаги | Ключът на проекта (публичен — `const SAAS_KEY` в HTML-а на сайта). Може и `?saas_key=`. |
| `Authorization: Bearer <token>` | 🔒 endpoint-ите | Токенът на логнатия потребител. |

В документа: 🔓 — публичен (само `saas_key`), 🔒 — изисква логнат потребител.
Роли: **продавач** (`userlevel` = marketplace seller, има запис `CarsSeller`) и **клиент/купувач** (има запис `CarsClient`).

POST/DELETE тялото е `application/x-www-form-urlencoded` или `multipart/form-data` (при файлове).
Масиви се подават с `[]` (`image_order[]=5&image_order[]=3`), групи — с bracket notation (`car_info[year]=2024`).

### 1.2. Формат на отговора

| Случай | Тяло |
|---|---|
| Единичен обект | обектът директно `{ "id": 5, ... }` |
| Списък (`get_all`) | плосък масив `[ {...}, {...} ]` |
| Списък с пагинация | `{ "pagination": {...}, "result": [ ... ] }` |
| Успешен запис | `{ "status": 1, "msg": "...", "obj": {...} }` (`obj` не винаги) |
| Грешка | HTTP **400** + `{ "status": 0, "error": "текст", "errors": ["текст"] }` |
| Липсващ задължителен параметър | HTTP **400** + `{ "status": 0, "errors": { "[name=\"deal_id\"]": "..." } }` |
| Без токен на 🔒 endpoint | HTTP **403** + `{ "status": 0, "errors": ["Login first v4!"] }` |
| Невалиден/изтекъл токен | `{ "errors": ["Failed authorization 1"], "redirect": "/logout" }` |

Пагинация:

```json
{
  "pagination": { "total_results": 11, "total_pages": 2, "current_page": 1, "results_per_page": 10 },
  "result": [ { "id": 96, "...": "..." } ]
}
```

| Параметър | Описание |
|---|---|
| `page` | номер на страница (от 1) |
| `limit` | брой на страница |
| `no_pagination=1` | плосък масив без обвивка |

### 1.3. Общи филтри на списъците

Всеки списък, който минава през `get_all`, приема филтър по **всяко поле на обекта**:
стрингове → `LIKE %v%`, числа/bool → `=`, `date_*` → по дата. Плюс `order_by` + `order_direction` (`ASC`/`DESC`).
Номенклатурите (горива, каросерии и т.н.) връщат само активните (`status_id = 1`).

### 1.4. Прикачени файлове

Снимки (`SimpleImage`) — `{ id, object_id, filename, sort_order, date_created, url, url_10x10 }`;
снимките на обява имат и `type` (`exterior` / `interior` / `detail`).
Файлове (`SimpleFile` — видео, репорт, брошура) — `{ id, object_id, filename, url, path, date_created }`.
Липсващ единичен файл идва като обект с `id: 0` и празни стрингове, не `null`.

---

## 2. Каталог — автомобили

Всички са 🔓 `GET`. Номенклатурите са плосък масив, подреден по `sort`.

| Endpoint | Връща | Бележки |
|---|---|---|
| `GET /cars_manufacturers` | `CarsManufacturer[]` | `?id=` — една марка |
| `GET /cars_models` | пагиниран `CarsModel[]` | само активни модели на активни марки; по подразбиране `order_by=manufacturer_title` (марка, после модел); филтри: `manufacturer_id`, `title`… |
| `GET /cars_models_after_2024` | пагиниран `CarsModel[]` | същото, но само модели с поне една вариация от 2024 г. нататък |
| `GET /cars_model_variants` | `CarsModelVariant[]` | `?model_id=` — вариациите на модел. ⚠ без филтър връща всички (~9000) |
| `GET /cars_model_specifications` | `CarsSpecification` / `[]` | `?variant_id=` → обект (спецификацията на вариацията); `?model_id=` → обект (първата за модела); `?id=`. ⚠ без филтър връща **всички** без пагинация (~90 MB) — винаги подавай филтър |
| `GET /cars_model_specifications_template?fuel_id=` | `{ секция: { поле: "" } }` | празен шаблон на `specs` за вид гориво; без `fuel_id` — само общите полета |
| `GET /cars_fuels` | `Nomenclature[]` | `{ id, title, sort, status_id }` |
| `GET /cars_transmissions` | `Nomenclature[]` | |
| `GET /cars_drivetrains` | `Nomenclature[]` | |
| `GET /cars_body_types` | `Nomenclature[]` | |
| `GET /cars_battery_types` | `Nomenclature[]` | |
| `GET /cars_adas_levels` | `Nomenclature[]` | |

Пример `GET /cars_fuels`:

```json
[ { "id": 8, "title": "BEV", "sort": 10, "status_id": 1 } ]
```

Пример `GET /cars_model_specifications_template?fuel_id=1` (съкратен):

```json
{
  "Основна информация": { "Премиера": "", "Гориво": "", "Дължина/Ширина/Височина": "" },
  "Каросерия": { "Купе": "", "Брой врати": "" },
  "Трансмисия": { "Описание на трансмисията": "", "Брой предавки": "", "Тип трансмисия": "" }
}
```

Схеми: [CarsManufacturer](#151-каталог-автомобили), [CarsModel](#151-каталог-автомобили), [CarsSpecification](#151-каталог-автомобили).

---

## 3. Каталог — мотоциклети

Всички са 🔓 `GET`. Атрибутите имат и `title_cn` (оригиналното китайско име).

| Endpoint | Връща | Бележки |
|---|---|---|
| `GET /moto_manufacturers` | `MotoManufacturer[]` | `?id=` |
| `GET /moto_models` | пагиниран `MotoModel[]` | `?manufacturer_id=`, `?no_pagination=1` |
| `GET /moto_model_variants` | `MotoModelVariant[]` | `?model_id=`. ⚠ без филтър — всички (~2400) |
| `GET /moto_model_specifications` | пагиниран `MotoSpecification[]` / обект | `?variant_id=` / `?model_id=` → обект. Диапазони: `price_from/to`, `displacement_from/to`, `range_from/to`, `power_from/to`, `seat_height_from/to`, `weight_from/to` |
| `GET /moto_types` | `MotoNomenclature[]` | скутер, нейкид, ендуро… |
| `GET /moto_categories` | `MotoNomenclature[]` | правни категории (ел. мотоциклет, ел. велосипед…) |
| `GET /moto_energies` | `MotoNomenclature[]` | задвижване |
| `GET /moto_battery_types` | `MotoNomenclature[]` | |
| `GET /moto_coolings` | `MotoNomenclature[]` | |
| `GET /moto_drives` | `MotoNomenclature[]` | предаване (верига, кардан…) |

`MotoNomenclature`: `{ id, title, title_cn, sort, status_id }`. Схеми: [мотоциклети](#152-каталог-мотоциклети).

---

## 4. Каталог — камиони

Всички са 🔓 `GET`.

| Endpoint | Връща | Бележки |
|---|---|---|
| `GET /truck_manufacturers` | `TruckManufacturer[]` | `?id=` |
| `GET /truck_models` | пагиниран `TruckModel[]` (серии) | `?manufacturer_id=`, `?category_id=`, `?on_sale=1` (без спрените от продажба), `?no_pagination=1` |
| `GET /truck_model_variants` | `TruckModelVariant[]` | `?model_id=`, `?in_production=1` |
| `GET /truck_model_specifications` | пагиниран `TruckSpecification[]` / обект | `?variant_id=` / `?model_id=` → обект. Диапазони: `price_from/to`, `power_from/to`, `horsepower_from/to`, `displacement_from/to`, `gross_weight_from/to`, `payload_from/to`, `range_from/to`, `wheelbase_from/to`, `length_from/to` |
| `GET /truck_categories` | `TruckNomenclature[]` (+ `source_id`) | товарен, самосвал, влекач… |
| `GET /truck_tonnages` | `TruckNomenclature[]` | |
| `GET /truck_energies` | `TruckNomenclature[]` | |
| `GET /truck_emissions` | `TruckNomenclature[]` | |
| `GET /truck_drive_forms` | `TruckNomenclature[]` | 4X2, 6X4… |
| `GET /truck_cabs` | `TruckNomenclature[]` | |
| `GET /truck_transmissions` | `TruckNomenclature[]` | |
| `GET /truck_battery_types` | `TruckNomenclature[]` | |
| `GET /truck_body_types` | `TruckNomenclature[]` | |

`TruckNomenclature`: `{ id, title, title_cn, sort, status_id }`. Схеми: [камиони](#153-каталог-камиони).

---

## 5. Обяви — публично

### `GET /cars_auctions` 🔓

Каталогът на обявите. Пагиниран `CarsAuctionFrontend[]`. Връща само статуси **Scheduled (5)** и **Live (6)**.
**[gz]** Свалените от продавача (`hidden = true`) **също излизат** — подай `hidden=0` (виж [т.16](#16-известни-проблеми)).

| Параметър | Тип | Описание |
|---|---|---|
| `id` | int | единична обява (без филтъра по статус) |
| `slug` | string | единична обява по slug (`96-byd-tang-l-ev-2025`) |
| `type` | string | `reservation` / `auction` |
| `location` | string | `china` (в Китай) / `on_road` (на път) / `bg` (в БГ) |
| `manufacturer_id`, `model_id` | int | |
| `manufacturer_name`, `model_name` | string | текстово търсене по име |
| `search` | string | ключови думи — марка, модел, двигател, град, оборудване, доп. информация |
| `year_from`, `year_to` | int | |
| `power_from`, `power_to` | int | к.с. (`car_info.power_hp`) |
| `battery_from`, `battery_to` | float | kWh |
| `price_from`, `price_to` | float | `price` (за `type=auction` — `start_price`) |
| `fuel_id`, `transmission_id`, `drivetrain_id`, `body_type_id` | int | |
| `is_new` | 0/1 | |
| `city` | string | LIKE |
| `exterior_color` | string | точно съвпадение |
| `featured` | 0/1 | |
| `hidden` | 0/1 | **[gz]** `hidden=0` скрива свалените от продавача |
| `buy_now` | 1 | само с buy-now цена (аукциони) |
| `order_by` | string | `price`, `year`, `mileage`, `date_created`, `date_end` + `order_direction`; или съкратено `price_desc`, `year_asc`… По подразбиране — най-скоро обновените (`date_updated DESC`) |
| `page`, `limit` | int | пагинация |

Пример (съкратен):

```json
{
  "pagination": { "total_results": 11, "total_pages": 2, "current_page": 1, "results_per_page": 10 },
  "result": [
    {
      "id": 83, "title": "Zeekr 8X 2026", "type": "reservation", "location": "china",
      "status_id": 6, "price": 81700, "reservation_fee": 8170, "reservation_fee_percent": 10,
      "can_reserve": true, "can_buy_now": true, "can_lease": false,
      "slug": "83-zeekr-8x-2026", "url": "https://autochinagz.com/en/auction/83-zeekr-8x-2026",
      "status": { "id": 6, "title": "Live", "color": "emerald" },
      "car_info": { "year": 2026, "mileage": 6000, "power_hp": 422, "battery_kwh": 76, "...": "..." },
      "seller": { "id": 40, "phone_code": "359", "phone": "883341911", "...": "..." },
      "images_exterior": [ { "id": 861, "url": "https://r2.expozy.com/...", "type": "exterior" } ]
    }
  ]
}
```

Схема: [CarsAuctionFrontend](#154-обява).

### `GET /cars_auctions_statuses` 🔓

`[{ id, title, color }]`. Статуси на обява:

| id | title | |
|---|---|---|
| 0 | Draft | |
| 1 | Pending review | |
| 2 | Needs changes | |
| 3 | Rejected | |
| 4 | Approved | |
| 5 | Scheduled | |
| 6 | Live | публикувана |
| 7 | Ended | продадена / изтекла |

> **[gz]** Няма преглед от админ: резервационните обяви стават `6` (30 дни) директно; аукционите при потвърждение
> стават `5` с автоматични дати (старт след 2–5 дни), после `6 → 7`. Админът може да върне обява в `2` или `3`.

### `GET /cars_auctions_bids?auction_id=` 🔓

Наддаванията по публична обява — масив от `{ id, auction_id, user_id, amount, date_created, ... }`.
Грешка, ако обявата не е публична.

### `POST /cars_auctions_calls` 🔓

Брои показване на телефона („Обади се"). Повторен клик от същия IP до 10 мин. не се брои.

| Параметър | Тип | Задължителен |
|---|---|---|
| `auction_id` | int | да |

```json
{ "status": 1, "calls_count": 4 }
```

### `POST /cars_model_specifications_views` 🔓

Брои отваряне на спецификация. Повторно от същия IP до 10 мин. не се брои.

| Параметър | Тип | Задължителен |
|---|---|---|
| `specification_id` | int | да |

```json
{ "status": 1, "views_count": 6 }
```

---

## 6. Регистрация и профил

### `POST /users` 🔓 — регистрация

Замества ядрената регистрация. Полетата на потребителя са тези на ядрото (`core-v4-front.md` → регистрация):
`email`, `password`, `first_name`, `last_name`, телефон и т.н.

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `password` | string | да | |
| `first_name`, `last_name` | string | да | |
| `seller` | 1 | не | `1` → регистрация като **продавач** (статус „Нов" — активира го админът); без него → **клиент** |
| … | | | останалите полета на ядрената регистрация |

Отговор: резултатът от ядрената регистрация + `seller` (`CarsSeller`) или `client` (`{ id, user_id }`).

> Нов продавач не може да публикува, докато админът не го активира (`status_id = 1`).

### `GET /my_cars_seller` 🔒

Търговският профил на логнатия. Обект [CarsSeller](#155-продавач-и-клиент) (`id: 0`, ако не е продавач).

### `POST /cars_seller` 🔒 — редакция на търговския профил

Всички полета са по избор; подават се само тези за промяна.

| Параметър | Тип | Описание |
|---|---|---|
| `email_notification` | 0/1 | мейл известия |
| `nolink`, `whatsapp`, `viber`, `telegram` | string | канали за връзка |

Отговор: `{ status: 1, obj: CarsSeller }`. Грешка: `_BC_ERR_SELLER_NOT_FOUND`.

### `GET /my_cars_client` 🔒

`{ id, user_id }` — клиентският профил (`id: 0`, ако няма).

### `GET /my_cars_dashboard` 🔒

Продавач:

```json
{ "active_listings": 15, "total_bids": 0, "completed_sales": 0, "total_revenue": 0 }
```

Клиент — статистиката на купувача (`CarsClientDashboard`). Без профил → грешка `_BC_ERR_NO_PROFILE`.

### `GET /my_web_notifications` 🔒

Непрочетените известия, масив от [CarsWebNotification](#159-известие).

### `POST /web_notifications_read` 🔒

| Параметър | Тип | Описание |
|---|---|---|
| `id` | int | една нотификация **или** |
| `all` | 1 | всички непрочетени на потребителя |

Грешки: `_BC_ERR_NOTIFICATION_NOT_FOUND`, `_BC_ERR_NOTIFICATION_NO_ACCESS`.

### `DELETE /my_account` 🔒

Изтрива собствения акаунт (ядрото). Параметър `password` (задължителен).
Ако е продавач — всичките му обяви се спират и профилът се деактивира.

### `GET /cars_sellers` 🔒

Активните търговци за избор при прехвърляне на обява. Само за **активен** търговец; собственият профил не влиза.

| Параметър | Тип | Описание |
|---|---|---|
| `search` | string | по име |

```json
[ { "id": 44, "user_id": 44, "name": "AutoChina Burgas" } ]
```

Грешка: `_BC_ERR_SELLER_NOT_ACTIVE`.

---

## 7. Моите обяви (продавач)

### `GET /my_cars_auctions` 🔒

- **Продавач** → неговите обяви, масив от [CarsAuction](#154-обява) (всички статуси; с `vin`, `hidden`, `reserved`, `admin_notes`).
- `?id=` → една негова обява (грешка `_BC_ERR_AUCTION_NO_ACCESS` за чужда).
- **Клиент** → обявите, по които е наддавал (бид, pre-bid, buy now) — `CarsAuctionFrontend[]`.

### `POST /cars_auctions` 🔒 — създаване / редакция (чернова)

Без `id` → нова обява; с `/cars_auctions/<id>` → редакция. Подават се само полетата за промяна.

Редактира се в статус Чернова (0) и Нужни корекции (2); резервационните — и в Активен (6).
Само **активен** продавач. Обява без ДДС изисква попълнени банкови данни на продавача.

| Параметър | Тип | Описание |
|---|---|---|
| `type` | string | `reservation` (по подразбиране) / `auction` |
| `location` | string | `china` (по подразбиране) / `on_road` / `bg` |
| `title` | string | заглавие |
| `quality` | string | `A` / `B` / `C` / `D` |
| `registered_bg` | 0/1 | регистрирана в БГ |
| `expected_import` | 0/1 | очакван внос |
| `vat_charging` | 0/1 | с начислен ДДС. `0` само ако продавачът има `can_no_vat_charging` |
| `reserved` | 0/1 | маркер „резервирана" |
| `hidden` | 0/1 | свалена от сайта |
| `price` | float | цена (`type=reservation`) |
| `auto_renew` | 0/1 | автоматично подновяване (`type=reservation`) |
| `start_price`, `reserve_price`, `buy_now_price`, `bid_step` | float | (`type=auction`) |
| `prebid_active` | 0/1 | (`type=auction`) |
| `video_url` | string | линк към видео (алтернатива на качено видео) |
| `service_history`, `known_defects`, `modifications`, `extras`, `equipment`, `additional_info` | string | описание |
| `car_info[...]` | | данните на колата: `manufacturer_id`, `model_id`, `variant_id`, `year`, `is_new`, `mileage`, `fuel_id`, `transmission_id`, `drivetrain_id`, `body_type_id`, `engine`, `power_hp`, `battery_kwh`, `ev_range`, `seats`, `doors`, `exterior_color`, `steering` (0 ляв / 1 десен), `city`, `country_id`, `vin` |

`location=bg` → `car_info[city]` е задължително.

Статус: нова резервационна обява става **Активен (6)** веднага, `date_start` = сега, `date_end` = +30 дни.
Редакция на резервационна запазва статуса и датите. Аукционът е Чернова (0).

Отговор: `{ status: 1, obj: CarsAuction }`.

Грешки: `_BC_ERR_AUCTION_NO_ACCESS`, `_BC_ERR_AUCTION_NOT_EDITABLE`, `_BC_ERR_SELLER_NOT_ACTIVE`, `_BC_ERR_SELLER_NO_BANK_INFO`,
`_BC_ERR_SELLER_NO_VAT_CHARGING`, `_BC_ERR_AUCTION_CITY_REQUIRED`.

### `POST /cars_auctions_confirm/<id>` 🔒 — подаване / потвърждение

Проверява задължителните полета и публикува.

- Резервационна в **Активен (6)** → остава активна, срокът (`date_start`/`date_end`) **не се пипа**.
- Резервационна в чернова/корекции → **Активен (6)** за 30 дни.
- Аукцион → **[gz]** **Scheduled (5)** веднага, с автоматично генерирани `date_start` (след 2–5 дни) и `date_end`.

Задължителни за потвърждение **[gz]**: `car_info` — `manufacturer_id`, `model_id`, `year`, `fuel_id`, `transmission_id`,
`drivetrain_id`, `body_type_id`; `price > 0` (резервация) / `start_price > 0` (аукцион); `car_info.city` при `location=bg`.
Пробег, двигател, места, врати, цвят и видео **не са** задължителни (продават се нови коли).

Отговор: `{ status: 1, msg, obj: CarsAuction }`.
Грешки: `_BC_ERR_AUCTION_FIELD_REQUIRED` (`%s` = полето), `_BC_ERR_AUCTION_FIELD_INVALID_NUMBER`, `_BC_ERR_RESERVATION_PRICE_REQUIRED`,
`_BC_ERR_AUCTION_START_PRICE_REQUIRED`, `_BC_ERR_AUCTION_CITY_REQUIRED` + тези от `POST /cars_auctions`.

### Медия на обява 🔒 (само продавачът на обявата)

| Endpoint | Параметри | Описание |
|---|---|---|
| `POST /cars_auctions_images` | `auction_id`, `type` (`exterior`/`interior`/`detail`), файлове `images[]` | качва снимки от един тип |
| `POST /cars_auctions_images_order` | `auction_id`, `image_order[]` (id-та) | подрежда снимките; първата става cover |
| `DELETE /cars_auctions_images/<id>` | | трие снимка (не и в Завършил) |
| `POST /cars_auctions_videos` | `auction_id`, файл `video` | качва видео |
| `DELETE /cars_auctions_videos/<id>` | | трие видео (не и в Завършил) |
| `POST /cars_auctions_report` | `auction_id`, файл `report` | качва репорт (PDF). **[gz]** Отговорът има и `reports_unlocked` — колко платени [заявки за репорт](#12a-репорти-gz) са станали Completed |
| `DELETE /cars_auctions_reports/<id>` | | трие репорт (не и в Активен/Завършил) |
| `POST /cars_auctions_brochure` | `auction_id`, файл `brochure` | брошура (PDF) — една на обява, новата заменя старата |
| `DELETE /cars_auctions_brochure/<id>` | | трие брошурата |

Отговор при качване: `{ status: 1, ... }` със записаните файлове.
Грешки: `_BC_ERR_AUCTION_NO_ACCESS`, `_BC_ERR_IMAGE_TYPE_INVALID`, `_BC_ERR_NO_PHOTOS_UPLOADED`, `_BC_ERR_NO_VIDEO_UPLOADED`,
`_BC_ERR_NO_BROCHURE_UPLOADED`, `_BC_ERR_IMAGES_ORDER_INVALID`, `_BC_ERR_PHOTO_NOT_FOUND`, `_BC_ERR_VIDEO_NOT_FOUND`,
`_BC_ERR_REPORT_NOT_FOUND`, `_BC_ERR_BROCHURE_NOT_FOUND`, `_BC_ERR_AUCTION_CANT_DELETE_PHOTOS` / `_VIDEO` / `_REPORT`.

### Действия по обява 🔒 (само продавачът)

| Endpoint | Параметри | Описание | Отговор |
|---|---|---|---|
| `POST /my_cars_auctions_reserved` | `auction_id`, `reserved` (0/1, задълж.) | маркер „резервирана"; обявата остава активна | `{ status: 1, msg, obj }` |
| `POST /my_cars_auctions_hidden` | `auction_id`, `hidden` (0/1, задълж.) | сваля/връща обявата от сайта; статусът не се пипа | `{ status: 1, msg, obj }` |
| `POST /my_cars_auctions_bump` | `auction_id` | вдига обявата напред в каталога (обновява `date_updated`) | `{ status: 1, msg, obj, date_updated }` |
| `POST /my_cars_auctions_transfer` | `auction_id`, `seller_id` (= user_id на търговеца) | прехвърля обявата на друг активен търговец. Сделките/заявките/огледите остават при стария. Обява със заявка не се мести. Подписът се трие. | `{ status: 1, msg, obj }` |
| `POST /cars_auctions_republish` | `auction_id` | **[gz]** само за обява в Ended (7); копие без бидове, сделки, дати; копието е в **Pending review (1)** — и за резервационни (виж т.16) | `{ status: 1, obj }` |
| `DELETE /cars_auctions/<id>` | | изтрива обява: аукцион — само в 0/2; резервационна — и в 6 | `{ status: 1 }` |

Грешки: `_BC_ERR_AUCTION_NO_ACCESS`, `_BC_ERR_AUCTION_NOT_FOUND`, `_BC_ERR_AUCTION_RESERVED_REQUIRED`, `_BC_ERR_AUCTION_HIDDEN_REQUIRED`,
`_BC_ERR_AUCTION_TRANSFER_SAME_SELLER`, `_BC_ERR_SELLER_NOT_FOUND`, `_BC_ERR_AUCTION_TRANSFER_SELLER_NOT_ACTIVE`,
`_BC_ERR_AUCTION_TRANSFER_NO_VAT_RIGHT`, `_BC_ERR_AUCTION_TRANSFER_HAS_REQUESTS`, `_BC_ERR_REPUBLISH_AUCTION_NOT_FOUND`, `_BC_ERR_AUCTION_REPUBLISH_ONLY_ENDED`,
`_BC_ERR_AUCTION_DELETE_ONLY_DRAFT`.

### Спецификации в каталога 🔒 (активен търговец)

Спецификациите са общи каталожни данни — редактира ги всеки активен търговец.

**`POST /cars_model_specifications`**

| Параметър | Тип | Описание |
|---|---|---|
| `id` | int | редакция на съществуваща спецификация **или** |
| `title` + `model_id` (+ `variant_id`) | | нова; без `variant_id` вариацията се създава от `title` (или се ползва съществуващата със същото име) |
| `year`, `body_type_id`, `fuel_id`, `drivetrain_id`, `transmission_id`, `battery_type_id`, `adas_level_id`, `seats`, `motors` | int | |
| `power_kw`, `torque_nm`, `battery_kwh`, `range_km` | number | |
| `specs` | object / JSON string | `{ секция: { поле: стойност } }` (виж шаблона в т.2) |
| `videos` | array / text | линкове към видео на **модела** (по един на ред) |
| `images[]` / `image` | файл | снимки на **модела** (общи за всички вариации) |

Отговор: `{ status: 1, obj: CarsSpecification }` (с `videos` и `images`).
Грешки: `_BC_ERR_SELLER_NOT_ACTIVE`, `_BC_ERR_SPECIFICATION_NOT_FOUND`, `_BC_ERR_SPECIFICATION_CREATE_FIELDS`, `_BC_ERR_VARIANT_NOT_FOUND`,
`_BC_ERR_SPECIFICATION_MODEL_MISMATCH`, `_BC_ERR_SPECIFICATION_SPECS_INVALID`.

**`DELETE /cars_model_images/<id>`** — трие снимка на модел. Грешки: `_BC_ERR_SELLER_NOT_ACTIVE`, `_BC_ERR_PHOTO_NOT_FOUND`.

---

## 8. Заявки — резервация, покупка, лизинг

Една заявка (`CarsRequest`) с три типа:

| `type` | Действие | Дължимо (`amount`) |
|---|---|---|
| `reservation` | „Резервирай" — кола в Китай / на път | такса 10% от цената |
| `purchase` | **[gz]** вече не се създава — „Купи сега" прави **сделка** (т.11); може да има стари записи | цялата цена |
| `leasing` | кандидатстване за лизинг | 0 |

Статуси на заявка:

| id | | |
|---|---|---|
| 0 | Нова | подадена, нищо не е платено |
| 1 | Частично платена | капаро / такса |
| 2 | Платена | цялата сума |
| 3 | Приключена | колата е предадена |
| 4 | Отказана | |
| 5 | Анулирана | от клиента |
| 6 | Одобрена | само лизинг |

Плащанията са **по банка** — въвежда ги продавачът (или админът). Първото плащане / одобрение сваля обявата (Завършил).

### `POST /cars_reservations` 🔒 / `POST /cars_purchases` 🔒

**[gz]** `POST /cars_purchases` **не** създава заявка: обявата става Ended веднага и се създава **сделка**
(`CarsAuctionDeal`, сума = цената на обявата) — отговор `{ status: 1, obj: CarsAuctionDeal }`, продължение в [т.11](#11-сделки).

Клиентът заявява резервация / покупка. Само за **Активна** резервационна обява, само клиент, не по собствена обява.
Резервацията е за коли извън БГ (`can_reserve`), покупката — за коли в БГ (`can_buy_now`).

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `auction_id` | int | да | |
| `names`, `email`, `phone` | string | не | по подразбиране — от профила |
| `city`, `note` | string | не | |

Огледът се заявява отделно — `POST /cars_inspections`.

Отговор: `{ status: 1, obj: CarsRequest }`.
Грешки: `_BC_ERR_RESERVATION_NOT_FOUND`, `_BC_ERR_RESERVATION_NOT_AVAILABLE`, `_BC_ERR_PURCHASE_NOT_AVAILABLE`,
`_BC_ERR_RESERVATION_OWN`, `_BC_ERR_RESERVATION_NO_CLIENT`.

### `POST /cars_leasing_applications` 🔓 — лизинг (и за гости)

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `auction_id` | int | да | само обява, за която `can_lease = true` (кола в БГ) |
| `names`, `email`, `phone` | string | да | |
| `city`, `note` | string | не | |
| `down_payment_percent` | float | да | |
| `months` | int | да | |
| `residual_percent` | float | не | по подразбиране от схемата |

Нов/употребяван се чете от обявата. Изчислението е на сървъра (виж `GET /cars_leasing_scheme`).
Отговор: `{ status: 1, obj: CarsRequest }`. Грешки: `_BC_ERR_LEASING_FIELD_REQUIRED`, `_BC_ERR_LEASING_EMAIL_INVALID`,
`_BC_ERR_LEASING_NOT_AVAILABLE` + грешките на калкулатора.

### `GET /my_cars_requests` 🔒

Заявките на логнатия — като клиент **и** като продавач. Масив от [CarsRequest](#156-заявка), подреден по `id` DESC. `admin_notes` винаги е `""`.

| Параметър | Тип | Описание |
|---|---|---|
| `type` | string | `reservation` / `purchase` / `leasing` — филтрира **само** частта „като клиент" |
| `archived` | 1 | само архивираните (като продавач) — виж [архив](#архив) |

Съвместимост (стари): `GET /my_cars_reservations` (резервации + покупки), `GET /my_cars_leasing_applications` (лизинг като клиент).

### `POST /my_cars_requests_status` 🔒 (продавач)

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `request_id` | int | да | своя заявка |
| `status_id` | int | да | всеки валиден статус (0 е валидна стойност) |

Отказан лизинг праща мейл на кандидата. Грешки: `_BC_ERR_REQUEST_NO_ACCESS`, `_BC_ERR_REQUEST_STATUS_NOT_ALLOWED`, `_BC_ERR_REQUEST_STATUS_REQUIRED`.

### `POST /my_cars_requests_payments` 🔒 (продавач)

Въвежда получено плащане по своя заявка (резервация/покупка). Сборът движи статуса: частично → платена.

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `request_id` | int | да | |
| `amount` | float | да | > 0 |
| `payment_date` | string | не | `Y-m-d`, по подразбиране днес |

Отговор: `{ status: 1, obj: CarsRequestPayment }`. Грешки: `_BC_ERR_REQUEST_NO_ACCESS`, `_BC_ERR_PAYMENT_NOT_ALLOWED` (лизинг), `_BC_ERR_PAYMENT_AMOUNT_INVALID`.

### `DELETE /my_cars_requests_payments/<id>` 🔒 (продавач)

Изтрива въведено плащане; статусът се преизчислява. Грешки: `_BC_ERR_PAYMENT_NOT_FOUND`, `_BC_ERR_REQUEST_NO_ACCESS`.

### `POST /my_cars_requests_archive` 🔒 (продавач)

Виж [архив](#архив). Параметри: `request_id` (int, задълж.), `archived` (1 архивира / 0 разархивира, по подразбиране 1).
Отговор: `{ status: 1, msg, obj: CarsRequest }`. Грешка: `_BC_ERR_REQUEST_NO_ACCESS`.

### Архив

Огледите, заявките и сделките имат поле **`archived`** (bool).

- **Архивира само продавачът** — по огледи/заявки за своите обяви и по своите сделки.
- Архивираният обект изчезва **само от неговия** списък. **Купувачът го вижда винаги.**
- Админът не архивира; в админ панела се виждат всички.

| Заявка | Частта „като купувач" | Частта „като продавач" |
|---|---|---|
| без `archived` | всички | **само неархивираните** |
| `?archived=1` | *не се връща* | **само архивираните** |

> **Промяна в поведението (2026-10-15):** преди продавачът получаваше всичките си обекти;
> сега архивираните не излизат без `?archived=1`.

---

## 9. Огледи

Купувачът заявява оглед; продавачът го потвърждава. Статуси (`GET /cars_inspections_statuses` 🔓):

| id | title |
|---|---|
| 0 | Pending |
| 1 | Approved |
| 2 | Cancelled by the seller |
| 3 | Cancelled by the buyer |

### `POST /cars_inspections` 🔒

**Без `inspection_id` → нова заявка (купувачът):**

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `auction_id` | int | да | |
| `inspection_date` | string | да | `Y-m-d` |
| `inspection_time` | string | не | `H:i` |
| `names`, `phone_code`, `phone` | string | не | по подразбиране от профила |

**С `inspection_id` → редакция (продавачът по обявата):** `inspection_date`, `inspection_time`, `status_id` (0 / 1).
Преход към Одобрен праща известие на купувача.

Отговор: `{ status: 1, obj: CarsInspection }`.
Грешки: `_BC_ERR_AUCTION_NOT_FOUND`, `_BC_ERR_INSPECTION_DATE_REQUIRED`, `_BC_ERR_INSPECTION_NO_ACCESS`, `_BC_ERR_INSPECTION_STATUS_INVALID`.

### `GET /my_cars_inspections` 🔒

Огледите на логнатия — като купувач и като продавач. Масив от [CarsInspection](#157-оглед), `id` DESC.

| Параметър | Тип | Описание |
|---|---|---|
| `status_id` | int | 0, 1, 2, 3 — без него всички |
| `archived` | 1 | само архивираните (като продавач) |

### `POST /my_cars_inspections_cancel` 🔒

Отказ — и от купувача, и от продавача, и в двата активни статуса. Статусът се избира според това кой отказва; другата страна получава известие.

| Параметър | Тип | Задължителен |
|---|---|---|
| `inspection_id` | int | да |

Грешки: `_BC_ERR_INSPECTION_NO_ACCESS`, `_BC_ERR_INSPECTION_ALREADY_CANCELLED`.

### `POST /my_cars_inspections_archive` 🔒 (продавач)

Виж [архив](#архив). `inspection_id` (int, задълж.), `archived` (1/0, по подразбиране 1).
Отговор: `{ status: 1, msg, obj: CarsInspection }`. Грешка: `_BC_ERR_INSPECTION_NO_ACCESS`.

---

## 10. Аукцион — наддаване, депозити, наблюдавани

Важи за обяви с `type=auction`. За наддаване клиентът трябва да има **активен депозит**.

| Endpoint | Параметри | Описание | Отговор / грешки |
|---|---|---|---|
| `POST /cars_auctions_bids` 🔒 | `auction_id`, `amount` | бид по Активен аукцион | `{ status: 1, obj }`; `_BC_ERR_BID_AUCTION_NOT_LIVE`, `_BC_ERR_BID_OWN_AUCTION`, `_BC_ERR_BID_NO_CLIENT_PROFILE`, `_BC_ERR_BID_NO_DEPOSIT`, `_BC_ERR_BID_AMOUNT_TOO_LOW`, `_BC_ERR_BID_STEP_INVALID`, `_BC_ERR_NOT_AUCTION_TYPE` |
| `POST /cars_auctions_prebids` 🔒 | `auction_id`, `amount` | pre-bid по Предстоящ аукцион (`prebid_active`) | `_BC_ERR_PREBID_NOT_SCHEDULED`, `_BC_ERR_PREBID_NOT_ENABLED`, `_BC_ERR_PREBID_OWN_AUCTION`, `_BC_ERR_PREBID_AMOUNT_TOO_LOW` |
| `GET /my_cars_auctions_prebids?auction_id=` 🔒 | | ⚠ **не работи** — виж [т.16](#16-известни-проблеми) | |
| `POST /cars_auctions_buy_now` 🔒 | `auction_id` | купува аукцион на buy-now цена; аукционът приключва | `{ status: 1, msg }`; `_BC_ERR_BUYNOW_NOT_LIVE`, `_BC_ERR_BUYNOW_NOT_AVAILABLE`, `_BC_ERR_BUYNOW_OWN_AUCTION`, `_BC_ERR_BUYNOW_NO_DEPOSIT` |
| `POST /cars_deposits` 🔒 | | депозит — Stripe checkout | `{ status: 1, deposit_id, checkout_url }` |
| `GET /my_cars_deposits` 🔒 | | депозитите на клиента (статусът се синхронизира със Stripe) | `CarsDeposit[]` |
| `POST /cars_deposits_refund` 🔒 | `deposit_id` | заявка за връщане (само Активен депозит) | `_BC_ERR_DEPOSIT_NOT_FOUND`, `_BC_ERR_DEPOSIT_NO_ACCESS`, `_BC_ERR_DEPOSIT_REFUND_ONLY_ACTIVE`, `_BC_ERR_DEPOSIT_REFUND_PENDING_EXISTS` |
| `POST /cars_auctions_watchlist` 🔒 | `auction_id` | добавя в наблюдавани (само публична обява) | `_BC_ERR_WATCHLIST_AUCTION_NOT_FOUND`, `_BC_ERR_WATCHLIST_ALREADY_ADDED` |
| `DELETE /cars_auctions_watchlist?auction_id=` 🔒 | | маха от наблюдавани | `_BC_ERR_WATCHLIST_NOT_IN_LIST` |
| `GET /my_cars_auctions_watchlist` 🔒 | | наблюдаваните обяви | `CarsAuction[]` |

Статуси на депозит: 1 Изчакващ, 2 Активен, 3 Блокиран, 4 Върнат, 5 Конфискуван.

---

## 11. Сделки

Сделка (`CarsAuctionDeal`) се създава автоматично при приключил аукцион с победител или при buy now.

Статуси на сделка:

| id | |
|---|---|
| 1 | Очаква контраоферта |
| 3 | Контраофертата е отказана |
| 4 | Начална |
| 5 | Потвърдена |
| 6 | Платена комисионна |
| 7 | Платена сума |
| 8 | Предадена (двете страни потвърдили) |
| 10 | Приключена (от админ) |
| 20 | Анулирана |
| 21 | Анулирана — купувачът не потвърди |
| 22 | Анулирана — неплатена комисионна |
| 23 | Анулирана — неплатена сума |

### `GET /my_deals` 🔒

Сделките на логнатия — като купувач и като продавач. Масив от [CarsAuctionDeal](#158-сделка).

| Параметър | Тип | Описание |
|---|---|---|
| `id` | int | една сделка (само за участник), с пълен VIN; **не се влияе** от архива |
| `archived` | 1 | само архивираните (като продавач) |

Грешки: `_BC_ERR_DEAL_NOT_FOUND`, `_BC_ERR_DEAL_NO_ACCESS`.

### `GET /my_deals_counter_offers?deal_id=` 🔒

Контраофертите по сделка (само участник): масив от `{ id, deal_id, amount, note, status (pending|accepted|rejected|expired), expires_at, responded_at, ... }`.

### `GET /my_deal_payment?deal_id=` 🔒

Плащането на комисионната по сделка: `{ id, deal_id, buyer_id, amount, stripe_intent_id, date_created, date_updated }` (`id: 0`, ако няма).

### Действия по сделка 🔒

Всички приемат `deal_id` (int, задължителен).

| Endpoint | Кой | Описание | Грешки |
|---|---|---|---|
| `POST /cars_auctions_deals_offer` | редуват се | оферта с цена; продавачът започва. + `amount` (задълж.), `note` | `_BC_ERR_OFFER_MAX_ROUNDS`, `_BC_ERR_OFFER_NOT_YOUR_TURN`, `_BC_ERR_OFFER_AMOUNT_REQUIRED`, `_BC_ERR_OFFER_AMOUNT_TOO_LOW` |
| `POST /cars_auctions_deals_accept` | чийто ред е | приема активната оферта (без оферта — продавачът приема top bid-а) | `_BC_ERR_OFFER_NOT_TURN`, `_BC_ERR_OFFER_NONE_ACTIVE`, `_BC_ERR_OFFER_EXPIRED` |
| `POST /cars_auctions_deals_reject` | чийто ред е | отказва — сделката се затваря | `_BC_ERR_OFFER_NOT_TURN` |
| `POST /cars_auctions_deals_confirm` | двете страни | потвърждава Начална сделка в срока (24 ч) | `_BC_ERR_DEAL_NOT_INITIAL`, `_BC_ERR_DEAL_CONFIRM_DEADLINE`, `_BC_ERR_DEAL_ALREADY_CONFIRMED`, `_BC_ERR_DEAL_NO_ACCESS` |
| `POST /cars_auctions_deals_cancel` | двете страни | отказ в статуси 1, 4, 5. Купувач → депозитът се конфискува; продавач → депозитът се отключва | `_BC_ERR_DEAL_NOT_CANCELLABLE` |
| `POST /cars_auction_payments` | купувачът | плащане на комисионната (Stripe) — сделката трябва да е Потвърдена | `{ status: 1, checkout_url }`; `_BC_ERR_DEAL_NOT_CONFIRMED`, `_BC_ERR_COMMISSION_ALREADY_PAID` |
| `POST /cars_auctions_deals_payment_received` | продавачът | потвърждава получено плащане | `_BC_ERR_DEAL_NOT_COMMISSIN_PAYED`, `_BC_ERR_DEAL_SELLER_PAYMENT_ADMIN` |
| `POST /cars_auctions_deals_handover` | двете страни | продавачът — предал, купувачът — приел. При двете → 8 | `_BC_ERR_DEAL_NOT_AUCTION_PAYED`, `_BC_ERR_DEAL_ALREADY_HANDOVER_BUYER` / `_SELLER` |
| `POST /my_deals_archive` | продавачът | [архив](#архив) + `archived` (1/0, по подразбиране 1) | `_BC_ERR_DEAL_NOT_FOUND`, `_BC_ERR_DEAL_NO_ACCESS` |

Всички връщат `{ status: 1, ... }`; навсякъде е възможна и `_BC_ERR_DEAL_NOT_FOUND` / `_BC_ERR_DEAL_NO_ACCESS`.

---

## 12. Калкулатори и справочни данни

### `GET /cars_calculator` 🔓 — крайна цена с доставка и мито

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `auction_id` | int | да | обявата (трябва да има страна на произход) |
| `port_id` | int | да | `GET /cars_shipping_ports` |
| `tax_id` | int | да | `GET /cars_customs_taxes` |
| `custom_amount` | float | не | **[gz]** сметката тръгва от тази сума вместо от цената на обявата (`final_price.custom_price = true`) |

```json
{
  "final_price": {
    "lot_price": 20000, "custom_price": false, "auction_fee": 800, "fee_percent": 4,
    "shipping": 100, "total": 20900,
    "port": { "id": 6, "city": "Burgas", "country_code": "BG", "title": "Burgas, Bulgaria" },
    "country": { "id": 102, "code": "CN", "title": "China" },
    "error": ""
  },
  "customs": {
    "show": true, "customs_value": 20900, "tax_rate": 22, "tax_amount": 4598,
    "vat_rate": 20, "vat_amount": 5099.6, "total": 9697.6,
    "tax": { "id": 3, "title": "Truck", "rate": 22 },
    "error": ""
  },
  "error": ""
}
```

Грешка (напр. обява без страна на произход): `{ status: 0, error }`.

### `GET /cars_leasing_scheme` 🔓

Схемата на лизинговия калкулатор — фронтът смята вноските локално; авторитетното изчисление е при кандидатстване.

```json
{
  "interest": { "new": 7.95, "used": 8.95 },
  "min_months": 12, "max_months": 60, "default_months": 60, "months_step": 1,
  "min_down_payment_percent": 10, "max_down_payment_percent": 60, "default_down_payment_percent": 30,
  "down_payment_step": 1,
  "down_payment_tiers": [ { "price_from": 100000, "min_percent": 50 } ],
  "min_residual_percent": 10, "max_residual_percent": 40, "default_residual_percent": 20, "residual_step": 1,
  "min_price": 5000, "max_price": 250000,
  "examination_fee_percent": 1.5, "vat_divider": 1.2
}
```

### Справочни

| Endpoint | Връща |
|---|---|
| `GET /cars_shipping_ports` 🔓 | `[{ id, city, country_code, sort_order, status_id, title }]` |
| `GET /cars_customs_taxes` 🔓 | `[{ id, title, rate, sort, status_id }]` — ставки на мито по вид МПС |
| `GET /cars_platform_bankinfo` 🔒 | банковата сметка на платформата: `{ id, company_name, company_reg_no, company_address, bank_name, iban, names, country, bank_address, swift, bank_code, currencies }` |

---

## 12a. Репорти [gz]

Купувачът поисква репорт (история на колата) по обява и плаща **в крипто директно на продавача**
(`seller.usdt_trc20` / `seller.usdt_erc20`) — платформата не следи плащането. Репортът е **един за обявата**
(`POST /cars_auctions_report`); щом е качен, всички платени заявки по обявата стават Completed.

Статуси: 0 New (чака плащане) · 1 Paid · 2 Completed (файлът е достъпен) · 3 Cancelled.

| Endpoint | Кой | Параметри | Описание |
|---|---|---|---|
| `POST /cars_reports` 🔒 | купувач | `auction_id` | заявява репорт; една заявка на купувач и обява |
| `POST /cars_reports_paid` 🔒 | продавач | `request_id` | потвърждава плащането; ако репортът вече е качен → направо Completed |
| `POST /cars_reports_cancel` 🔒 | продавач | `request_id` | отказва заявката |
| `GET /my_cars_reports` 🔒 | купувач | | неговите заявки; `report` е попълнен само при Completed |
| `GET /my_cars_reports_seller` 🔒 | продавач | `status_id` (по избор) | заявките по неговите обяви |

Отговорите на POST: `{ status: 1, obj: CarsReportRequest }`.

**CarsReportRequest** — `id`, `auction_id`, `buyer_id`, `seller_id`, `status_id` int, `amount` float (цената към момента на заявката),
`date_created`, `date_updated`, `status` `{ id, title, color }`, `auction` CarsAuction, `report` File (празен, докато не е Completed).

Грешки: `_BC_ERR_AUCTION_NOT_FOUND`, `_BC_ERR_REPORT_OWN_AUCTION`, `_BC_ERR_REPORT_ALREADY_REQUESTED` (с `obj` — съществуващата заявка),
`_BC_ERR_REPORT_NO_ACCESS`, `_BC_ERR_REPORT_NOT_NEW`, `_BC_ERR_REPORT_ALREADY_COMPLETED`.

---

## 13. Въпроси и отговори (Q&A)

| Endpoint | Параметри | Описание |
|---|---|---|
| `POST /qa` 🔒 | `product_id` (= auction_id), `text` | въпрос по обява (чака модерация, `status_id = 0`) |
| `POST /qa_answer` 🔒 | `qa_id`, `text` | отговор от продавача на обявата |

Четенето е през ядрените Q&A endpoint-и.

---

## 14. Служебни — не за фронта

| Endpoint | Описание |
|---|---|
| `POST /stripe_webhook`, `POST /stripe_webhook_deposit` | Stripe webhooks (`payload`, `stripe_signature`) |
| `POST /cron_test`, `POST /cron_calculate_finalprice` | ръчно пускане на крона — само за разработка |
| `GET /cars_cron_5m` | **[gz]** ръчно пуска петминутния крон (без автентикация). Не се вика от фронта — паралелно пускане може да дублира сделки |

---

## 15. Схеми

Типовете са по реалните отговори. `object` = вложен обект, `X[]` = масив.

### 15.1. Каталог автомобили

**CarsManufacturer** — `id` int, `title` string, `status_id` int, `sort` int, `top` bool, `logo` Image.

**CarsModel** — `id` int, `manufacturer_id` int, `title` string, `status_id` int, `sort` int, `videos_json` string,
`videos` string[] (линкове), `manufacturer` CarsManufacturer, `image` Image (първата), `images` Image[].

**CarsModelVariant** — `id` int, `model_id` int, `title` string, `year` int, `status_id` int, `sort` int.

**CarsSpecification**

| Поле | Тип | |
|---|---|---|
| `id`, `model_id`, `variant_id` | int | |
| `title` | string | името на вариацията |
| `year` | int | |
| `body_type_id`, `fuel_id`, `drivetrain_id`, `transmission_id`, `battery_type_id`, `adas_level_id` | int | |
| `power_kw`, `torque_nm`, `range_km`, `seats`, `motors` | int | |
| `battery_kwh` | float | |
| `specs` | string | JSON `{ секция: { поле: стойност } }` |
| `views_count` | int | |
| `videos` | string[] | видеата на модела |
| `images` | Image[] | снимките на модела |
| `status_id`, `sort` | int | |

### 15.2. Каталог мотоциклети

**MotoManufacturer** — `id`, `title`, `title_cn`, `source_id` (string), `status_id`, `sort`, `top` bool, `logo` Image.

**MotoModel** — `id`, `manufacturer_id`, `title`, `title_cn`, `source_id`, `type_id`, `energy_id`, `status_id`, `sort`, `manufacturer` MotoManufacturer, `image` Image.

**MotoModelVariant** — `id`, `model_id`, `title`, `title_cn`, `source_id`, `year`, `price`, `status_id`, `sort`.

**MotoSpecification** — `id`, `model_id`, `variant_id`, `title`, `year`, `type_id`, `category_id`, `energy_id`, `origin`, `license` string,
`price`, `displacement_cc`, `cylinders`, `cooling_id`, `torque_nm`, `emission_std` string, `fuel_tank_l`, `battery_type_id`,
`voltage_v`, `capacity_ah`, `battery_kwh` float, `range_km`, `range_const_km`, `charge_time_h`, `motor_layout` string,
`power_kw` float, `power_ps`, `drive_id`, `top_speed_kmh`, `seat_height_mm`, `wheelbase_mm`, `length_mm`, `width_mm`, `height_mm`,
`weight_kg`, `abs` string, `cbs` string, `specs` string (JSON), `status_id`, `sort`,
и вложени `type`, `category`, `energy`, `battery_type`, `cooling`, `drive` — MotoNomenclature.

### 15.3. Каталог камиони

**TruckManufacturer** — като MotoManufacturer.

**TruckModel** (серия) — `id`, `manufacturer_id`, `title`, `title_cn`, `source_id`, `category_id`, `tonnage_id`, `sale_status` int, `status_id`, `sort`, `manufacturer`, `image` Image.

**TruckModelVariant** — `id`, `model_id`, `title`, `title_cn`, `source_id`, `announcement_code` string, `discontinued` bool, `status_id`, `sort`.

**TruckSpecification** — `id`, `model_id`, `variant_id`, `title`, `category_id`, `tonnage_id`, `energy_id`, `emission_id`, `drive_form_id`,
`cab_id`, `body_type_id`, `announcement_code`, `origin` string, `wheelbase_mm`, `length_mm`, `width_mm`, `height_mm`, `curb_weight_kg`,
`gross_weight_kg`, `payload_kg`, `front_axle_kg`, `rear_axle_kg`, `cargo_length_mm`, `cargo_width_mm`, `cargo_height_mm`,
`engine_brand`, `engine_model` string, `displacement_l`, `power_kw` float, `power_hp`, `torque_nm`, `cylinders`, `fuel_cn` string,
`transmission_model` string, `gears_forward`, `battery_type_id`, `battery_brand` string, `battery_kwh`, `range_km`, `motor_model` string,
`motor_power_kw`, `seats`, `seat_rows`, `top_speed_kmh`, `tire_size` string, `tire_count`, `abs` string, `specs` string (JSON), `status_id`, `sort`,
и вложени `category`, `tonnage`, `energy`, `emission`, `drive_form`, `cab`, `body_type`, `battery_type` — TruckNomenclature.

### 15.4. Обява

**CarsAuctionFrontend** — публичният изглед (каталог, `GET /cars_auctions`). Без `vin`, `reserve_price`, `admin_notes`, `hidden`.

| Поле | Тип | Описание |
|---|---|---|
| `id`, `user_id` | int | `user_id` = продавачът |
| `title` | string | |
| `status_id` | int | виж статусите в т.5 |
| `type` | string | `reservation` / `auction` |
| `location` | string | `china` (в Китай) / `on_road` (на път) / `bg` (в БГ) |
| `quality` | string | |
| `registered_bg`, `expected_import`, `vat_charging`, `reserved`, `featured` | bool | |
| `price` | float | цена (резервационна обява) |
| `reservation_fee`, `reservation_fee_percent` | float | таксата за резервация (10%) |
| `can_reserve`, `can_buy_now`, `can_lease` | bool | кои бутони да се покажат |
| `auto_renew` | bool | |
| `start_price`, `buy_now_price`, `bid_step` | float | аукцион |
| `buy_now_active`, `prebid_active` | bool | аукцион |
| `bids` | Bid[] | |
| `bids_count` | int | |
| `current_bid` | float | |
| `slug`, `url` | string | `url` — пълен адрес на обявата в сайта |
| `calls_count` | int | |
| `date_start`, `date_end` | string | `Y-m-d H:i:s` |
| `date_end_label` | string | текст за оставащото време |
| `service_history`, `known_defects`, `modifications`, `extras`, `equipment`, `additional_info` | string | |
| `admin_note` | string | публична бележка от админа |
| `vin_masked` | string | |
| `video_url` | string | |
| `status` | `{ id, title, color }` | |
| `car_info` | CarsInfo | виж долу |
| `seller` | CarsSeller | |
| `report`, `brochure` | File | |
| `images_exterior`, `images_interior`, `images_detail` | Image[] | (+ `type`) |
| `videos` | File[] | |
| `date_created` | string | |

**CarsAuction** — пълният изглед (собственик / участник: `my_cars_auctions`, `obj` в отговорите, `auction` в заявки/огледи/сделки).
Полетата на Frontend без изчислените (`reservation_fee*`, `can_*`, `date_end_label`, `vin_masked`, `admin_note`), плюс:
`parent_auction_id` int, `hidden` bool, `reserve_price` float, `auto_renew_count` int, `admin_notes` string,
`forced_winner_id` int, `vin` string, `ending_soon_notified` bool, `extensions_count` int, `prebids_count` int,
`date_updated` string, `seller_signature` Image (+ `user_id`).

**CarsInfo** (`car_info`)

| Поле | Тип | |
|---|---|---|
| `id`, `auction_id` | int | |
| `manufacturer_id`, `model_id`, `variant_id` | int | |
| `year`, `mileage` | int | |
| `is_new` | bool | |
| `fuel_id`, `transmission_id`, `drivetrain_id`, `body_type_id` | int | |
| `engine` | string | |
| `power_hp`, `ev_range`, `seats`, `doors` | int | |
| `battery_kwh` | float | |
| `steering` | int | 0 ляв / 1 десен волан |
| `city` | string | |
| `country_id` | int | страна на произход |
| `exterior_color` | string | |
| `manufacturer` | CarsManufacturer | |
| `model` | CarsModel | |
| `fuel`, `transmission`, `drivetrain`, `body_type` | Nomenclature | |
| `country` | `{ id, region_id, code, title, vat, flag, iso3, phoneCode }` | |

### 15.5. Продавач и клиент

**CarsSeller**

| Поле | Тип | |
|---|---|---|
| `id`, `user_id` | int | `id` = `user_id` |
| `status_id` | int | 0 Нов, 1 Активен, 2 Неактивен |
| `payment_status_id` | int | |
| `show_bank_info` | bool | показва ли се банковата сметка в обявите |
| `email_notification` | bool | |
| `can_no_vat_charging` | bool | може да публикува без ДДС (задава админът) |
| `nolink`, `whatsapp`, `viber`, `telegram` | string | |
| `usdt_trc20`, `usdt_erc20` | string | **[gz]** крипто портфейли (плащане на репорти) |
| `phone_code`, `phone`, `phone_code2`, `phone2` | string | от потребителя |
| `date_created` | string | |
| `auctions_count`, `auctions_active_count`, `auctions_draft_count` | int | |
| `bank_info` | `{ id, user_id, bank_name, iban, names }` | |
| `user_company` | object | **[gz]** фирмата на потребителя |
| `status` | `{ id, title, color }` | |

**CarsClient** — `{ id, user_id }`.

### 15.6. Заявка

**CarsRequest**

| Поле | Тип | Описание |
|---|---|---|
| `id` | int | |
| `type` | string | `reservation` / `purchase` / `leasing` |
| `auction_id` | int | |
| `user_id` | int | клиентът (0 = гост при лизинг) |
| `seller_id` | int | |
| `names`, `email`, `phone`, `city`, `note` | string | |
| `total_price` | float | цената към момента |
| `amount` | float | дължимо |
| `payment_method` | string | `""` / `bank` / `stripe` |
| `fee_intent_id` | string | |
| `down_payment_percent`, `residual_percent` | float | лизинг |
| `is_new` | bool | лизинг |
| `months` | int | лизинг |
| `monthly_installment` | float | лизинг, без ДДС |
| `status_id` | int | виж т.8 |
| `admin_notes` | string | винаги `""` към фронта |
| `archived` | bool | архивирана от продавача |
| `date_created`, `date_paid` | string / null | |
| `status` | `{ id, title, color }` | |
| `auction` | CarsAuction | |
| `payments` | `[{ id, request_id, amount, payment_date, user_id, date_created }]` | |
| `paid_amount`, `due_amount` | float | платено / остатък |

### 15.7. Оглед

**CarsInspection**

| Поле | Тип | |
|---|---|---|
| `id`, `auction_id`, `buyer_id`, `seller_id` | int | |
| `names`, `phone_code`, `phone` | string | |
| `inspection_date` | string | `Y-m-d` |
| `inspection_time` | string | `H:i` или `""` |
| `status_id` | int | виж т.9 |
| `archived` | bool | архивиран от продавача |
| `date_created`, `date_updated` | string | |
| `status` | `{ id, title, color }` | |
| `auction` | CarsAuction | |

### 15.8. Сделка

**CarsAuctionDeal**

| Поле | Тип | |
|---|---|---|
| `id`, `auction_id`, `buyer_id`, `seller_id` | int | |
| `status_id` | int | виж т.11 |
| `final_amount` | float | |
| `deposit_id` | int | |
| `buyer_confirmed`, `seller_confirmed` | bool | |
| `confirmation_deadline` | string | |
| `seller_handover`, `buyer_handover` | bool | |
| `seller_direct_payment`, `seller_received_payment` | bool | |
| `archived` | bool | архивирана от продавача |
| `vin` | string | |
| `date_created`, `date_updated` | string | |
| `phone_seller`, `phone_client` | string | |
| `seller_names` | string | |
| `auction` | CarsAuction | |
| `status` | `{ id, title, color }` | |
| `commission` | `{ id, deal_id, buyer_id, amount, stripe_intent_id, date_created, date_updated }` | |
| `counter_offers` | CounterOffer[] | |
| `client_info` | `{ id, email, first_name, middle_name, last_name, names, date_created }` | |
| `bank_info` | object | **[gz]** сметката за плащане на сделката (на продавача или на платформата): `{ is_platform, company_name, company_reg_no, company_address, names, account_number, bank_name, country, bank_address, swift, bank_code, currencies }` |

### 15.9. Известие

**CarsWebNotification** — `id` int, `user_id` int, `type` string (напр. `seller_auction_approved`), `message` string,
`url` string (относителен път във фронта), `auction_id` int, `deal_id` int, `is_read` bool, `time_left` string, `date_created` string.

---

## 16. Известни проблеми

| Endpoint | Проблем |
|---|---|
| `GET /my_cars_auctions_prebids` | PHP fatal error — `CarsAuctionPrebid::get_all_byAuctionAndUser()` не съществува. Не го ползвай, докато не бъде оправен. |
| `GET /cars_model_specifications` без филтър | връща всички спецификации без пагинация (~90 MB). Винаги подавай `variant_id`, `model_id` или `id`. |
| `GET /cars_model_variants`, `/moto_model_variants` без филтър | връщат всички вариации (хиляди). Подавай `model_id`. |
| `GET /cars_auctions` | **[gz]** не скрива обявите, свалени от продавача (`hidden`). Подавай `hidden=0`. |
| `POST /cars_auctions_republish` | **[gz]** копието на резервационна обява отива в Pending review (1), въпреки че резервационните иначе не минават през преглед. |
