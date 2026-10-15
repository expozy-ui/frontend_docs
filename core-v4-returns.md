# Expozy Core v4 — Product Returns

> Everything a frontend needs to implement product returns (customer side) on any v4 shop.
> Self-contained: endpoints, rules, payloads, response shapes and errors. The general API conventions
> (base URL, `Authentication: Basic <saas_key>`, `Authorization: Bearer <token>`) are in `core-v4-front.md`.

---

## 1. Flow

1. Logged-in customer opens "Return a product" → frontend loads the returnable orders (`GET returnable_orders`)
   and the nomenclatures (options, conditions, reasons, return methods, refund methods).
2. Customer picks an order, ticks one or more items and a quantity for each (up to `returnable_qty`),
   fills the form and submits → `POST returns`. One request = one return request (header) with one or more items.
3. The response contains the request number (`obj.id`) and the submission date and time (`obj.date_created`) — show both.
4. The shop processes the request in the admin (status changes). The customer follows it in the profile → `GET returns`.

---

## 2. Rules (enforced by the server)

| Rule | Value |
|---|---|
| Who can return | Logged-in user, only own orders |
| Order statuses that allow a return | `5` Sent, `7` Completed |
| Return period | 14 days from `order.date_delivered` (set automatically when the order becomes status 7) |
| Order without `date_delivered` (still "Sent") | Customer enters `date_received`; it must be between the order date and today. The 14 days run from it, but never later than 30 days from the order date |
| Partial returns | Yes — any quantity from 1 to `returnable_qty` per item |
| Returnable quantity | `quantity` minus all non-rejected requests for that item. A rejected request (status 4) frees its quantity, so the item can be requested again |
| Several requests for one order | Yes, while there is returnable quantity left |
| Cancel a request | Not possible for the customer |
| Refund data | Only when `option_id = 1` (refund). For other options (e.g. replacement) no refund method / IBAN is needed |
| Refund method vs return method | `cod` (2) only with a return method of type `courier`; `cash` (3) only with type `store`; `bank` (1) with any |
| Bank refund | `iban`, `iban_name`, `bank_name` are required |

---

## 3. Endpoints (frontend)

All need `Authentication: Basic <saas_key>`. Endpoints marked **Bearer** also need `Authorization: Bearer <token>`
and return HTTP 403 `{ status: 0, errors: ["Login first v4!"] }` without it.

### GET /returnable_orders — **Bearer**
Alias: `GET /orders?returnable=1` (same response).

Orders of the current user that still have at least one returnable item. Newest first.
Only orders with status 5 or 7 that are inside the return period.

Each order is a normal `Order` object plus:
- `date_delivered` — `"Y-m-d H:i:s"` or `null`. When set, show "Date of receipt" pre-filled and locked; when `null`, the customer must enter `date_received`.
- `returnable_until` — `"Y-m-d"` (`date_delivered + 14 days`) or `""` when `date_delivered` is `null`. Show as "Can be returned until …".
- `products[].order_item_id` — the id to send in `items[]` / `qty`.
- `products[].returnable_qty` — max quantity that can still be returned; `0` = show the item disabled.
- `products[].ref_number`, `products[].id_code`, `products[].title`, `products[].quantity`, `products[].total` — for display.

Rows with a negative `total` are returns already deducted from the order — never returnable (`returnable_qty: 0`); hide them or show them as "Returned".

```json
[
  {
    "id": 3,
    "number": "162916",
    "status_id": 7,
    "date_created": "2026-10-14 07:18:26",
    "date_delivered": "2026-10-14 07:20:56",
    "returnable_until": "2026-10-28",
    "price": 220,
    "products": [
      { "id": 13, "order_item_id": 6, "title": "Product 6", "ref_number": "1", "quantity": 1, "total": 133, "returnable_qty": 0 },
      { "id": 14, "order_item_id": 5, "title": "Product 7", "ref_number": "1", "quantity": 3, "total": 330, "returnable_qty": 2 }
    ]
  }
]
```

### GET /returns_options
What the customer wants. Fields: `id`, `title`, `date_created`.
`id = 1` is **refund** ("The value of the product") — only then the refund block is shown. Other ids (e.g. `2` replacement) — hide the refund block.

### GET /returns_conditions
Product condition (packaged / unpacked). Fields: `id`, `title`, `date_created`. Required in the form.

### GET /returns_reasons
Reason for the return (managed by the shop). Fields: `id`, `title`, `date_created`. Optional in the form; free text goes to `note`.

### GET /returns_methods
How the goods are returned (managed by the shop). Fields: `id`, `title`, `type` (`courier` | `store`), `date_created`.

### GET /returns_refund_methods
Fixed list: `1` bank transfer, `2` cash on delivery, `3` cash in store. Fields: `id`, `title`, `date_created`.
Filter the list by the selected return method: `type = courier` → `1, 2`; `type = store` → `1, 3`.

All five accept an optional id (`/returns_reasons/4`) and return a single object.

### POST /returns — **Bearer**
Creates a return request. JSON (`Content-Type: application/json`) or form-encoded.

