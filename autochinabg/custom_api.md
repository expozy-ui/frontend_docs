# autochinabg — Custom API

Документация за къстъм endpoint-ите на проекта (`projects/autochinabg/api/` в devcore).
Ядрените endpoint-и са в `core-v4-front.md` и тук не се повтарят.

> Документът се попълва постепенно — засега съдържа **архива** на огледи, заявки и сделки.

---

## 1. Общи конвенции

**Base URL:** `https://devcore.myexpozy.com/api`

| Хедър | Задължителен | Описание |
|---|---|---|
| `Authentication: Basic <saas_key>` | да | Ключът на проекта (публичен, `const SAAS_KEY` в HTML-а). |
| `Authorization: Bearer <token>` | да (за всичко в този документ) | Токенът на логнатия потребител. |

POST тялото е `application/x-www-form-urlencoded` (или `multipart/form-data`).

**Формат на грешка:**

```json
{ "status": 0, "error": "Текст на грешката", "errors": ["Текст на грешката"] }
```

Липсващ задължителен параметър:

```json
{ "status": 0, "errors": { "[name=\"deal_id\"]": "Please enter" } }
```

Без токен:

```json
{ "status": 0, "errors": ["Login first v4!"] }
```

---

## 2. Архив (огледи, заявки, сделки)

### 2.1. Как работи

- Всеки оглед, заявка и сделка има поле **`archived`** (`bool`, по подразбиране `false`).
- **Архивира само продавачът** — по огледи/заявки за своите обяви и по своите сделки.
  Купувачът/клиентът не може да архивира.
- Архивът е **само за продавача**: архивираният обект изчезва от *неговия* списък.
  **Купувачът вижда обекта винаги**, независимо от `archived`.
- Админът не архивира; в админ панела се виждат всички, архивирани или не.

Списъчните endpoint-и (`my_cars_inspections`, `my_cars_requests`, `my_deals`) връщат
обединение от две части — „като купувач" и „като продавач":

| Заявка | Частта „като купувач" | Частта „като продавач" |
|---|---|---|
| без `archived` | всички | **само неархивираните** |
| `?archived=1` | *не се връща* | **само архивираните** |

> **Промяна в поведението:** досега продавачът получаваше всичките си обекти.
> Сега архивираните не излизат без `?archived=1`. Докато нищо не е архивирано,
> отговорът е същият като преди.

---

### 2.2. Огледи

#### `GET /my_cars_inspections`

