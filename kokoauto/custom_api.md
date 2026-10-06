# KokoAuto — Public GET API

Документация за публичните endpoint-и от `api/front/Get.php` + схеми на върнатите обекти. Използва се от АИ за изграждане на фронтенд.

> Всички endpoint-и са **GET**. Не изискват вход на потребител (за разлика от admin endpoint-ите), освен където изрично е отбелязано (напр. `favourites`).
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
Дървото на категориите части — менюто („Спирачна система" → „Спирачни накладки", ...).

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Връща една категория (пълен обект, вкл. `tecdoc_nodes`). |
| `type_id` | int | Задължителен, ако няма `car_id`. **Внимание:** сървърът го презаписва на `0`, така че в момента за всеки `type_id` идва едно и също дърво. |
| `car_id` | int | Само категориите, вързани към колата (през `kokoauto_cars_categories_href`). |
| `parent_id` | int | **Специален режим** — извиква външното Parts API (`partsapi.autochina.mk` чрез [`KokoAutoPartsApi::getSearchTree`](#kokoautopartsapi-външно-api)). Изисква и `car_id`. Връща поддърво за категория + конкретен автомобил. |

**Връща (без `id` и `parent_id`):** **дърво** — масив от корените (`level = 1`, `parent_id = 0`), всеки с децата си в `subcats` (рекурсивно). Всеки възел е [`KokoAutoCarsCategories`](#kokoautocarscategories).

- В дървото `tecdoc_nodes` винаги е `[]` — мапингът към TecDoc не се зарежда, за да е бързо менюто.
- Категория без снимка има празен `image` (`id: 0`, `url: ""`) — обектът винаги го има.

> Бележка: при `parent_id` обектите идват от външен източник и `id`-тата им са `nodeId` от Parts API-то, а `title` идва от `nodeName`. Полета `date_created`/`date_updated` отсъстват.

---

### 2.13. `GET /brands` (не е част от KokoAuto)
Endpoint от `Brands` класа на core-а. Включен е в Get.php, но не е част от KokoAuto домейна. Не го документираме тук.

---

## 2A. Продукти и каталог

Продуктите са части от TecDoc каталога, които се продават през нашите доставчици (Inter Cars и др.). Всеки артикул се идентифицира с **`id_code`** = `"<brandNo>::<artNo>"` (напр. `"81::H97W08"`) — с него се отварят детайлът, кросовете и OE номерата. Връща се като `id_code` във всеки [`Product`](#product).

Цените са **с ДДС**, в €.

### 2.14. `GET /kokoauto_products` — листинг за кола + категория
Частите от дадена категория, които пасват на дадена кола.

| Param | Тип | Default | Описание |
|-------|-----|---------|----------|
| `car_id` | int | — | **Задължителен.** Колата (`KokoAutoCar.id`). |
| `category_id` | int | — | **Задължителен.** Листова категория (`is_leaf = 1`). |
| `page` | int | 1 | Страница. |
| `limit` / `per_page` | int | 20 | Брой на страница. |
| `brands` | CSV int | — | Само тези брандове (`brandNo`), напр. `30,81`. |
| `available` (или `in_stock`) | `1` | — | Само наличните. Без него идват всички — **наличните първи**, после тези без наличност. |
| `criteria[<id>]` | string \| string[] | — | Филтър по характеристика (стойност от `filters.criteria[].values[].value`). Няколко стойности на един критерий = ИЛИ; различни критерии = И. |
| `criteria_min[<id>]`, `criteria_max[<id>]` | number | — | Числов диапазон по характеристика (за тези с `num_min`/`num_max`). |
| `min_price`, `max_price` | number | — | Ценови диапазон. |

**Връща:**
```ts
{
  pagination:   Pagination,
  ancestorNode: object | null,     // служебно (от PartsAPI), фронтът не го ползва
  truncated:    bool,              // служебно
  brands:       BrandFacet[],      // брандовете в целия набор (преди филтъра по бранд)
  filters: {
    criteria:   CriteriaFilter[],  // характеристиките с наличните стойности
    price:      { min: number, max: number }   // граници за slider-а (от целия набор)
  },
  result:       Product[]
}
```

> Първото отваряне на нова комбинация кола + категория може да отнеме няколко секунди (данните се дърпат от PartsAPI и се кешират). Следващите са бързи (~1 s за 50 продукта).

---

### 2.15. `GET /kokoauto_products?id_code=…` — детайл на продукт

| Param | Тип | Описание |
|-------|-----|----------|
| `id_code` | string | **Задължителен.** `"<brandNo>::<artNo>"`. |
| `car_id` | int | По желание — колата, от която идва потребителят. |

**Връща:** един [`Product`](#product) (не е в масив). Спрямо листинга има и детайлни полета: `criteria`, `cars`, `car_external_ids` (виж схемата). Непознат артикул → празен продукт с `id: 0`.

---

### 2.16. `GET /kokoauto_aftermarket_products` — алтернативи (кросове)
Взаимнозаменяеми части от други брандове — за блока „Алтернативи" в детайла.

| Param | Тип | Описание |
|-------|-----|----------|
| `id_code` | string | **Задължителен.** Артикулът, за който търсим заместители. |

**Връща:** плосък масив `Product[]` — най-много по един артикул на бранд, само такива, които продаваме. За оригинални (OE) части винаги `[]`.

> Прави заявка към PartsAPI → ~1–2 s. Ако някой от кросовете още го няма при нас, първото отваряне е по-бавно.

---

### 2.17. `GET /kokoauto_oe_numbers` — оригинални номера
OE номерата на автопроизводителите (BMW, FORD, ...), на които отговаря артикулът.

| Param | Тип | Описание |
|-------|-----|----------|
| `id_code` | string | **Задължителен.** |

**Връща:** плосък масив [`OeNumber[]`](#oenumber). За самите OE части — `[]`.

---

### 2.18. `GET /kokoauto_brand_categories` — категориите на бранд
Нашите категории, в които даден бранд има части (за страницата на бранда).

| Param | Тип | Default | Описание |
|-------|-----|---------|----------|
| `brand_id` | int | — | **Задължителен.** `brandNo` (= `Product.brand_id`). |
| `type` | int | 1 | 1 = леки, 2 = товарни. |

**Връща:** дърво като [`/kokoauto_cars_categories`](#212-get-kokoauto_cars_categories) — корени със `subcats`, но в `subcats` са **само** листата, в които брандът има части.

---

### 2.19. `GET /kokoauto_brand_products` — продукти на бранд в категория (без кола)

| Param | Тип | Default | Описание |
|-------|-----|---------|----------|
| `brand_id` | int | — | **Задължителен.** |
| `category_id` | int | — | **Задължителен.** Листова категория (от `kokoauto_brand_categories`). |
| `type` | int | 1 | 1 = леки, 2 = товарни. |
| `page` | int | 1 | |
| `limit` / `per_page` | int | 50 | |
| `criteria[...]`, `criteria_min[...]`, `criteria_max[...]` | | | Като в `kokoauto_products`. |

**Връща:**
```ts
{
  pagination: Pagination,
  filters:    CriteriaFilter[],   // ВНИМАНИЕ: плосък масив (не {criteria, price}) и само за текущата страница
  result:     Product[]
}
```
Няма филтри по наличност и цена.

---

### 2.20. `GET /kokoauto_search` — глобалната търсачка

| Param | Тип | Default | Описание |
|-------|-----|---------|----------|
| `q` | string | — | Текстът. **Под 3 символа → всички групи идват празни** (без търсене). |
| `limit` | int | 10 | Брой на група. |
| `page` | int | 1 | Страница (еднаква за всички групи). |
| `category_id` | int | — | Само за `products` — стеснява до категория. |
| `min_price`, `max_price` | number | — | Само за `products`. |

**Връща** обект с група за всеки домейн:
```ts
{
  brands:     Brands[],                  // брандове части
  makers:     KokoAutoCarMaker[],        // марки коли (само фокус-марките)
  series:     KokoAutoCarModelSeries[],  // модели (в админа: „Модел")
  models:     KokoAutoCarModel[],        // купета (в админа: „Купе")
  cars:       KokoAutoCar[],             // модификации — по име И по код на двигателя
  categories: KokoAutoCarsCategories[],  // само листови категории
  products: {
    pagination: Pagination,
    brands:     BrandFacet[],
    categories: KokoAutoCarsCategories[], // категориите, в които има намерени продукти
    filters:    { criteria: CriteriaFilter[], price: { min, max } },
    result:     Product[]
  }
}
```
- Групите извън `products` са **плоски масиви** (без `pagination`), всяка с най-много `limit` елемента. `page` важи за всички групи освен `cars` (там винаги идват първите `limit`).
- Търсенето е по начало на дума: „bmw x" намира „BMW X3", но не „BMW iX".
- `products` идват от външното PartsAPI → търсенето отнема **от 3 до 30 s** при широки заявки (напр. „bosch"). Показвай отделно зареждане за продуктите.

---

### 2.21. `GET /products` — общ списък продукти (напр. „Най-продавани")

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Един продукт. |
| `sort` | string | `sales` (най-продавани), `newest`, `oldest`, `min_price`, `max_price`, `title`, `random`, `discount`, `boost`. |
| `page`, `limit` | int | Пагинация. |

Връща само наличните продукти.

**Връща:** `{ pagination: Pagination, result: Product[] }`.

> `sort=sales` връща **само продукти, които вече са поръчвани**, подредени по брой продажби. Ако такива са по-малко от `limit`, идват толкова, колкото са (няма допълване със случайни).

---

### 2.22. `GET /favourites` — любими 🔒
**Изисква вход** (Bearer токен на потребителя).

Любимите продукти на потребителя, включително без наличност. Параметри: `page`, `limit`.

**Връща:** `{ pagination: Pagination, result: Product[] }`.

---

### 2.23. `GET /promotions` — промоции

| Param | Тип | Описание |
|-------|-----|----------|
| `id` | int | Една промоция. |

Без `id` връща само активните и валидни към момента промоции (без вътрешните за регистрация).

**Връща:** плосък масив [`FrontPromo[]`](#frontpromo) (или един обект при `id`).

#### Отстъпка „Поръчки над" (`discount_subject = "orders_over"`)
Отстъпка за цялата поръчка, когато стоките в количката (с ДДС, след намаленията по продукти, **без доставката**) стигнат `min_amount`. Може да е автоматична (`type = 2`, без код) или промокод (`type = 1`).

- **Не се комбинират** с промокод — прилага се по-голямата отстъпка.
- Промокод „Поръчки над" под прага се отказва със съобщение „Кодът важи за поръчки над X €". Ако количката по-късно падне под прага, кодът се маха от нея.

В отговора на количката (`GET /cart`) има и:
```ts
{
  subtotal_goods:       number,   // стоките преди отстъпката за поръчка
  promocode_applied:    bool,     // false, ако има код, но автоматичната отстъпка е по-изгодна
  order_promotion:      { id, title, discount_type, discount, min_amount, amount } | null,  // приложената автоматична; amount = отстъпката в €
  next_order_promotion: { id, title, discount_type, discount, min_amount, remaining } | null // следващият праг; remaining = колко още до него
}
```
Пример за банер: `next_order_promotion` → „Добави още {remaining} € и вземи {discount}% отстъпка".

---

### 2.24. `GET /product_comments` — отзиви

| Param | Тип | Описание |
|-------|-----|----------|
| `product_id` | int | Отзивите за продукта. |
| `rating` | int | Само с тази оценка. |

Връща само одобрените (`status_id = 1`) коментари с `type = "global"`.

**Връща:** плосък масив [`Comment[]`](#comment).

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
  focus:        bool,
  title:        string,
  image: {                      // винаги го има; без снимка → id: 0, url: ""
    id:           int,
    object_id:    int,          // = id на категорията
    filename:     string,
    sort_order:   int,
    date_created: datetime,
    url:          string,       // пълен URL на снимката
    url_10x10:    string        // миниатюра
  },
  subcats:      KokoAutoCarsCategories[],  // децата — само в дървото
  tecdoc_nodes: object[],       // мапинг към TecDoc — вътрешно; в дървото винаги []
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

### Pagination
```ts
{
  total_results:    int,
  total_pages:      int,
  current_page:     int,
  results_per_page: int
}
```

---

### Product
Продукт (част). Едни и същи полета в листингите, детайла, кросовете и търсенето; някои се пълнят само в детайла (отбелязано).

```ts
{
  id:            int,
  type:          int,       // 8 = част от каталога (TecDoc)
  id_code:       string,    // "<brandNo>::<artNo>" — ключът за детайл / кросове / OE номера
  external_id:   string,    // същото като id_code
  ref_number:    string,    // артикулният номер (artNo)
  gen_art_no:    int,       // TecDoc тип част
  title:         string,
  slug:          string,
  url:           string,    // линк към продукта на сайта, вече с ?id_code=...
  description:   string,
  short_body:    string,
  seo_title, seo_description, seo_tags: string,

  brand_id:      int,       // = brandNo
  brand:         Brands,    // { id, title, images[], website, company, available_parts_count, cars_count, categories_count, ... }
  is_oe:         bool,      // оригинална част на автопроизводителя

  price_min:     number,    // цената за показване: най-ниската продажна, с промоцията
  delivery_days: int,       // срок на доставка в дни
  variations:    ProductVariation[],   // в kokoauto винаги 1 — цена и наличност
  images:        ProductImage[],

  categories_cache: KokoAutoCarsCategories[], // нашите категории, в които е продуктът (тук с пълния tecdoc_nodes)
  criteria_preview: CriteriaPreview[],        // 2–3 характеристики за картата — листинг / търсене / кросове
  criteria:         { id: int, name: string, value: string }[], // ВСИЧКИ характеристики — само в детайла
  cars:             CompactCar[],             // колите, на които пасва — само в детайла
  car_external_ids: int[],                    // само в детайла

  barcodes:      { id, product_id, barcode }[] | string[],  // в листинга обекти, в детайла само низове
  single_promo_active: { id, product_id, promoprice, promo_startdate, promo_enddate, active },

  rating:        number,    // средна оценка от одобрените отзиви
  isWishlisted:  bool,      // в любими ли е (за вписан потребител)
  isNew:         bool,      // създаден през последните 30 дни

  seller:        null,      // не се ползва в kokoauto
  supplier:      null,
  // винаги празни в kokoauto: categories, features, files, files_private, providers, quantity_discounts, subProducts
}
```

---

### ProductVariation
```ts
{
  id:            string,    // = id на продукта, като низ
  product_id:    int,
  sku:           string,    // = id_code
  external_id:   string,
  price:         number,    // редовна цена
  promoprice:    number,    // цена с промоция; 0 ако няма
  selling_price: number,    // цената, която се плаща
  discount:      number,    // отстъпка в %
  qty:           number,    // налично количество (0 = няма наличност)
  currency:      string,    // "€"
  unit_id, unit, attributes, name, file, rec_price, delivery_price  // не се ползват
}
```

---

### ProductImage
```ts
{
  id:           int,
  object_id:    int,        // id на продукта
  url:          string,     // пълен URL
  url_10x10:    string,
  external_url: string,
  alt:          string,
  sort_order:   int
}
```

---

### CriteriaPreview
```ts
{ criteria_id: int, name: string, value: string }   // напр. { 100, "страна на монтаж", "предна ос" }
```

### CriteriaFilter
Една характеристика във филтрите на листинга.
```ts
{
  criteria_id: int,
  name:        string,
  values:      { value: string, value_norm: string, count: int }[],  // за избор (enum)
  num_min:     number | null,   // за числов диапазон
  num_max:     number | null
}
```
Избраната стойност се подава като `criteria[<criteria_id>]=<value>`.

### BrandFacet
```ts
{ brandNo: int, brandName: string, articleCount: int }
```

### CompactCar
Кола в детайла на продукт. **Внимание:** числата идват като низове.
```ts
{
  id: string, external_id: string, title: string, maker: string,
  fuel: string, body_type: string | null,
  kw: string, kwTo: string, hp: string, hpTo: string,
  yearFrom: string, yearTo: string      // "ГГГГММ"; "0" = няма
}
```

### OeNumber
```ts
{
  oeNumber:    string,             // напр. "3 521 840"
  oeBrandName: string,             // напр. "FORD"
  oeBrandId:   int,
  maker:       KokoAutoCarMaker,   // марката с логото (logo.url)
  product:     Product | null      // нашият оригинален артикул, ако го продаваме; иначе null
}
```

### FrontPromo
```ts
{
  id:               int,
  type:             int,       // 1 = промокод, 2 = за всички
  discount_type:    string,    // "percent" | "fixed" | "free_shiping"
  discount:         number,    // % или сума
  discount_subject: string,    // "product" | "category" | "brand" | "all_orders" | "orders_over" | "collection"
  min_amount:       number,    // за "orders_over": от каква стойност на стоките важи (€, с ДДС, без доставката); иначе 0
  title:            string,
  subtitle:         string,
  description:      string
}
```

### Comment
```ts
{
  id:         int,
  type:       string,       // "global"
  subject_id: int,          // id на продукта
  comment:    string,
  rating:     int,
  user_id:    int,
  user_name:  string,       // имейлът на автора НЕ се връща
  parent_id:  int,          // > 0 = отговор на друг коментар
  status_id:  int,          // 1 = одобрен
  about:      [],           // за type "global" е празно — продуктът е в subject_id
  images:     object[],
  date, date_created, date_updated: datetime
}
```

---

## 4. Бележки за фронтенд имплементация

1. **Цикличност при `KokoAutoCar`:** eager nested обектите (`model`, `engineType`, ...) са вложени в JSON-а само при single fetch (`?id=N`). При list НЕ са. Ако трябва да покажеш списък с конфигурации с имена на двигател/гориво — направи отделни заявки до съответните lookup endpoint-и (те са малко записи) и кеширай локално.

2. **`title` и `description`** винаги са string — никога `null`. При липсваща стойност → `""`.

3. **`yearFrom` / `yearTo`** са във формат `ГГГГММ` (напр. `200307` = 07.2003). Според endpoint-а идват като число или като низ — приемай и двете. `0`, `"0"` или `null` = „без ограничение".

4. **Lookup типове** (`engine_types`, `fuel_types`, ...) са малко записи и не сменят често — могат да се кешират на frontend startup.

5. **Категориите** имат два различни режима — локален (от нашата БД) и външен (от Parts API). Винаги подавай `type_id` в стандартния режим, и `car_id` при `parent_id`.

6. **`id_code` в URL** — съдържа `::`, а артикулният номер може да има интервали (`"30::0 986 452 041"`). Винаги го кодирай с `encodeURIComponent`.

7. **Цена за показване** — `price_min`. Ако `variations[0].promoprice > 0`, има промоция: зачеркната е `variations[0].price`, а отстъпката в % е `variations[0].discount`. Наличност — `variations[0].qty > 0`; срок — `delivery_days`.

8. **Бавни заявки** — `kokoauto_search` (продуктите от PartsAPI, до 30 s), `kokoauto_aftermarket_products` (~1–2 s) и първото отваряне на нова кола + категория в `kokoauto_products`. Зареждай ги отделно от останалата страница и показвай индикатор.
