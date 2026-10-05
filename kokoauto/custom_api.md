# KokoAuto — Public GET API

Документация за публичните endpoint-и от `api/front/Get.php` + схеми на върнатите обекти. Използва се от АИ за изграждане на фронтенд.

> Всички endpoint-и са **GET**. Не изискват auth (за разлика от admin endpoint-ите).
> Source: `projects/kokoauto/api/front/Get.php` (в devcore repo-то)

---

## 1. Общи конвенции

### 1.1. Идентифициране на endpoint-а
Името на PHP метода = path-ът на endpoint-а. Пример: `kokoauto_car_makers()` → `GET /kokoauto_car_makers`.

### 1.2. Език и `title` / `description`
Полетата `title` и `description` идват от Content системата (i18n) и се join-ват автоматично към резултата като отделни колони. **НЕ** са в `DB_COLUMNS`, но винаги ги има в JSON-а.

### 1.3. Общи query параметри (валидни за всички endpoint-и)

#### Филтри
| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един обект (single, не масив). Прескача всички други филтри. |
| `{property}` | mixed | Филтър по която и да е public property от обекта. Стрингове → `LIKE %v%`. Числа → `=`. Виж секция Schemas за списък. |
| `title` | string | LIKE търсене по title (минава през Content). |
| `only_ids` | int\|int[] | `WHERE id IN (...)`. Подавай като CSV или повтарящ се query param. |
| `date_created_from` | date | `date_created > DATE(v)` |
| `date_created_to` | date | `date_created < DATE(v)` |

#### Пагинация (само за endpoint-и, които ползват `get_all_pagination`)
| Param | Тип | Default | Описание |
|-------|-----|---------|----------|
| `page` | int | 1 | Номер на страница (1-based). |
| `limit` | int | server default | Брой записи на страница. |
| `no_limit` | bool | — | Ако е истина → връща всички (limit = 999999999). |
| `no_pagination` | bool | — | Връща плосък масив без `pagination` обвивка. |

### 1.4. Формат на отговора

#### Single (`?id=N`)
Връща обекта директно като JSON:
```json
{ "id": 5, "title": "BMW", "...": "..." }
```

#### List с пагинация (`get_all_pagination`)
```json
{
  "pagination": {
    "total_results": 123,
    "total_pages": 7,
    "current_page": 1,
    "results_per_page": 20
  },
  "result": [ { "id": 1, "...": "..." }, { "id": 2, "...": "..." } ]
}
```

#### List без пагинация (`get_all`)
Плосък масив:
```json
[ { "id": 1, "...": "..." }, { "id": 2, "...": "..." } ]
```

---

## 2. Endpoints

### 2.1. `GET /kokoauto_car_makers`
Марки автомобили (BMW, Toyota, ...).

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един `KokoAutoCarMaker`. |
| (стандартни филтри + пагинация) | | |