| Field | Type | Required | Notes |
|---|---|---|---|
| `order_id` | int | yes | from `returnable_orders` |
| `items` | int[] | yes | `order_item_id` values |
| `qty` | object | yes | `{ "<order_item_id>": count }`, `1 … returnable_qty` |
| `option_id` | int | yes | `returns_options` |
| `condition_id` | int | yes | `returns_conditions` |
| `return_method_id` | int | yes | `returns_methods` |
| `refund_method_id` | int | when `option_id = 1` | must fit the return method type |
| `iban`, `iban_name`, `bank_name` | string | when `refund_method_id = 1` | |
| `date_received` | `Y-m-d` | when the order has no `date_delivered` | between the order date and today |
| `reason_id` | int | no | `returns_reasons` |
| `note` | string | no | free text (reason details) |
| `names`, `phone`, `email`, `city`, `address` | string | yes | contact data, stored on the request (pre-fill from the profile/order) |

```json
{
  "order_id": 3,
  "items": [5, 6],
  "qty": { "5": 1, "6": 1 },
  "option_id": 1,
  "condition_id": 1,
  "reason_id": 4,
  "note": "Wrong model",
  "return_method_id": 2,
  "refund_method_id": 1,
  "iban": "BG80BNBG96611020345678",
  "iban_name": "Ivan Petrov",
  "bank_name": "DSK",
  "date_received": "2026-10-14",
  "names": "Ivan Petrov",
  "phone": "0888123456",
  "email": "test@example.com",
  "city": "Sofia",
  "address": "Test str. 5"
}
```

Success:
```json
{ "status": 1, "obj": { "id": 3, "status_id": 1, "date_created": "2026-10-14 07:20:06", "...": "full Returns object (§5)" }, "msg": "Success added" }
```
Show "Request number #`obj.id`" and the submission date/time `obj.date_created` — they go into the legal confirmation.

Errors (`status: 0`):

| Case | Response |
|---|---|
| Missing required fields | `{ "errors": { "[name=\"order_id\"]": "Please enter", ... } }` |
| Order not found / not the user's / not in status 5 or 7, or an item not from this order | `{ "error": "Access Denied" }` |
| Period expired | `{ "error": "The return period has expired" }` |
| `date_received` missing / in the future / before the order | `{ "errors": { "[name=\"date_received\"]": "..." } }` |
| Quantity 0 or above `returnable_qty` | `{ "errors": { "[name=\"qty[<order_item_id>]\"]": "Invalid quantity" } }` |
| Refund method does not fit the return method | `{ "errors": { "[name=\"refund_method_id\"]": "..." } }` |
| Bank refund without bank data | `{ "errors": { "[name=\"iban\"]": "Please enter", "[name=\"iban_name\"]": "...", "[name=\"bank_name\"]": "..." } }` |
| Invalid email | `{ "errors": { "[name=\"email\"]": "Please enter a valid email address." } }` |

Field errors are keyed by the input selector — show each next to its field. Texts come in the site language.

### GET /returns — **Bearer**
The current user's return requests (array of `Returns`, §5). `GET /returns/<id>` — one request; someone else's → HTTP 403 `{ "error": "Access Denied" }`.

---

## 4. Statuses

| id | Status | Meaning for the customer |
|---|---|---|
| 1 | New request | Submitted, waiting for the shop |
| 2 | Awaiting payment | Approved, refund pending |
| 3 | Paid | Refund paid |
| 4 | Disapproved | Rejected (see `note`); the quantity can be requested again |
| 5 | Waiting | On hold |
| 6 | Done | Closed |
| 7 | Received | The shop has received the goods |

Titles come translated in `status.title` — show them as they are. The shop can rename statuses, not add or delete them.

---

## 5. Model `Returns`

| Field | Type | Notes |
|---|---|---|
| `id` | int | request number |
| `order_id`, `user_id` | int | |
| `status_id`, `option_id`, `condition_id`, `reason_id`, `return_method_id`, `refund_method_id` | int | `0` = not set |
| `date_created` | string | submission date and time |
| `date_received` | string \| null | `Y-m-d`, only when entered by the customer |
| `date_paid` | string \| null | set by the shop |
| `names`, `phone`, `email`, `city`, `address` | string | |
| `iban`, `iban_name`, `bank_name` | string | empty unless bank refund |
| `note` | string | customer's text (as submitted) |
| `admin_note` | string | shop's note to the customer — e.g. the reason for a rejection; also sent in the approved / rejected email. Show it in "My returns" when not empty |
| `items` | array | `[{ id, return_id, order_item_id, product_id, qty, date_created, product: { id, id_code, title } }]` |
| `status`, `option`, `condition`, `reason`, `return_method`, `refund_method` | object | `{ id, title, date_created }` (`return_method` also has `type`); `id: 0` when not set |
| `product_id`, `shop_orders_items_id` | int | legacy (first item) — use `items` |
| `stock_returned`, `order_reduced`, `date_updated` | | internal |

---

## 6. Form checklist (frontend)

- Load `returnable_orders` + the five nomenclatures once when the page opens.
- Order selector → item list with checkbox + quantity input (`max = returnable_qty`, disabled at `0`).
- `date_received`: hidden/locked when `date_delivered` is set; required date input (max = today, min = order date) otherwise.
- Option radio → refund block only for `option_id = 1`.
- Return method → filter refund methods by `type`; bank fields only for `refund_method_id = 1`.
- Pre-fill `names`, `phone`, `email`, `city`, `address` from the profile/order; all editable.
- On success: show `#obj.id` and `obj.date_created`, refresh `returnable_orders`.
- Profile → "My returns" from `GET returns` with `status.title`, `items`, `date_created`.