Огледите на логнатия потребител — като купувач и като продавач. Масив от [Inspection](#31-inspection), подреден по `id` DESC.

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `status_id` | int | не | `0` Изчакващ, `1` Одобрен, … Без него — всички статуси. |
| `archived` | int | не | `1` → само архивираните (като продавач). |

```bash
curl -s "https://devcore.myexpozy.com/api/my_cars_inspections?archived=1" \
  -H "Authentication: Basic <saas_key>" -H "Authorization: Bearer <token>"
```

#### `POST /my_cars_inspections_archive`

Продавачът архивира / разархивира оглед по своя обява.

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `inspection_id` | int | да | id на огледа. |
| `archived` | int | не | `1` архивира, `0` разархивира. По подразбиране `1`. |

Отговор:

```json
{ "status": 1, "msg": "Success edited", "obj": { "id": 4, "archived": true, "...": "..." } }
```

`obj` е пълният [Inspection](#31-inspection).

Грешки (`status: 0`):

| Кога | `error` |
|---|---|
| огледът не съществува, или потребителят не е продавачът по него | `_BC_ERR_INSPECTION_NO_ACCESS` |
| липсва `inspection_id` | `errors["[name=\"inspection_id\"]"]` |
| без токен | `Login first v4!` |

---

### 2.3. Заявки (резервации, покупки, лизинг)

#### `GET /my_cars_requests`

Заявките на логнатия потребител — като клиент и като продавач. Масив от [Request](#32-request), подреден по `id` DESC. `admin_notes` винаги е `""`.

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `type` | string | не | `reservation` / `purchase` / `leasing` — филтрира **само** частта „като клиент". |
| `archived` | int | не | `1` → само архивираните (като продавач). `type` тогава се игнорира. |

> `GET /my_cars_reservations` (съвместимост) също вече не връща архивираните от продавача.

#### `POST /my_cars_requests_archive`

Продавачът архивира / разархивира заявка по своя обява.

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `request_id` | int | да | id на заявката. |
| `archived` | int | не | `1` архивира, `0` разархивира. По подразбиране `1`. |

Отговор:

```json
{ "status": 1, "msg": "Success edited", "obj": { "id": 15414, "archived": true, "...": "..." } }
```

`obj` е пълният [Request](#32-request) (с `admin_notes: ""`).

Грешки (`status: 0`):

| Кога | `error` |
|---|---|
| заявката не съществува, или потребителят не е продавачът по нея | `_BC_ERR_REQUEST_NO_ACCESS` |
| липсва `request_id` | `errors["[name=\"request_id\"]"]` |
| без токен | `Login first v4!` |

---

### 2.4. Сделки

#### `GET /my_deals`

Сделките на логнатия потребител — като купувач и като продавач. Масив от [Deal](#33-deal).

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `id` | int | не | Единична сделка (само за участник) — **не се влияе** от архива. |
| `archived` | int | не | `1` → само архивираните (като продавач). |

#### `POST /my_deals_archive`

Продавачът архивира / разархивира своя сделка.

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `deal_id` | int | да | id на сделката. |
| `archived` | int | не | `1` архивира, `0` разархивира. По подразбиране `1`. |

Отговор:

```json
{ "status": 1, "msg": "Success edited", "obj": { "id": 25, "archived": true, "...": "..." } }
```

`obj` е пълният [Deal](#33-deal).

Грешки (`status: 0`):

| Кога | `error` |
|---|---|
| сделката не съществува | `_BC_ERR_DEAL_NOT_FOUND` |
| потребителят не е продавачът по сделката | `_BC_ERR_DEAL_NO_ACCESS` |
| липсва `deal_id` | `errors["[name=\"deal_id\"]"]` |
| без токен | `Login first v4!` |

---

## 3. Схеми

### 3.1. Inspection

| Поле | Тип | Описание |
|---|---|---|
| `id` | int | |
| `auction_id` | int | обявата |
| `buyer_id` | int | купувачът (заявил огледа) |
| `seller_id` | int | продавачът по обявата |
| `names` | string | имена за контакт |
| `phone_code` | string | |
| `phone` | string | |
| `inspection_date` | string | `Y-m-d` |
| `inspection_time` | string | `H:i` или `""` |
| `status_id` | int | виж `GET /cars_inspections_statuses` |
| `archived` | bool | архивиран от продавача |
| `date_created` | string | `Y-m-d H:i:s` |
| `date_updated` | string | `Y-m-d H:i:s` |
| `status` | object | `CarsInspectionStatus` (`id`, `title`, …) |
| `auction` | object | обявата (`CarsAuction`) |

### 3.2. Request

| Поле | Тип | Описание |
|---|---|---|
| `id` | int | |
| `type` | string | `reservation` / `purchase` / `leasing` |
| `auction_id` | int | обявата |
| `user_id` | int | клиентът (`0` = гост при лизинг) |
| `seller_id` | int | продавачът |
| `names`, `email`, `phone`, `city`, `note` | string | контакти и бележка на клиента |
| `total_price` | float | цената на обявата към момента на заявката |
| `amount` | float | дължимо: такса 10% / цялата цена / `0` при лизинг |
| `payment_method` | string | `""` / `bank` / `stripe` |
| `fee_intent_id` | string | |
| `down_payment_percent`, `residual_percent` | float | лизинг |
| `is_new` | bool | лизинг |
| `months` | int | лизинг |
| `monthly_installment` | float | лизинг, без ДДС |
| `status_id` | int | статус на заявката |
| `admin_notes` | string | винаги `""` към фронта |
| `archived` | bool | архивирана от продавача |
| `date_created` | string | |
| `date_paid` | string\|null | |
| `status` | object | `CarsRequestStatus` |
| `auction` | object | обявата |
| `payments` | array | `CarsRequestPayment[]` |
| `paid_amount` | float | сбор на плащанията |
| `due_amount` | float | остатък за доплащане |

### 3.3. Deal

| Поле | Тип | Описание |
|---|---|---|
| `id` | int | |
| `auction_id` | int | |
| `buyer_id` | int | |
| `seller_id` | int | |
| `status_id` | int | статус на сделката |
| `final_amount` | float | крайна сума |
| `deposit_id` | int | |
| `buyer_confirmed`, `seller_confirmed` | bool | |
| `confirmation_deadline` | string | |
| `seller_handover`, `buyer_handover` | bool | |
| `seller_direct_payment`, `seller_received_payment` | bool | |
| `vin` | string | |
| `archived` | bool | архивирана от продавача |
| `date_created`, `date_updated` | string | |
| `phone_seller`, `phone_client` | string | |
| `seller_names` | string | |
| `auction` | object | обявата |
| `status` | object | `CarsAuctionDealStatus` |
| `commission` | object | `CarsAuctionCommission` |
| `counter_offers` | array | `CarsAuctionCounterOffer[]` |
| `client_info` | object | `UserInfoSimple` |