**Връща:** [`KokoAutoCarMaker`](#kokoautocarmaker) (single) или paginated list.

---

### 2.2. `GET /kokoauto_car_models`
Модели автомобили (3 Series, Corolla, ...). Винаги обвързани с марка.

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един `KokoAutoCarModel`. |
| `maker_id` | int | Филтър по марка. |
| `type_id` | int | Филтър по тип марка. |
| (стандартни филтри + пагинация) | | |

**Връща:** [`KokoAutoCarModel`](#kokoautocarmodel) (single) или paginated list.

---

### 2.3. `GET /kokoauto_cars`
Конкретна конфигурация/вариант на автомобил (двигател, купе, мощност, години).

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща една `KokoAutoCar`. |
| `model_id` | int | Филтър по модел. |
| `engineType_id`, `fuelType_id`, `driveType_id`, `bodyType_id`, `brakeType_id`, `transmType_id`, `axleConfig_id`, `type_id` | int | Филтри по съответните типове. |
| (стандартни филтри + пагинация) | | |

**Връща:** [`KokoAutoCar`](#kokoautocar) (single) или paginated list.

---

### 2.4. `GET /kokoauto_engine_types`
Типове двигател.

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един обект. |

**Връща:** [`KokoAutoEngineType`](#kokoautoenginetype) (single) или **плосък масив** (без пагинация).

---

### 2.5. `GET /kokoauto_fuel_types`
Типове гориво.

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един обект. |

**Връща:** [`KokoAutoFuelType`](#kokoautofueltype) (single) или плосък масив.

---

### 2.6. `GET /kokoauto_drive_types`
Типове задвижване (FWD, RWD, AWD, 4WD, ...).

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един обект. |

**Връща:** [`KokoAutoDriveType`](#kokoautodrivetype) (single) или плосък масив.

---

### 2.7. `GET /kokoauto_body_types`
Типове купе (седан, комби, хечбек, купе, SUV, ...).

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един обект. |

**Връща:** [`KokoAutoBodyType`](#kokoautobodytype) (single) или плосък масив.

---

### 2.8. `GET /kokoauto_brake_types`
Типове спирачна система.

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един обект. |

**Връща:** [`KokoAutoBrakeType`](#kokoautobraketype) (single) или плосък масив.

---

### 2.9. `GET /kokoauto_transm_types`
Типове трансмисия (ръчна, автоматична, CVT, DSG, ...).

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един обект. |

**Връща:** [`KokoAutoTransmType`](#kokoautotransmtype) (single) или плосък масив.

---

### 2.10. `GET /kokoauto_car_maker_types`
Типове марка (автомобил, товарен/комерсиален, мотор, ...).

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един обект. |

**Връща:** [`KokoAutoCarMakerType`](#kokoautocarmakertype) (single) или плосък масив.

---

### 2.11. `GET /kokoauto_axle_configs`
Конфигурации на осите (4x2, 6x4, 8x8/4, ...).

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща един обект. |

**Връща:** [`KokoAutoAxleConfig`](#kokoautoaxleconfig) (single) или плосък масив.

---

### 2.12. `GET /kokoauto_cars_categories`
Йерархична категоризация на автомобили (групи системи/възли: „Пневматична система", „Спирачна система", ...).

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща една категория. |
| `parent_id` | int | **Специален режим** — извиква външното Parts API (`partsapi.autochina.mk` чрез [`KokoAutoPartsApi::getSearchTree`](#kokoautopartsapi-външно-api)). Изисква и `car_id`. Връща поддърво за категория + конкретен автомобил. |
| `car_id` | int | Задължителен ако е подаден `parent_id`. Може и самостоятелно като филтър — тогава JOIN-ва към `kokoauto_cars_categories_href` и връща категориите за този автомобил. |
| `type_id` | int | **Задължителен** в стандартния режим (без `parent_id`). |

**Връща:** [`KokoAutoCarsCategories`](#kokoautocarscategories) (single) или плосък масив.

> Бележка: при `parent_id` обектите идват от външен източник и `id`-тата им са `nodeId` от Parts API-то, а `title` идва от `nodeName`. Полета `date_created`/`date_updated` отсъстват.

---

### 2.13. `GET /brands` (не е част от KokoAuto)
Endpoint от `Brands` класа на core-а. Включен е в Get.php, но не е част от KokoAuto домейна. Не го документираме тук.

---

## 3. Schemas

> Конвенции:
> - `int` / `string` / `float` / `bool` — скалари.
> - `datetime` — стринг във формат `YYYY-MM-DD HH:MM:SS`.
> - `date` — `YYYY-MM-DD`.
> - `Type[]` — масив от обекти от тип `Type`.
> - Полетата маркирани с **(eager)** идват автоматично при single fetch (`?id=N` или вътрешен `::get()`). При list endpoint-ите (`get_all` / `get_all_pagination`) `loadObject` НЕ се извиква, така че eager nested обектите **не са** в paginated отговора — ползвай отделни заявки.
> - `title` / `description` идват от Content системата (i18n).

---

### KokoAutoCarMaker
Марки автомобили. Table: `kokoauto_car_makers`.

```ts
{
  id:            int,
  external_id:   int,             // ID от външна система (ако е импортиран)
  title:         string,          // i18n през Content
  description:   string,          // i18n през Content
  date_created:  datetime,
  date_updated:  datetime,

  types:         KokoAutoCarMakerType[]   // (eager) many-to-many през kokoauto_car_makers_types_href
}
```

---

### KokoAutoCarModel
Модели автомобили. Винаги обвързан с марка. Table: `kokoauto_car_models`.

```ts
{
  id:            int,
  external_id:   int,
  maker_id:      int,             // FK → KokoAutoCarMaker
  type_id:       int,             // FK → KokoAutoCarMakerType
  title:         string,
  description:   string,
  yearFrom:      int|null,        // годишен диапазон (0/null = без ограничение)
  yearTo:        int|null,
  date_created:  datetime,
  date_updated:  datetime,

  maker:         KokoAutoCarMaker // (eager)
}
```

---

### KokoAutoCar
Конкретна конфигурация/вариант на автомобил. Table: `kokoauto_cars`.

```ts
{
  id:             int,
  external_id:    int,
  model_id:       int,            // FK → KokoAutoCarModel
  type_id:        int,            // FK → KokoAutoCarMakerType
  title:          string,
  description:    string,

  // Период на производство
  yearFrom:       int|null,
  yearTo:         int|null,

  // Мощност
  kw:             int,            // долна граница
  kwTo:           int,            // горна граница (0 = без диапазон)
  hp:             int,
  hpTo:           int,

  // Двигател
  ccm:            int,            // обем на двигателя в куб.см.
  ccmTax:         int,            // данъчна обемна стойност
  cylinders:      int,
  valves:         int,

  // Каросерия
  doors:          int,
  tankCapacity:   int,            // литри
  tonnage:        float,          // тонаж (за товарни)

  // FK-та към типове
  axleConfig_id:  int,
  engineType_id:  int,
  fuelType_id:    int,
  driveType_id:   int,
  bodyType_id:    int,
  brakeType_id:   int,
  transmType_id:  int,

  date_created:   datetime,
  date_updated:   datetime,

  // Eager nested обекти
  model:          KokoAutoCarModel,        // (eager)
  engineType:     KokoAutoEngineType,      // (eager)
  fuelType:       KokoAutoFuelType,        // (eager)
  driveType:      KokoAutoDriveType,       // (eager)
  bodyType:       KokoAutoBodyType,        // (eager)
  brakeType:      KokoAutoBrakeType,       // (eager)
  transmType:     KokoAutoTransmType,      // (eager)
  axleConfig:     KokoAutoAxleConfig       // (eager)
}
```

---

### KokoAutoEngineType
Тип двигател. Table: `kokoauto_engine_types`.

```ts
{
  id:           int,
  title:        string,
  description:  string,
  date_created: datetime,
  date_updated: datetime
}
```

---

### KokoAutoFuelType
Тип гориво. Table: `kokoauto_fuel_types`.

```ts
{
  id:           int,
  title:        string,
  description:  string,
  date_created: datetime,
  date_updated: datetime
}
```

---

### KokoAutoDriveType
Тип задвижване (FWD, RWD, AWD, 4WD, ...). Table: `kokoauto_drive_types`.

```ts
{
  id:           int,
  title:        string,
  description:  string,
  date_created: datetime,
  date_updated: datetime
}
```

---

### KokoAutoBodyType
Тип купе (седан, комби, хечбек, купе, SUV, ...). Table: `kokoauto_body_types`.

```ts
{
  id:           int,
  title:        string,
  description:  string,
  date_created: datetime,
  date_updated: datetime
}
```

---

### KokoAutoBrakeType
Тип спирачна система. Table: `kokoauto_brake_types`.

```ts
{
  id:           int,
  title:        string,
  description:  string,
  date_created: datetime,
  date_updated: datetime
}
```

---

### KokoAutoTransmType
Тип трансмисия (ръчна, автоматична, CVT, DSG, ...). Table: `kokoauto_transm_types`.

```ts
{
  id:           int,
  title:        string,
  description:  string,
  date_created: datetime,
  date_updated: datetime
}
```

---

### KokoAutoCarMakerType
Тип марка (автомобил, товарен/комерсиален, мотор, ...). Table: `kokoauto_car_maker_types`.

Един `KokoAutoCarMaker` може да има няколко типа чрез junction `kokoauto_car_makers_types_href`.

```ts
{
  id:           int,
  title:        string,
  description:  string,
  date_created: datetime,
  date_updated: datetime
}
```

---

### KokoAutoAxleConfig
Конфигурация на осите (4x2, 6x4, 8x8/4, ...). Table: `kokoauto_axle_configs`.

```ts
{
  id:           int,
  title:        string,
  description:  string,
  date_created: datetime,
  date_updated: datetime
}
```

---

### KokoAutoCarsCategories
Йерархична категоризация на автомобили. Table: `kokoauto_cars_categories`.

`parent_id = 0` → корен (`level = 1`). `is_leaf = 1` → няма деца.

```ts
{
  id:           int,
  parent_id:    int,            // 0 = root
  level:        int,            // 1 = root, 2 = ...
  is_leaf:      int,            // 0 | 1
  type_id:      int,            // тип на автомобила, за който важи категорията
  title:        string,
  description:  string,
  date_created: datetime,
  date_updated: datetime
}
```

> При повикване с `?parent_id=...&car_id=...` обектите се конструират от външно API (виж по-долу) и съдържат само: `id`, `parent_id`, `level`, `is_leaf`, `title`. Останалите полета липсват.

---

### KokoAutoPartsApi (външно API)
Не е entity клас — клиент за външното API на `partsapi.autochina.mk`.
Извиква се само индиректно през `GET /kokoauto_cars_categories?parent_id=...&car_id=...`.

При успех: връща `KokoAutoCarsCategories[]` (виж бележката по-горе).
При грешка от външното API:
```json
{
  "status": 0,
  "http_code": 500,
  "error": "съобщение",
  "raw": "..."
}
```

---

## 4. Бележки за фронтенд имплементация

1. **Цикличност при `KokoAutoCar`:** eager nested обектите (`model`, `engineType`, ...) са вложени в JSON-а само при single fetch (`?id=N`). При list НЕ са. Ако трябва да покажеш списък с конфигурации с имена на двигател/гориво — направи отделни заявки до съответните lookup endpoint-и (те са малко записи) и кеширай локално.

2. **`title` и `description`** винаги са string — никога `null`. При липсваща стойност → `""`.

3. **`yearFrom` / `yearTo`** идват като `int` или `null`. Третирай `0` като „без ограничение".

4. **Lookup типове** (`engine_types`, `fuel_types`, ...) са малко записи и не сменят често — могат да се кешират на frontend startup.

5. **Категориите** имат два различни режима — локален (от нашата БД) и външен (от Parts API). Винаги подавай `type_id` в стандартния режим, и `car_id` при `parent_id`.
