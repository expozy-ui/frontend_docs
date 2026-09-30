# Expozy Core v4 — Frontend API Reference

> Self-contained catalog of the v4 frontend API (`/core/v4/api/front/`) — endpoints, auth, params, return shapes, and model fields.
> Drop this file into the frontend project; Claude (or any LLM) can load it to implement features without re-reading the PHP backend.

---

## 1. Base

- **Base URL:** `https://<host>/core/v4/api/front/?route=<endpoint>`
- **HTTP methods:** GET, POST, PUT, DELETE
- **Content-Type:** `application/json` (responses are JSON; requests may be JSON or form-encoded; file uploads use multipart)
- **Charset:** UTF-8, JSON encoded with `JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE`
- **CORS:** `Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: GET, POST, PUT, DELETE`. Allowed headers: `Authentication, Authorization, Origin, X-Requested-With, Content-Type, Accept, Cache-Control, X-CSRF-TOKEN, X-XSRF-TOKEN, Session`.

Routing dispatch: `index.php` → `Api\Factory::create()` instantiates `Get`/`Post`/`Put`/`Delete` by HTTP method → `Base::execute_method()` invokes the public method whose name equals `?route=`. Every public method in those four classes is exactly one endpoint.

---

## 2. Authentication

### Bearer token (logged user)
- Header: `Authorization: Bearer <token>` (case-insensitive; `bearer` also accepted).
- Token is resolved with `Users::get_byToken()`. Invalid → HTTP 403 with `{ errors: "Failed authorization 1", redirect: "/logout" }`.
- `Required::logged()` inside an endpoint returns HTTP 403 `{ status: 0, errors: "Login first v4!" }` if no valid token.
- `Required::admin()` / `Required::superAdmin()` → HTTP 403 `{ errors: "Failed admin authorization" }` for non-admins.
- `Required::userLevel(n)` / `Required::userLevels(...)` → HTTP 400 `{ errors: "Failed level authorization ..." }` on mismatch.

### Session header (guest)
- Header: `Authorization: Session <sesid>` (case-insensitive).
- Used for guest carts/checkout. The `sesid` is any string the client keeps across requests.
- `Required::header('authorization', 'session')` enforces the header's presence.

### Ban check
- `Api\Request::check_fro_ban()` runs on every request: more than 300 requests per hour from one IP returns `{ status: 0, error: "You are banned 1!" }`.

---

## 3. Request parameters

- **GET:** query string. `?query_id=N` (or `?id=N`) sets the internal `Request::$id` used by endpoints that act on a single record.
- **POST / PUT / DELETE:** JSON body **or** `application/x-www-form-urlencoded`. If empty, `php://input` is re-parsed. DELETE also merges `$_GET` into the payload. A body field `id` also sets `Request::$id`.
- **File uploads:** multipart/form-data. Field name must match the one in the endpoint's `Required::file('<field>')` (e.g. `files`, `image`, `identity`).
- **Validation errors:** HTTP 400 with `{ status: 0, errors: { '[name="<field>"]': "<message>", ... } }`.

Typed parameter hints (from `Required::check`): `string`, `int`, `float`/`double`, `bool`, `array`, `object`, `email`, `date`. Coercion is applied before the handler runs.

---

### Generic list parameters (inherited by every model-backed list endpoint)

Almost every GET list endpoint ends in `Model::get_all()` / `get_all_pagination()`, which run through `SimpleObject::_get_all()` (`lib/classes/baseClasses/class.simpleobject.php`). That means these parameters work on **any** such endpoint without being repeated per route:

| Param | Type | Effect |
|---|---|---|
| `<any model field>` | scalar | Filter. String fields → `LIKE '%value%'`; int/float/bool fields → exact `=`. Array/NULL-typed properties are ignored. So **the model field list in §10 doubles as the list of accepted filters**. |
| `title` | string | Always `LIKE '%...%'` against the translated content value, not the raw column. |
| `date_*` (any field starting with `date_`) | date | Matched by day: `DATE(col) = DATE(value)`. |
| `date_created_from` / `date_created_to` | `YYYY-MM-DD` | Range on `date_created` (strict `>` / `<`). |
| `only_ids` | int or int[] | Restrict to `id IN (...)`. Empty value → empty result set. |
| `user` | string | Joins `users` and matches `first_name`, `last_name` or `email` with `LIKE`. |
| `admin_search` | string | Matches `id` exactly OR `title` with `LIKE` (only on models that have `title`). |
| `order_by` | string | Must be a real model field/DB column, otherwise ignored. |
| `order_direction` / `sortDirection` | `ASC`\|`DESC` | Sort direction; defaults to `ASC`. |
| `page`, `limit` | int | Pagination (only on `get_all_pagination` endpoints). |
| `no_pagination` | 0\|1 | `1` flattens a paginated response to a plain array. |

Models that override `_get_all()` (e.g. `Product`, `Blog`, `Rental`, `Order`) add their own filters on top; those are listed on the individual endpoint.

---

## 4. Response shapes

All endpoints terminate with `Api\Response::output(...)` or `Response::output_single(...)`. Four recurring shapes:

**A. Single object** — returned by `output_single`. HTTP 404 and empty `{}` when the object has `id == 0`.
```json
{ "id": 1, "title": "...", "...": "..." }
```

**B. Flat list** — model class `get_all(data)`; typically used for lightweight or unbounded collections.
```json
[ { "id": 1 }, { "id": 2 } ]
```

**C. Paginated list** — model class `get_all_pagination(data)`.
```json
{
  "pagination": { "page": 1, "limit": 20, "total_rows": 150, "total_pages": 8 },
  "result": [ { "id": 1, "...": "..." } ]
}
```
Disable with `?no_pagination=1` → becomes shape **B**.

**D. CRUD / process result** — returned by `Model::process(data)` and similar factory methods.
```json
{ "status": 1, "obj": { "id": 7, "...": "..." } }
```
or on failure:
```json
{ "status": 0, "errors": { "[name=\"field\"]": "message" } }
```
When `status == 0` + `errors|error|msg`, HTTP 400 is set automatically.

Generic error envelope (any 4xx/5xx):
```json
{ "errors": ["...", "..."] }
```

---

## 5. Conventions used below

Each endpoint block uses this compact template:

```
### <METHOD> /?route=<name>
- Auth: <none | Session | Bearer | Bearer+admin | ...>
- Params: <required ..., optional ...>
- Returns: <shape> of <Model>
- Notes: <any gotchas>
```

When a route accepts `query_id` for single-record reads, it is written as `id=<int>` in params. `Required::id()` resolves to either `?query_id=` or body `id`.

---

## 6. GET endpoints

### GET /?route=accounts
- Auth: Bearer
- Params: —
- Returns: single [Users](#users) (the logged-in user; full profile)
- Notes: HTTP 401 `{error:"Log in first"}` if not logged.

### GET /?route=admin_public_url
- Auth: none
- Params: `url` (string, required)
- Returns: decoded admin public URL payload (from `AdminMenuPublicUrl::decode`)

### GET /?route=admin_url
- Auth: none
- Returns: string — `https://admin.expozy.com/login/<project>`

### GET /?route=ads_categories
- Auth: none
- Params: optional `id=<int>` for single, otherwise filters
- Returns: single [AdsCategory](#adscategory) or flat list

### GET /?route=ads_fields
- Auth: none
- Params: optional `id=<int>`
- Returns: single [AdsField](#adsfield) or flat list

### GET /?route=ads_plans
- Auth: none
- Params: optional `id=<int>`
- Returns: single [AdsPlan](#adsplan) or flat list

### GET /?route=ads_requests
- Auth: none
- Params: optional `id=<int>`
- Returns: single [AdsRequest](#adsrequest) or flat list

### GET /?route=ads_types
- Auth: none
- Params: optional `id=<int>`
- Returns: single [AdsType](#adstype) or flat list

### GET /?route=ads_wishlist
- Auth: Bearer
- Returns: flat list of favorite object ids (see [Favorites](#favorites))

### GET /?route=advertisements
- Auth: none
- Params: optional `id=<int>` (also increments view counter)
- Returns: single [Ads](#ads) or flat list

### GET /?route=audit_files
- Auth: none (shared secret)
- Params: `eik`, `month`, `year`, `secret_key` (all required; `secret_key` must match `AuditFileNAP::SECRET_KEY`)
- Returns: audit XML/JSON payload generated by `AuditFileNAP::generate`.

### GET /?route=auctions
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Auction](#auction) or flat list

### GET /?route=banners
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Banners](#banners) or flat list

### GET /?route=blog_comments
- Auth: none
- Params: `subject_id` (int, required) — blog post id
- Returns: flat list of [Comments](#comments) (type=post)

### GET /?route=blogCategories
- Auth: none
- Params: optional `id=<int>`
- Returns: single [BlogCategory](#blogcategory) or flat list

### GET /?route=blogPosts
- Auth: none
- Params: optional `id=<int>`, pagination
- Returns: single [Blog](#blog) or paginated [Blog](#blog) list

### GET /?route=blogPosts_filters
- Auth: none
- Returns: filter metadata for blog list (categories, tags) — vendor-specific shape.

### GET /?route=brands
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Brands](#brands) or flat list

### GET /?route=cart
- Auth: Session or Bearer (401 if neither)
- Returns: [Cart](#cart) object
- Notes: Alias `carts` delegates here.

### GET /?route=carts
- Auth: Session or Bearer
- Returns: [Cart](#cart)
- Notes: delegates to `cart`.

### GET /?route=categories
- Auth: none
- Params: optional `id=<int>`, optional `droplist=1` (returns nested category tree), filters
- Returns: single [ProductCategory](#productcategory) (with `parents` tree) or flat list / tree

### GET /?route=category
- Auth: none
- Params: `id` (int, required)
- Returns: single [ProductCategory](#productcategory) with `parents` array

### GET /?route=cities
- Auth: none
- Params: optional `id=<int>`, filters
- Returns: single [Cities](#cities) or flat list

### GET /?route=combinations
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Combination](#combination) or flat list

### GET /?route=company
- Auth: none
- Returns: single [ProjectCompany](#projectcompany) (always `id=1`)

### GET /?route=contacts
- Auth: none
- Returns: settings map of contact info (phones, emails, socials) — from `AdminFunctions::get_contacts`

### GET /?route=cookie
- Auth: none
- Returns: cookie consent config from `Cookies::get_cookies`

### GET /?route=countries
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Country](#country) or flat list

### GET /?route=currencies
- Auth: none
- Params: `default=1` | `code=<string>` | `id=<int>` | none
- Returns: default currency | single [Currency](#currency) by code or id | flat list

### GET /?route=deposit_plans
- Auth: none
- Returns: flat list of active [DepositPlans](#depositplans) (status_id=1)

### GET /?route=deposits
- Auth: Bearer
- Params: optional `id=<int>`
- Returns: single [Deposit](#deposit) (403 `Access error!` if not owner) or flat list for current user

### GET /?route=econt
- Auth: none
- Params: optional `offices=1`
- Returns: Econt offices or cities — vendor-specific arrays (see `lib/classes/econt/`)

### GET /?route=epay
- Auth: none
- Params: `link` (string, required)
- Returns: ePay form markup/metadata (see `lib/classes/epay/`)

### GET /?route=experience_cancellationPolicies
- Auth: none
- Params: optional `id=<int>`
- Returns: single policy or flat list — `ExperienceCancellationPolicies`

### GET /?route=faq
- Auth: none
- Params: optional `id=<int>`
- Returns: single [FAQ](#faq) or flat list

### GET /?route=faqCategories
- Auth: none
- Params: optional `id=<int>`
- Returns: single FAQ category or flat list

### GET /?route=favourites
- Auth: none (but uses logged user if present)
- Returns: paginated [Product](#product) list with `only_favorites=1` applied
- Notes: alias `wishlist` delegates here.

### GET /?route=gallery
- Auth: none
- Params: optional `id=<int>`, pagination
- Returns: single [Gallery](#gallery) or paginated list

### GET /?route=gallery_categories
- Auth: none
- Params: optional `id=<int>`, pagination
- Returns: single [GalleryCategory](#gallerycategory) or paginated list

### GET /?route=git
- Auth: none
- Params: `github_token`, `github_route` (verify | repos | repo). For `repo`: `repo_name`, `repo_owner`.
- Returns: GitHub API wrapper response.

### GET /?route=hotel_reservations
- Auth: none
- Params: `last_name`, `date` (both required)
- Returns: matching [HotelReservationsOnline](#hotelreservationsonline)

### GET /?route=hotel_room_types
- Auth: none
- Params: optional `id=<int>`, `daterange=YYYY-MM-DD,YYYY-MM-DD`
- Returns: single [HotelRoomType](#hotelroomtype), or flat list. With `daterange` each item includes computed `price` for the stay.

### GET /?route=hybridAuth
- Auth: none
- Returns: HybridAuth config (social login providers)

### GET /?route=invoices
- Auth: Bearer (unless `hash` is supplied)
- Params: `hash=<string>` | `id=<int>` | `order_id=<int>`
- Returns: single [Invoice](#invoice). 401 if the invoice/order belongs to a different user.

### GET /?route=keys
- Auth: Bearer + admin
- Returns: flat list of all integration keys

### GET /?route=languages
- Auth: none
- Returns: flat list of [Language](#language) entries

### GET /?route=loyalty_card
- Auth: none
- Params: `id` (int, required)
- Returns: single [LoyaltyCard](#loyaltycard)

### GET /?route=marketplace_orders
- Auth: Bearer
- Params: optional `id=<int>`
- Returns: single [MarketplaceOrder](#marketplaceorder) or paginated list for current user

### GET /?route=marketplace_sellers
- Auth: none
- Params: optional `id=<int>`
- Returns: single [MarketplaceSeller](#marketplaceseller) or flat list

### GET /?route=meeting_calendars
- Auth: none
- Params: optional `id=<int>`
- Returns: single [MeetingCalendar](#meetingcalendar) or frontend list (public calendars)

### GET /?route=membership_check
- Auth: Bearer
- Returns: `{ status: 0|1 }` — whether the user has any active membership

### GET /?route=memberships
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Membership](#membership) or flat list

### GET /?route=menu
- Auth: none
- Returns: nested menu tree (see [Menu](#menu))

### GET /?route=messages_rooms
- Auth: Bearer
- Params: optional `id=<int>`
- Returns: single [MessagesRooms](#messagesrooms) (403 if no access) or frontend list for current user

### GET /?route=my_addresses
- Auth: Bearer
- Returns: flat list of [UserAddress](#useraddress) for current user

### GET /?route=my_bank_info
- Auth: Bearer
- Returns: single [UsersBankInfo](#usersbankinfo) for current user

### GET /?route=my_company
- Auth: Bearer
- Returns: single [UserCompany](#usercompany) for current user

### GET /?route=my_deposits
- Auth: Bearer
- Returns: flat list of [Deposit](#deposit) for current user

### GET /?route=my_dividents
- Auth: Bearer
- Returns: flat list of [DepositDivident](#depositdivident) for current user

### GET /?route=my_loyalty_card
- Auth: Bearer
- Returns: single [LoyaltyCard](#loyaltycard) for current user

### GET /?route=my_marketplace_sellers
- Auth: Bearer
- Returns: single [MarketplaceSeller](#marketplaceseller) owned by current user

### GET /?route=my_rentals
- Auth: Bearer
- Params: optional `id=<int>`
- Returns: single [Rental](#rental) (403 if not owner) or flat list for current user

### GET /?route=my_rentals_reservations
- Auth: Bearer
- Params: optional `id=<int>`, `host=1` (returns reservations for rentals owned by current user)
- Returns: reservation detail (assoc array with `rental` sub-object) or flat/paginated list.

### GET /?route=my_referrals
- Auth: Bearer
- Returns: flat list `[{ email }]` of referred users

### GET /?route=my_saas_template
- Auth: none
- Params: `saas_key` (required)
- Returns: [SaasFrontendTemplates](#saasfrontendtemplates) record matching the key

### GET /?route=my_transactions
- Auth: Bearer
- Returns: flat list of [UsersWalletsTransactionsAmount](#userswallets) entries for current user

### GET /?route=my_users_spoken_languages
- Auth: Bearer
- Returns: list of spoken-language hrefs for current user (see [UsersSpokenLanguages](#usersspokenlanguages))

### GET /?route=my_wallets
- Auth: Bearer
- Returns: flat list of [UsersWallets](#userswallets) for current user

### GET /?route=npos_tables
- Auth: none
- Params: `hash` (required)
- Returns: single [NPosTable](#npostable) by hash

### GET /?route=order
- Auth: Bearer OR guest session (ownership required)
- Params: `id` (int, required)
- Returns: single [Order](#order). 401 if not owner.

### GET /?route=order_payments
- Auth: none
- Returns: frontend [Payment](#payment) gateway list (`Payment::frontGateways`)

### GET /?route=order_statuses
- Auth: none
- Returns: flat list of [OrderStatus](#orderstatus)

### GET /?route=orders
- Auth: Bearer
- Params: optional `id=<int>` (delegates to `order`)
- Returns: single [Order](#order) or paginated [Order](#order) list for current user (default limit 1000)

### GET /?route=pages
- Auth: none
- Params: `id=<int>` | `slug=<string>` | none
- Returns: single [Pages](#pages) (increments visitor log) or flat list

### GET /?route=partners
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Partner](#partner) or flat list

### GET /?route=payment_methods
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Payment](#payment) or flat list (frontend-visible only)

### GET /?route=private_file
- Auth: Bearer
- Params: `id` (int), `class` (one of: `MessagesFile`, `DoctorRecordFiles`, `ProductFilesPrivate`)
- Returns: binary file stream (`Content-Disposition` exposed). No JSON body.

### GET /?route=product
- Auth: none
- Params: `id` (int, required), optional `slug`
- Returns: single [Product](#product). Empty `{}` if slug mismatch. Increments view counter.

### GET /?route=product_attributes
- Auth: none
- Params: `id=<int>` (single attribute) OR (`product_id`, `attr_value_ids=comma-list`)
- Returns: single [ProductAttribute](#productattribute), or map of related attribute values grouped by id.

### GET /?route=product_comments
- Auth: none
- Params: `product_id` (int, required unless `only_featured=1`)
- Returns: flat list of [Comments](#comments) for that product

### GET /?route=product_filters
- Auth: none
- Returns: attribute-based filter tree for product listings (`Product::gen_filters`)

### GET /?route=products
- Auth: none (a logged user additionally gets `isWishlisted` per product)
- Returns: paginated [Product](#product) list — `{ pagination, result[], filters? }`. With `id` it delegates to `product` and returns a single object. Private files are attached as `files_private`; prices are VAT-converted when `core.price_with_vat` is false.
- Notes: `Product::get_all()` is a hand-written query, **not** the generic `_get_all()` — only the params below apply. Products with `type = 5` are never returned.

Params:

| Param | Type | Meaning |
|---|---|---|
| `id` / `query_id` | int | Single product (delegates to `product`) |
| `category_id` | int \| int[] | Filter by category |
| `subcategories` | 0\|1 | With `category_id`, also include child categories |
| `brands` | int[] | Filter by brand ids |
| `seller_id` | int | Marketplace seller |
| `user_id` | int | Owner of the product |
| `search` | string | Full-text search over title/description/id_code; logged via `Search::logSearch` |
| `tag` | string | Single tag (FIND_IN_SET over the `tags` content) |
| `tags` | string[] | Multiple tags |
| `price_from`, `price_to` | float | Price range (defaults `price_from=0.01` on this route) |
| `product_price_from`, `product_price_to` | float | Range against warehouse price table |
| `warehouse_id` | int | Which warehouse's price/qty to use (default `1`) |
| `attr_val` | array | Filter by attribute value ids |
| `features_values` | array | Filter by feature values (OR) |
| `features_values_and` | array | Filter by feature values (AND) |
| `promotions` | int \| array | Filter by promotion |
| `related_id` | int | Products related to this product |
| `only_active` | 0\|1 | Only active |
| `only_instock` | 0\|1 | Only in stock |
| `only_promo` | 0\|1 | Only with promo price |
| `only_featured` | 0\|1 | Only featured |
| `only_image` | 0\|1 | Only products with an image |
| `only_price` | 0\|1 | Only products with a price |
| `only_favorites` | 0\|1 | Only the logged user's favourites |
| `only_product_ids` | int[] | Restrict to these ids |
| `deleted` | 0\|1 | Include deleted (this route forces `0`) |
| `reverse_auction` | 0\|1 | Reverse-auction products |
| `filter` | 0\|1 | Enables the filter-mode query paths |
| `show_filters` | 0\|1 | Adds `filters` to the response, computed for the current result set |
| `show_parent_filters` | 0\|1 | Adds `filters` computed without the current filters applied |
| `sort` | enum | `newest`, `oldest`, `min_price`, `max_price`, `random`, `discount`, `title`, `order`, `boost`, `sales`, `relevant` (only meaningful with `search`) |
| `order_by` + `sortDirection` | string | Alternative sort by a raw model field |
| `page`, `limit` | int | Pagination |
| `no_pagination` | 0\|1 | Return a flat array instead (this route forces `0`) |

### GET /?route=products_features
- Auth: none
- Params: optional `id=<int>`
- Returns: single [ProductFeatures](#productfeatures) or flat list

### GET /?route=providers
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Provider](#provider) or flat list

### GET /?route=qa
- Auth: none
- Params: `product_id` (int, required), optional `id=<int>`
- Returns: single [QaQuestion](#qaquestion) or flat list (visible=1)

### GET /?route=QRtoken
- Auth: Bearer + admin
- Returns: `{ status: 1, QR: "<base64 png>" }` for mobile admin onboarding

### GET /?route=regions
- Auth: none
- Params: `id` | `selected=1` | filters
- Returns: single [Region](#region) or the currently selected one, or flat list

### GET /?route=releva
- Auth: none
- Params: `page` (int, required), `warehouse_token` (string, required)
- Returns: Releva-format product payload (vendor-specific)

### GET /?route=rental_res_statuses
- Auth: none
- Params: optional `id=<int>`
- Returns: single [RentalReservationsStatus](#rentalreservationsstatus) or flat list

### GET /?route=rentals
- Auth: none (except `only_comment=1`)
- Params: `date_start`, `date_end`, `guests` (when searching availability); `only_comment=1` for comment-enabled rentals; `id` for single
- Returns: single [Rental](#rental), list of rentals, or availability-filtered reservations (`get_all_reservations`)

### GET /?route=rentals_calendar
- Auth: Bearer (owner only)
- Params: `month`, `year`, `rental_id` (all int required)
- Returns: per-room reservation calendar (`RentalRoomReservations::get_calendar`)

### GET /?route=rentals_cancellationPolicies
- Auth: none
- Params: optional `id=<int>`
- Returns: single [RentalCancellationPolicies](#rentalcancellationpolicies) or flat list

### GET /?route=rentals_charges
- Auth: none
- Returns: flat list of [RentalRatesCharges](#rentalratescharges)

### GET /?route=rentals_comments
- Auth: none
- Params: optional `id=<int>`
- Returns: single [RentalRatings](#rentalratings) or flat list

### GET /?route=rentals_settings
- Auth: none
- Returns: `{ commission_host, commission_client }` from admin settings

### GET /?route=rentals_statuses
- Auth: none
- Params: optional `id=<int>`
- Returns: single [RentalStatus](#rentalstatus) or flat list

### GET /?route=rentals_type
- Auth: none
- Params: optional `id=<int>`
- Returns: single [RentalType](#rentaltype) or flat list

### GET /?route=returns_conditions
- Auth: none
- Params: optional `id=<int>`
- Returns: single return-condition record or flat list (`ReturnsConditions`)

### GET /?route=returns_options
- Auth: none
- Params: optional `id=<int>`
- Returns: single [ReturnsOptions](#returnsoptions) or flat list

### GET /?route=saas_templates
- Auth: none
- Params: optional `id=<int>`
- Returns: single [SaasFrontendTemplates](#saasfrontendtemplates) or flat list

### GET /?route=sameday
- Auth: none
- Returns: flat list of Sameday offices (sorted by city/address)

### GET /?route=search
- Auth: none
- Params: `search` (string, required), optional category/type filters
- Returns: heterogenous `Search::search` result (products, blog, pages grouped)

### GET /?route=settings
- Auth: none
- Returns: sanitized `core` settings object (SMTP fields removed)

### GET /?route=settings_web
- Auth: none
- Returns: public web-only settings (`core.get_settings_web`)

### GET /?route=shipping_price
- Auth: none
- Params: `econt=1` + (`city`|`office_id`|`postCode`) or `speedy=1` + (`city`|`office_id`). Optional `price`.
- Returns: calculated shipping (`{ status, ... }`) or `{ status: 0 }` if courier not requested.

### GET /?route=sliders
- Auth: none
- Params: `region=<string>` (slider region), `id=<int>`, otherwise list
- Returns: [Slider](#slider) single / region content / flat list

### GET /?route=speedy
- Auth: none
- Returns: flat list of Speedy offices

### GET /?route=surveys
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Survey](#survey) or flat list

### GET /?route=tbi
- Auth: none
- Params: `amount` (required)
- Returns: TBI financing schemes for that amount

### GET /?route=time_slots
- Auth: none
- Params: optional `id=<int>`
- Returns: single [TimeSlot](#timeslot) or flat list

### GET /?route=time_slots_days
- Auth: none
- Params: `days` (int, required), `warehouse_id` (int, required)
- Returns: next N available days for that warehouse

### GET /?route=users
- Auth: Bearer
- Returns: single [Users](#users) (delegates to `accounts`)

### GET /?route=users_avatar
- Auth: Bearer
- Returns: single [UsersAvatar](#usersavatar) for current user

### GET /?route=users_files
- Auth: Bearer
- Returns: flat list of [UsersFiles](#usersfiles) for current user

### GET /?route=users_identity
- Auth: Bearer
- Params: optional filters
- Returns: flat list of [UsersIdentity](#usersidentity) for current user

### GET /?route=users_spoken_languages
- Auth: Bearer
- Params: optional `id=<int>`
- Returns: single spoken-language record or flat list

### GET /?route=warehouses
- Auth: none
- Params: optional `id=<int>`
- Returns: single [Warehouse](#warehouse) or flat list

### GET /?route=wishlist
- Auth: none
- Returns: paginated [Product](#product) favorites list (alias for `favourites`)

### GET /?route=withdraws
- Auth: Bearer
- Returns: flat list of [Withdraw](#withdraw) for current user

---

## 7. POST endpoints

### POST /?route=ads_requests
- Auth: Bearer
- Body: `ads_id` (required) + optional message
- Returns: process result with [AdsRequest](#adsrequest)

### POST /?route=advertisements
- Auth: Bearer
- Body: `type_id` (required), optional `id` (edit; owner only), ad fields
- Returns: process result with [Ads](#ads)

### POST /?route=blog_comments
- Auth: none (user_id stored if logged)
- Body: `comment`, `subject_id` (blog post id)
- Returns: process result with [Comments](#comments) (type=post, active=0 pending)

### POST /?route=carts
- Auth: Session OR Bearer
- Body (add product): `product_id` (int, required), optional `quantity|qty`, `variation_id`, `warehouse_id`
- Body (add combination): `combination_id` (int, required)
- Body (add time slot): `timeSlot_id`, `timeSlot_date`, optional `post_code`
- Returns: process result `{ status, cart, ... }` — updated [Cart](#cart)

### POST /?route=contacts
- Auth: none (email required if guest)
- Body: `message`, `subject`, `email` (guest)
- Returns: process result

### POST /?route=contacts_new
- Auth: none
- Body: `connection` (string, required), message payload
- Returns: process result (ContactsNew)

### POST /?route=deposits
- Auth: Bearer
- Body variants:
  - `continue=1` + `deposit_id`
  - `reinvest=1` + `deposit_id`, `amount`
  - New: `plan_id`, `payment_code`
- Returns: process result with [Deposit](#deposit)

### POST /?route=email_verification
- Auth: none
- Body: `code`, `email` OR `resend=1` + `email`
- Returns: `{ status, ... }`

### POST /?route=forgot_password
- Auth: none
- Body: `email`
- Returns: `{ status, ... }` — triggers password reset email

### POST /?route=forgot_password_link
- Auth: none
- Body: `code` (string, required — from the reset email), `password` (string, required), `password_confirm` (string, required)
- Returns: `{ status, ... }` from `Users::reset_password_by_code()`
- Notes: second step of the reset flow started by `forgot_password`.

### POST /?route=git
- Auth: none
- Body: `github_token`, `github_route` (`create_repo` | `change_visibility`) + route-specific fields (`owner`, `repo_name`, `visibility`)
- Returns: vendor payload from `GitHub::*`

### POST /?route=hotel_reservations
- Auth: none
- Body: `daterange`, `feeding_id`, `room_count`, `room_guests` (array), `room_types` (array), `payment_code`, optional `clients` (json string), optional `calculate=1`
- Returns: calculation result or created [HotelReservationsOnline](#hotelreservationsonline)

### POST /?route=hotel_rooms_pms
- Auth: Bearer + admin
- Returns: internal import (currently stubbed with `d('test'); die();` — not for frontend use)

### POST /?route=login
- Auth: none
- Body variants:
  - Email/password: `email`, `password`
  - Google: `google` (token) + optional profile
  - Facebook: `facebook` (token)
  - Apple: `apple` (token)
- Returns: login payload `{ status, token?, user?, error? }`. Successful login includes token for Bearer use.

### POST /?route=logout
- Auth: Bearer
- Returns: `{ status }` — current token revoked

### POST /?route=marketplace_orders
- Auth: none (checks session/user in process)
- Body: marketplace order payload
- Returns: process result with [MarketplaceOrder](#marketplaceorder)

### POST /?route=marketplace_sellers
- Auth: none
- Body: `title`, `email` (required)
- Returns: process result with [MarketplaceSeller](#marketplaceseller)

### POST /?route=meetings
- Auth: none; if guest, `name` + `email` required
- Body: `calendar_id`, `timeSlot_id`, `date_start`
- Returns: process result with [Meeting](#meeting)

### POST /?route=messages
- Auth: Bearer
- Body: `room_id` OR `to_user_id`, plus message fields (`text`, optional file)
- Returns: process result with appended [Messages](#messages)

### POST /?route=messages_log
- Auth: Bearer
- Body: `room_id` (marks room as read) OR `message_id` (marks single message read)
- Returns: `{ status }`

### POST /?route=my_marketplace_sellers
- Auth: Bearer
- Body: seller fields (frontend edit subset)
- Returns: updated [MarketplaceSeller](#marketplaceseller)

### POST /?route=orders
- Auth: Bearer OR guest session
- Body: full checkout payload (`shipping_address`, `payment_code`, `cart` fields, etc.)
- Returns: process result — created [Order](#order) (`Order::process`)

### POST /?route=payment_confirm
- Auth: none (varies by gateway)
- Body variants (mutually exclusive):
  - `order_id` → dispatched by the order's payment code: Stripe, ePay (needs `hash`), MyPos (needs `hash`), FiBank (needs `transaction_id`), TBI (needs `order_id`,`type`), Coinbase
  - `deposit_id` → Stripe / PayPal / Binance
  - `reservation_id` → Stripe (rental)
  - `exp_reservation_id` → Stripe (experience)
  - `hotel_reservation_id` → MyPos
- Returns: gateway-specific confirmation payload or `{ status: 0, error: "no_payment" }`

### POST /?route=product_comments
- Auth: none
- Body: `comment`, `product_id` (int, required)
- Returns: process result with [Comments](#comments) (type=product, active=0 pending moderation)

### POST /?route=promocode
- Auth: none
- Body: `promocode` (string, required)
- Returns: cart with applied [Promotion](#promotion)

### POST /?route=qa
- Auth: none (user_id attached when logged)
- Body: `product_id`, `text`, `user_name`, `subject`
- Returns: process result with [QaQuestion](#qaquestion)

### POST /?route=releva
- Auth: none
- Body: `page` (required), optional `product_id`, `relevaId`, `warehouse_id`
- Returns: `Releva::send_click` response

### POST /?route=rental_ratings
- Auth: Bearer
- Body: `rental_id`, `clean`, `accuracy`, `accommodation`, `communication`, `location`, `price_quality`, `comment`
- Returns: process result with [RentalRatings](#rentalratings)

### POST /?route=rentals
- Auth: Bearer
- Body (create): `type_id`, `title`, `qty`
- Body (edit): `id`, `type_id`, `title` (owner only)
- Returns: process result with [Rental](#rental)

### POST /?route=rentals_block
- Auth: Bearer (owner only)
- Body: `room_id`, `date_start`, `date_end`, optional `note`
- Returns: `RentalRoom::block` result

### POST /?route=rentals_copy
- Auth: Bearer
- Body: `id` (rental to copy)
- Returns: `Rental::copy` result (new rental)

### POST /?route=rentals_image_order
- Auth: Bearer
- Body: `image_order` (array/list of image ids)
- Returns: `{ status }`

### POST /?route=rentals_rooms
- Auth: Bearer (owner only)
- Body: `qty`, `rental_id`
- Returns: process result with added [RentalRoom](#rentalroom) items

### POST /?route=reservations
- Auth: `calculate=1` → none; else Bearer
- Body: `guests`, `date_start`, `date_end`, `rental_id`; without `calculate` also needs `payment_code` (defaulted to `stripe`)
- Returns: calculation object or process result with [RentalReservations](#rentalreservations)

### POST /?route=reservations_cancellation
- Auth: Bearer (owner only)
- Body: `reservation_id`
- Returns: `RentalReservationsCancellation::add` result

### POST /?route=reservations_confirm
- Auth: Bearer (host only)
- Body: `reservation_id`, `confirm` (bool-like)
- Returns: result of `RentalReservations::host_confirm`

### POST /?route=reservations_pay_url
- Auth: Bearer
- Body: `reservation_id`
- Returns: `{ url: "<payment url>" }`

### POST /?route=returns
- Auth: none
- Body: `option_id`, `condition_id`, `orders_items_id`
- Returns: process result with created [Returns](#returns) record

### POST /?route=revisions
- Auth: Bearer + superAdmin
- Body: `page_id`, optional `description`
- Returns: created revision record

### POST /?route=sms_verification
- Auth: Bearer
- Body: `code` (or `resend=1`)
- Returns: `{ status, ... }`

### POST /?route=speedy
- Auth: none
- Body: —
- Returns: `{ status: 1 }`
- Notes: maintenance trigger — refreshes the local Speedy offices cache (`Speedy::update_offices()`). Not for frontend use.

### POST /?route=stripe_webhook
- Auth: none (Stripe signed webhook)
- Body: Stripe event payload
- Returns: `{ status: 1|0 }`

### POST /?route=subscribe
- Auth: none
- Body: `email`
- Returns: `{ status }`

### POST /?route=tbi
- Auth: none
- Body: (stubbed — currently returns empty response; not for frontend use)

### POST /?route=token_verify
- Auth: none — not documented (internal magic-link verification)
- Body: `token`
- Returns: login result

### POST /?route=user_address
- Auth: Bearer (owner check when `id` supplied)
- Body: optional `id` (edit), plus `country_id`, `city`, `shipping_address`, etc.
- Returns: process result with [UserAddress](#useraddress)

### POST /?route=users
- Auth: none — registration
- Body: `password` (required), optional `password2`, `email`, profile fields. Sets `userlevel=1`, `active=1`.
- Returns: `Users::register_user_from_frontend` result

### POST /?route=users_avatar
- Auth: Bearer
- Body: multipart `image`
- Returns: uploaded [UsersAvatar](#usersavatar)

### POST /?route=users_bankinfo
- Auth: Bearer
- Body: `iban` (required), `bank_name`, `names`
- Returns: process result with [UsersBankInfo](#usersbankinfo)

### POST /?route=users_company
- Auth: Bearer
- Body: company fields; optional `seller_id` (admin override for seller's company)
- Returns: updated [UserCompany](#usercompany)

### POST /?route=users_files
- Auth: Bearer
- Body: multipart `files` (one or more)
- Returns: uploaded [UsersFiles](#usersfiles)

### POST /?route=users_identity
- Auth: Bearer
- Body: multipart `identity`, optional `override=1`, `back_side=1`, `type`
- Returns: [UsersIdentity](#usersidentity) record

### POST /?route=wishlist
- Auth: Bearer
- Body: `object_id` (int)
- Returns: `Favorites::add` result

### POST /?route=withdraws
- Auth: Bearer
- Body variants:
  - `dividents=1` + `amount`, `deposit_id` (amount ≤ sum of dividents)
  - `deposit_id` only → withdraws full deposit amount
  - Plain: `amount`
- Returns: process result with [Withdraw](#withdraw)

---

## 8. PUT endpoints

### PUT /?route=accounts
- Auth: Bearer
- Body: any editable user fields. If `invoice_country` present → must be int.
- Returns: `Users::edit_user` result with updated [Users](#users)
- Notes: alias `users` delegates here.

### PUT /?route=blogPosts
- Auth: Bearer + admin
- Body: `id`, `html`, `lang`
- Returns: `Blog::edit_blog_html` result
- Notes: there is a `die('1');` guard — currently inaccessible from frontend.

### PUT /?route=carts
- Auth: none
- Body: `variation_id` (scalar) + `qty`/`quantity`, optional `warehouse_id`; OR `variation_id` as object `{ variation_id: qty, ... }` for bulk.
- Returns: `{ status, obj: Cart }` — see [Cart](#cart)

### PUT /?route=company
- Auth: Bearer
- Body: company fields
- Returns: `UserCompany::update_company` → updated [UserCompany](#usercompany)

### PUT /?route=orders
- Auth: none (legacy behaviour)
- Params: `id` (required). Body: `order_status` (int) and/or `payment_method` (int)
- Returns: `{ status: 1, obj: Order }` — see [Order](#order)
- Notes: intended for internal integrations.

### PUT /?route=user_address
- Auth: Bearer (owner only)
- Body: `id`, optional `default=1` (marks address as default) OR regular fields
- Returns: process / `setAsDefault` result with [UserAddress](#useraddress)

### PUT /?route=users
- Auth: Bearer
- Returns: delegates to `accounts`

---

## 9. DELETE endpoints

### DELETE /?route=ads_images
- Auth: Bearer (image owner)
- Params: `id`
- Returns: `{ status }`

### DELETE /?route=advertisements
- Auth: Bearer (ad owner)
- Params: `id`
- Returns: `{ status }`

### DELETE /?route=carts
- Auth: none (session or logged)
- Params: `id` (variation id; optional — when missing clears cart entry)
- Returns: `{ ..., cart: Cart }`

### DELETE /?route=cartAllProducts
- Auth: none
- Returns: `{ cart: Cart }` with cart fully emptied

### DELETE /?route=cartsEmpty
- Auth: none
- Returns: `{ status: 1, cart: Cart }`

### DELETE /?route=cart_combination
- Auth: none
- Params: `id` (combination id)
- Returns: `{ ..., cart: Cart }`

### DELETE /?route=my_account
- Auth: Bearer
- Body: `password` (required)
- Returns: `Users::delete_myAccount` result

### DELETE /?route=promocode
- Auth: none
- Returns: `Promotion::remove_promocode` result

### DELETE /?route=rental_reservations
- Auth: Bearer (owner; currently no-op — retained for backwards compatibility)
- Params: `id`
- Returns: empty (handler currently does not emit response — avoid use)

### DELETE /?route=rentals
- Auth: Bearer (owner)
- Params: `id`
- Returns: `{ status }`

### DELETE /?route=rentals_block
- Auth: Bearer (rental owner)
- Params: `id` (room reservation id; must have status=BLOCKED)
- Returns: `{ status }`

### DELETE /?route=rentals_images
- Auth: Bearer (image owner)
- Params: `id`
- Returns: `{ status }`

### DELETE /?route=rentals_seasons
- Auth: Bearer (owner)
- Params: `id`
- Returns: `{ status }`

### DELETE /?route=rentals_special
- Auth: Bearer (owner)
- Params: `id`
- Returns: `{ status }`

### DELETE /?route=subscribe
- Auth: none
- Body: `email`
- Returns: `NewsletterUser::unsubscribe` result

### DELETE /?route=user_address
- Auth: Bearer (owner)
- Params: `id`
- Returns: `{ status }`

### DELETE /?route=users_identity
- Auth: Bearer (access check)
- Params: `id`
- Returns: `{ status }`

### DELETE /?route=wishlist
- Auth: Bearer
- Params: `id` (object id)
- Returns: `Favorites::delete_byUserAndObject` result

---

## 10. Models

Field lists below are the public typed properties of each model class as returned by the v4 API. Types come from PHP property declarations. `date_*` columns are strings (MySQL format `YYYY-MM-DD HH:MM:SS`).

### Ads
Source: `lib/classes/ads/class.ads.php`
- id: int
- user_id: int
- views: int
- view_phone: int
- vip: bool
- title: string
- subtitle: string
- description: string
- tags: string
- type_id: int
- status_id: int
- category_id: int
- date_created
- date_updated
- images: array
- fields: array
- user_info: array

### AdsCategory
Source: `lib/classes/ads/class.adsCategory.php`
- id: int
- parent_id: int
- title: string
- slug: string
- body: string
- date_created
- position: int
- active: int
- image: AdsCategoryImage

### AdsField
Source: `lib/classes/ads/class.adsField.php`
- id: int
- type: string
- title: string
- options: array
- value: string
- required: int

### AdsPlan
Source: `lib/classes/ads/class.adsPlan.php`
- id: int
- title: string
- time: int
- price: float
- date_created
- status_id: int

### AdsRequest
Source: `lib/classes/ads/class.adsRequest.php`
- id: int
- ads_id: int
- user_id: int
- date_created

### AdsType
Source: `lib/classes/ads/class.adsType.php`
- id: int
- title: string
- date_created
- date_updated
- fields: array
- ads_count: int

### Auction
Source: `lib/classes/auction/class.auction.php`
- id: int
- bids: array
- user_id: int
- price_start: float
- price_buyNow: float
- product_id: int
- product: Product
- status_id: int
- date_start
- date_end
- date_created
- date_updated
- Constants: STATUS_NEW=0, STATUS_COMPLETED=1

### Banners
Source: `lib/classes/banners/class.banners.php`
- id: int
- link: string
- active: int
- popup: int
- popup_timer: int
- page_id: int
- script: string
- title: string
- image: BannersImage
- date_created
- date_updated

### Blog
Source: `lib/classes/blog/class.blog.php`
- id: int
- title: string
- slug: string
- description: string
- description_short: string
- date_publish
- date_created
- views: int
- active: int
- main_article: int
- seo_title: string
- seo_tags: string
- seo_description: string
- related_product: int
- url: string
- css: string
- facebookTags: string
- twitterTags: string
- linkedInTags: string
- editor_url: string
- read_time: int
- cid: int
- tags: array
- related_posts: array
- category: BlogCategory
- attributes: array
- images: array

### BlogCategory
Source: `lib/classes/blog/class.blog.category.php`
- id: int
- title: string
- slug: string
- position: int
- active: int
- parent_id: int
- url: string

### Brands
Source: `lib/classes/brands/class.brands.php`
- id: int
- title: string
- description: string
- active: int
- providers: array
- images: array

### Cart
Source: `lib/classes/shop/cart/class.cart.php` (or `lib/classes/shop/class.cart.php`)
- id: int
- user_id: int
- sesid: string
- promocode_id: int
- discount: float
- date_created
- pos_table_id: int
- subtotal: float
- subtotal_without_disc: float
- shipping: float
- discount_total: float
- total_quantity: float
- total: float
- promocode_info: Promotion
- products: array (of Product with `variation`, `quantity`, `total`)
- combinations: array

### Cities
Source: `lib/classes/cities/class.cities.php` (or root classes)
- id: int
- country_id: int
- post_code: int
- important: bool
- region_id: int
- title: string

### Combination
Source: `lib/classes/shop/combination/class.combination.php`
- id: int
- slug: string
- title: string
- description: string
- status_id: int
- unit_id: int
- date_created
- date_updated
- url: string
- images: array
- categories: array
- products: array
- unit: Unit

### Comments
Source: `lib/classes/comments/class.comments.php`
- id: int
- type: string  (`product` | `post`)
- subject_id: int
- comment: string
- date
- user_id: int
- user_name: string
- parent_id: int
- rating: int
- status_id: int
- about: array
- email: string
- date_created
- date_updated

### Country
Source: `lib/classes/class.country.php`
- id: int
- region_id: int
- code: string
- title: string
- vat: int
- flag: string
- iso3: string
- phoneCode: string
- region: Region

### Currency
Source: `lib/classes/class.currency.php`
- id: int
- code: string
- symbol: string
- value: float
- status_id: int
- auto_upd: int
- date_update
- rate: float

### Deposit
Source: `lib/classes/deposits/class.deposit.php`
- id: int
- user_id: int
- amount: float
- plan_id: int
- status_id: int
- payment_status_id: int
- payment_code: string
- reinvest_deposit_id: int
- date_created
- date_updated
- amount_dividents: float
- reinvest_allowed: int
- reinvest_days: int
- plan: DepositPlans
- Constants: STATUS_NEW=0, STATUS_ACTIVE=1, STATUS_INACTIVE=2, STATUS_EXPIRED=3, STATUS_COMPLETED=5, PAYMENT_STATUS_UNPAID=0, PAYMENT_STATUS_PAID=1

### DepositDivident
Source: `lib/classes/deposits/class.depositDivident.php`
- id: int
- user_id: int
- deposit_id: int
- principal: float
- amount: float
- percent: float
- date: string
- hour: string
- date_created

### DepositPlans
Source: `lib/classes/deposits/class.depositPlans.php`
- id: int
- title: string
- description: string
- amount: float
- percent_min: int
- percent_max: int
- accrual_type: int
- status_id: int
- period: int
- date_created
- date_updated

### FAQ
Source: `lib/classes/faq/class.faq.php`
- id: int
- status_id: int
- category_id: int
- question: string
- answer: string
- date_created
- date_updated

### Favorites
Source: `lib/classes/class.favorites.php`
- id: int
- user_id: int
- object_id: int
- date_created

### Gallery
Source: `lib/classes/gallery/class.gallery.php`
- id: int
- category_id: int
- title: string
- description: string
- image: GalleryImage

### GalleryCategory
Source: `lib/classes/gallery/class.galleryCategory.php`
- id: int
- page_id: int
- title: string
- gallery: array

### HotelReservationsOnline
Source: `lib/classes/hotel/class.hotelReservationsOnline.php`
- id: int
- user_id: int
- price: float
- date_created
- date_updated
- reservations: array (line items, each with `payment_code` etc.)

### HotelRoomType
Source: `lib/classes/hotel/class.hotelRoomType.php`
- id: int
- title: string
- title_online: string
- description: string
- desc_bed: string
- room_count: int
- guests: int
- extra_bed: int
- show_on_web: int
- quadrature: float
- channel_manager: int
- channel_manager_reserv: int
- images: array
- properties: array
- date_created
- date_updated

### Invoice
Source: `lib/classes/invoice/class.invoice.php`
- id: int
- company: string
- eik: string
- vat_number: string
- vat_number_code: string
- vat_number_dig: string
- vat_registered: int
- country_id: int
- city: string
- address: string
- phone: string
- names: string
- order_id: int
- class_name: string
- mol: string
- receiver: string
- number: string
- hash: string
- tourist_tax: float
- user_id: int
- status_id: int
- amount: float
- amount_noVat: float
- amount_vat: float
- slovom: string
- public_url: string
- deliverer_company: UserCompany
- accepter_company: UserCompany
- items: array
- date_created
- date_updated
- payments: array
- country: Country
- Constants: STATUS_ACTIVE=1, STATUS_INACTIVE=0

### Language
Source: `lib/classes/lang/class.lang.php`
- language: string
- dblang: string
- langlist: array
- langdir: string
- orig_language: string
- Constants: DEF_LANG="bg"

### LoyaltyCard
Source: `lib/classes/loyalty/class.loyaltyCard.php`
- id: int
- card_number: string
- user_id: int
- discount: float
- tokens: int

### MarketplaceOrder
Source: `lib/classes/marketplace/class.marketplace.order.php`
- id: int
- order_ids: string
- user_id: int
- orders: array

### MarketplaceSeller
Source: `lib/classes/marketplace/class.marketplace.seller.php`
- id: int
- user_id: int
- public: int
- title: string
- status_id: int
- verified: bool
- commission: float
- delivery_days: int
- country_id: int
- tiktok: string
- instagram: string
- top_seller: bool
- orders_count: int
- date_created
- date_updated
- bank_info: UsersBankInfo
- user: Users
- company: UserCompany
- avatar: UsersAvatar
- country: Country
- Constants: STATUS_NEW=0, STATUS_APPROVED=1, STATUS_NOTAPPROVED=2

### Meeting
Source: `lib/classes/meetings/class.meeting.php`
- id: int
- calendar_id: int
- timeSlot_id: int
- date_start: string
- name: string
- email: string
- phone: string
- message: string
- service: string
- user_id: int
- date_created
- date_updated

### MeetingCalendar
Source: `lib/classes/meetings/class.meeting.calendar.php`
- id: int
- title: string
- interval: int
- status_id: int
- description: string
- link: string
- date_created
- date_updated
- day0..day6: bool  (weekday enabled flags)
- dayOfWeek: array
- time_slots: array
- days: array

### Membership
Source: `lib/classes/membership/class.membership.php`
- id: int
- title: string
- product_id: int
- membership_type_id: int
- provider_action: int
- product_type_action: int
- product_type: int
- product_title: string
- price: float
- currency: string
- status: string
- membership_type: MembershipType
- provider: Provider
- Constants: TYPE_ACTION_ALL=0, TYPE_ACTION_PHISICAL=1, TYPE_ACTION_DIGITAL=2

### Menu
Source: `lib/classes/class.menu.php`
- id: int
- title: string
- parent_id: int
- slug: string
- active: int
- position: int
- icon: string
- img: string
- warehouse_id: int
- cattree: array (nested categories under this menu item)

### Messages
Source: `lib/classes/messages/class.messages.php`
- id: int
- user_id: int
- room_id: int
- text: string
- is_file: bool
- file: MessagesFile
- user_info: MessagesUserInfo
- date_created

### MessagesRooms
Source: `lib/classes/messages/class.messages.rooms.php`
- id: int
- name: string
- public: bool
- object_id: int
- object_class: string
- date_created
- new_messages: int
- users_info: array
- messages: array

### NPosTable
Source: `lib/classes/pos_new/class.pos.table.php`
- id: int
- object_id: int
- price: float
- title: string
- hash: string
- opened_bills: array
- public_url: string
- date_created
- date_updated

### Order
Source: `lib/classes/shop/order/class.order.php` (or `lib/classes/shop/class.order.php`)
- id: int
- user_id: int
- sesid: string
- cart_id: int
- number: string
- fast_order: int
- first_name: string
- last_name: string
- payment_code: string
- payment_submethod: string
- shipping_info: int
- email: string
- phone: string
- country: string
- country_id: int
- city: string
- city_id: int
- post_code: string
- state: string
- shipping_address: string
- shipping_address2: string
- officeId: int
- shipping_company: string
- comment: string
- shipping_price: float
- email_was_sent: bool
- status_id: int
- payment_status_id: int
- invoice: int
- date_created
- price: float
- currency_code: string
- currency_rate: float
- vat: float
- pos: int
- reduce_qty: int
- tracking_number: string
- promocode_id: int
- mobileApp: int
- total_price: float
- seller_id: int
- status: OrderStatus
- address: UserAddress
- payment_method: Payment
- currency: Currency
- products: array (of Product entries with `variation`, `quantity`, `total`, `order_item_id`)
- options: array
- Constants: PAYMENT_STATUS_UNPAID=0, PAYMENT_STATUS_PAID=1

### OrderStatus
Source: `lib/classes/shop/order/class.orderStatus.php`
- id: int
- title: string
- date_created
- date_updated
- Constants: STATUS_CANCEL=0, STATUS_NEW=1, STATUS_PROCESSING=2, STATUS_OUTOFSTOCK=3, STATUS_PAID=4, STATUS_SEND=5, STATUS_PREPARED=6, STATUS_COMPLETED=7

### Pages
Source: `lib/classes/pages/class.pages.php`
- id: int
- title: string
- slug: string
- category_id: int
- cover_photo: string
- active: int
- editable: bool
- deletable: bool
- url: string
- private: bool
- saasModule_id: int
- seo_title: string
- seo_tags: string
- seo_description: string
- banners: array
- popups: array
- surveys: array
- category: PageCategory
- Constants: DEFAULT_PAGES=[1..21, 100, 101] (system-reserved ids)

### Partner
Source: `lib/classes/partner/class.partner.php`
- id: int
- title: string
- link: string
- status_id: int
- discount: float
- image: PartnerImage
- date_created

### Payment
Source: `lib/classes/payment/class.payment.php`
- id: int
- code: string
- icon: string
- title: string
- description: string
- active: int
- position: int
- fiscal: bool
- values: array
- subMethods: array
- address: string
- qr: string
- Payment IDs: ID_PAYPAL=1, ID_BANK_TR=2, ID_EPAY=3, ID_CARD=4, ID_COD=5, ID_VOUCHER=6, ID_STRIPE=7, ID_MYPOS=8, ID_GET_SPOT=9, ID_FIBANK=10, ID_CASH=11, ID_TBI=13, ID_COINBASE=14, ID_WALLET=15, ID_REVOLUT=16, ID_BINANCE=17, ID_USDT_TRC20=18, ID_USDT_ERC20=19, ID_WALLET_HOTELAGENTS=20, ID_WALLET_TOKENS=21, ID_NOWPAYMENT=22
- Payment codes: paypal, bank_transfer, epay, card, cod, vaoucher, stripe, mypos, get_on_spot, fibank, cash, tbi, coinbase, wallet, revolut, binance, usdt_trc20, usdt_erc20, wallet_agent, wallet_tokens, nowpayment
- Payment status: STATUS_UNPAID=0, STATUS_PAID=1

### Product
Source: `lib/classes/shop/product/class.product.php`
- id: int
- id_code: string
- type: int  (see constants below)
- slug: string
- title: string
- short_body: string
- description: string
- table_info: string
- tags: string
- length: float
- width: float
- height: float
- weight: float
- focus: bool
- status_id: int
- views: int
- date_created
- date_updated
- seo_title: string
- seo_tags: string
- seo_description: string
- made_of: string
- label_info: string
- model_info: string
- ref_number: string
- deleted: bool
- external_id: string
- media: string
- images: array
- url: string
- isWishlisted: bool
- isNew: bool
- brand_id: int
- boost: int
- row_id: int
- user_id: int  (adder)
- condition_id: int
- single_promo_active: ProductSinglePromotion
- brand: Brands
- providers: array
- seller_id: int
- seller: MarketplaceSeller
- price_min: float
- variations: ProductVariation[]
- categories: ProductCategory[]
- barcodes: array
- subProducts: array
- quantity_discounts: array  (used in orders/cart)
- variation: ProductVariation  (selected variation, used in orders/cart)
- quantity: float  (used in orders/cart)
- total: float  (used in orders/cart)
- warehouse_id: int  (used in orders/cart)
- order_item_id: int  (used in orders)
- rating: int
- files: array
- files_private: array
- features: array
- Constants: PRODUCT_TYPE_SIMPLE=1, PRODUCT_TYPE_OPTIONS=2, PRODUCT_TYPE_DIGITAL=3, PRODUCT_TYPE_SERVICE=4, PRODUCT_TYPE_MEMBERSHIP=5, PRODUCT_TYPE_COMBINATION=6

### ProductAttribute
Source: `lib/classes/shop/product/class.productAttribute.php`
- id: int
- title: string
- type: string
- values: ProductAttributeValue[]
- global: bool
- user_id: int

### ProductAttributeValue
Source: `lib/classes/shop/product/class.productAttributeValue.php`
- id: int
- attribute_id: int
- value: string
- color: string

### ProductCategory
Source: `lib/classes/shop/product/class.productCategory.php`
- id: int
- parent_id: int
- title: string
- slug: string
- description: string
- position: int
- active: bool
- focus: bool
- boost: int
- url: string
- date_created
- date_updated
- image: ProductCategoryImage
- icon: ProductCategoryIcon
- cat_paths: array
- parents: array

### ProductFeatures
Source: `lib/classes/shop/product/features/class.productFeatures.php`
- id: int
- title: string
- category_id: int
- values: array
- value: ProductFeaturesValues

### ProductVariation
Source: `lib/classes/shop/product/class.productVariation.php`
- id: string
- product_id: int
- price: float
- delivery_price: float
- rec_price: float
- sku: string
- unit_id: int
- file: string
- external_id: string
- selling_price: float
- promoprice: float
- discount: float
- unit: Unit
- currency: string
- qty: float
- name: string
- attributes: ProductAttributeValue[]

### ProjectCompany
Source: `lib/classes/class.project.company.php`
- id: int  (always 1)
- phone: string
- country_id: int
- mol: string
- eik: string
- vat_number: string
- dop: int
- city: string
- address: string
- name: string
- general_regulations: string
- country: Country

### Promotion
Source: `lib/classes/shop/promotion/class.promotion.php`
- id: int
- promocode: string
- name: string
- type: int
- discount_type: string
- discount: float
- discount_subject: string
- discount_subject_ids: string
- used: int
- validaty: int
- date_created
- date_started
- date_expires
- active: int
- user_id: int
- order: int
- categories: array
- products: array
- once_used: int
- Constants: TYPE_PROMOCODE=1, TYPE_ALL=2, TYPE_REGISTRATION=4; DISCOUNT_SUBJECTS: product, all_orders, category, orders_over, collection; DISCOUNT_TYPE: fixed, percent, free_shiping

### Provider
Source: `lib/classes/provider/class.provider.php`
- id: int
- title: string
- address: string
- phone: string
- phone_second: string
- email_second: string
- delivery: int
- markup: float
- markup_min: int
- shipping_price: float
- shipping_price_free: float
- image: ProviderImage
- status_id: int
- user_id: int
- company_id: int
- terms: string
- hasMembership: bool
- user: UserInfoSimple

### QaQuestion
Source: `lib/classes/qa/class.qa.php`
- id: int
- product_id: int
- user_id: int
- user_name: string
- date: string
- status_id: int
- visible: int
- parent_id: int
- subject: string
- text: string
- answer: Qa

### Region
Source: `lib/classes/class.region.php`
- id: int
- title: string
- code: string
- language_id: int
- currency_id: int
- percent: int
- show_vat: bool
- status_id: int
- language: Language
- currency: Currency

### Rental
Source: `lib/classes/rental/class.rental.php`
- id: int
- user_id: int
- status_id: int
- type_id: int
- sahring_options_id: int
- qty: int
- guests: int
- bedrooms: int
- bathrooms: int
- checkIn_start: string
- checkIn_end: string
- checkOut: string
- cancellation_id: int
- reservation_type: int
- address_id: int
- slug: string
- baat: bool
- green_house: bool
- address: RentalAddress
- title: string
- description: string
- views: int
- date_created
- date_updated
- deleted: bool
- rental_rating: float
- top_rental: bool
- rental_info: array
- is_wishlisted: bool
- url: string
- type: RentalType
- rooms: RentalRoom[]
- rates_seasons: array
- rates_special: array
- images: array
- desc_facilities: string
- desc_services: string
- desc_rules: string
- desc_location: string
- desc_activities: string
- desc_instructions: string
- groups_facilities: array
- groups_services: array
- groups_rules: array
- groups_location: array
- groups_activities: array
- user_info: RentalUserInfo
- charges: array
- status: RentalStatus
- Constants: TYPE_DIRECT=1, TYPE_CONFIRM=2

### RentalCancellationPolicies
Source: `lib/classes/rental/class.cancellationPolicies.php`
- id: int
- step_checkin_days: int
- step_checkin_perc: int
- step_2_days: int
- step_2_perc: int
- step_3_perc: int
- title: string
- description: string
- Constants: ID_STRICT=1, ID_MODERATE=2

### RentalRatesCharges
Source: `lib/classes/rental/class.ratesCharges.php`
- id: int
- title: string
- percent: bool

### RentalRatings
Source: `lib/classes/rental/class.rentalRatings.php`
- id: int
- rental_id: int
- user_id: int
- clean: int
- accuracy: int
- accommodation: int
- communication: int
- location: int
- price_quality: int
- comment: string
- user_info: RentalUserInfo
- date_created

### RentalReservations
Source: `lib/classes/rental/class.reservations.php`
- id: int
- rental_id: int
- payment_status_id: int
- payment_host: bool
- payment_code: string
- user_id: int
- price: float
- price_for_host: float
- tax_client: float
- guests: int
- children: int
- infants: int
- pets: int
- date_created
- date_updated
- note: string
- user_info: RentalUserInfo
- client_info: RentalUserInfo
- cancellation: RentalReservationsCancellation
- payment: RentalReservationsPayments
- calculation: array
- room_reservations: array (each has `room_id`, dates, etc.)

### RentalReservationsStatus
Source: `lib/classes/rental/class.reservationsStatuses.php`
- id: int
- title: string
- Constants: STATUS_INACTIVE=0, STATUS_ACTIVE=1, STATUS_BLOCKED=2, STATUS_CANCELED=3, STATUS_COMPLETED=4, STATUS_HOST_UNCORFIRMED=5, STATUS_CANCELED_BYHOST=6, STATUS_EXPECTED_PAYMENT=7

### RentalRoom
Source: `lib/classes/rental/class.room.php`
- id: int
- rental_id: int
- status_id: int
- title: string
- reservations: array
- Constants: STATUS_INACTIVE=0, STATUS_ACTIVE=1

### RentalStatus
Source: `lib/classes/rental/class.rentalSatus.php`
- id: int
- title: string
- Constants: STATUS_ACTIVE=1, STATUS_INACTIVE=2, STATUS_INCOMPLETED=3

### RentalType
Source: `lib/classes/rental/class.rentalType.php`
- id: int
- title: string
- description: string
- room_title: string
- rental_count: int
- rental_count_ctry: int
- image: RentalTypeImage
- date_created
- date_updated

### Returns
Source: `lib/classes/returns/class.returns.php`
- id: int
- product_id: int
- shop_orders_items_id: int
- order_id: int
- user_id: int
- status_id: int
- option_id: int
- condition_id: int
- date_created
- date_paid
- iban: string
- iban_name: string
- note: string
- Constants: COMPLETED_STATUSES=[3, 6]

### ReturnsOptions
Source: `lib/classes/returns/class.returnsOptions.php`
- id: int
- title: string
- date_created: string

### SaasFrontendTemplates
Source: `lib/classes/saas_other/frontTemplates/class.saas.front.templates.php`
- id: int
- github_folder: string
- code: string
- category_id: int
- title: string
- description: string

### Slider
Source: `lib/classes/slider/class.slider.php`
- id: int
- title: string
- date_created
- active: int
- region_id: int
- background: array
- buttons: array
- layers: array
- slides: array

### Survey
Source: `lib/classes/survey/class.survey.php`
- id: int
- title: string
- status_id: int
- page_id: int
- questions: array
- date_created
- date_updated

### TimeSlot
Source: `lib/classes/warehouse/class.timeslot.php`
- id: int
- start: string
- end: string
- quantity: int
- warehouse_id: int
- free_shipping: float
- min_price: float
- day0..day6: int (weekday enabled flags)
- active: int
- amountUntilActivation: float
- amountUntilFreeShipping: float
- postcodes: array
- warehouse: Warehouse

### UserAddress
Source: `lib/classes/users/class.users.address.php`
- id: int
- user_id: int
- country_id: int
- shipping_address: string
- shipping_address2: string
- city: string
- state: string
- post_code: string
- default: int  (1 = default address)
- long: string
- lat: string
- neighborhood: string
- streetName: string
- streetNumber: string
- blok: string
- input: string
- apartment: string
- country: Country

### UserCompany
Source: `lib/classes/users/class.users.company.php`
- id: int
- user_id: int
- email: string
- name: string
- city: string
- post_code: string
- country_id: int
- mol: string
- vat: string
- vat_number_code: string
- vat_number_dig: string
- eik: string
- address: string
- phone: string
- phone_code: string
- vat_registered: int
- company_registered: int

### Users
Source: `lib/classes/users/class.users.php`
- id: int
- email: string
- sesBeforeLogin: string
- first_name: string
- middle_name: string
- last_name: string
- phone_code: string
- phone: string
- gender: string
- birthday: string
- egn: string
- nationality: string
- passportType: string
- passportId: string
- passportCountry: string
- mps: string
- userlevel: int  (see constants)
- group_id: int
- referral_link: string
- active: int
- date_created
- date_updated
- lastlogin
- lastip: string
- token: string  (Bearer access token)
- email_verified: int
- phone_verified: int
- fp_id: int
- pin: string
- sesid: string
- defaultAddressId: int
- balance: float
- tokens: int
- group: UserGroup
- admin2FA: Users_Admin2FA
- warehousesAccess: array
- Levels: SUPERADMIN=100, ADMIN=99, DOCTOR=13, AUDITOR=12, A1=11, HOTEL_RECEPTION=10, RENTAL_HOST=9, HOTEL_AGENT=8, MARKETPLACE_SELLER=7, VENDOR=6, WAREHOUSE=5, MODERATOR=4, COURIER=3, POS_SELLER=2, SIMPLE_USER=1, POS_USER=-1, HOTEL_CLIENT=-2

### UsersAvatar
Source: `lib/classes/users/class.users.avatar.php`
- (extends SingleImage — exposes `url`, `path`, `mime`, `size`, `width`, `height`, `title`, `alt`)

### UsersBankInfo
Source: `lib/classes/users/class.users.bank_info.php`
- id: int
- user_id: int
- bank_name: string
- iban: string
- names: string

### UsersFiles
Source: `lib/classes/users/class.users.files.php`
- (extends SimpleFile — exposes `id`, `object_id`, `url`, `path`, `filename`, `ext`, `size`, `mime`, `date_created`)

### UsersIdentity
Source: `lib/classes/users/class.users.identity.php`
- back_side: bool
- type: string  (e.g. passport, id)
- (plus base SimpleFile fields: id, url, path, mime, size, date_created)

### UsersSpokenLanguages
Source: `lib/classes/users/class.spokenLanguages.php`
- id: int
- title: string

### UsersWallets
Source: `lib/classes/wallets/class.usersWallets.php`
- id: int
- user_id: int
- hash: string
- balance: float
- tokens: int
- date_created
- date_updated

### UsersWalletsTransactionsAmount
Source: `lib/classes/wallets/` (see folder)
- Transactions payload — monetary movement entries. Key fields typically: `id`, `user_id`, `amount`, `type`, `description`, `balance_after`, `date_created`.

### Warehouse
Source: `lib/classes/warehouse/class.warehouse.php`
- id: int
- title: string
- country_id: int
- date_created
- date_updated
- user_id: int
- region_id: int
- delivery_rates: array
- address: string
- min_qty: int
- subwarehouse_id: int
- country: Country

### Withdraw
Source: `lib/classes/withdraw/class.withdraw.php`
- id: int
- user_id: int
- amount: float
- status_id: int
- note: string
- date_created
- date_updated
- Constants: STATUS_NEW=0, STATUS_COMPLETED=1, STATUS_CANCEL=2, STATUS_SYSTEM=3

---

## 11. Vendor / integration payloads

These endpoints return provider-specific shapes, not managed domain models. Consult the PHP class for specifics if the exact contract is required.

- **Econt** (`lib/classes/econt/`) — cities, offices, shipping quotes
- **Speedy** (`lib/classes/speedy/`) — offices, shipping quotes
- **Sameday** (`lib/classes/sameday/`) — offices list (array of `{ city, address, ... }`)
- **ePay** (`lib/classes/epay/`) — HTML form fields for redirect
- **MyPos, FiBank, Stripe, PayPal, Binance, Coinbase, NowPayment, TBI** (`lib/classes/*`) — payment confirm responses
- **HybridAuth** (`lib/classes/hybridauth/`) — OAuth provider config
- **Releva** (`lib/classes/releva/`) — recommendation click/feed payloads
- **GitHub** (`lib/classes/gitHub/`) — proxied GitHub API responses
- **AuditFileNAP** (`lib/classes/auditfile/`) — tax audit export
- **QRtoken** — `{ status: 1, QR: "<base64 png>" }`

---

## 12. LLM usage hints (frontend)

1. **Identifiers:** single-record GETs accept `?query_id=N` or `?id=N`. Body `id` is equivalent for mutating methods.
2. **Bearer header must win over Session.** If the user logs in, attach `Authorization: Bearer <token>` from the login response and drop any Session header.
3. **Pagination defaults:** endpoints that return paginated shapes honour `page`, `limit`. Pass `no_pagination=1` to flatten.
4. **Ordering / filtering:** `order_by=<field>` + `sortDirection=ASC|DESC`; filter by any DB-backed field on the model (e.g. `status_id=1`). Text filters on `title` are `LIKE %...%` (case-insensitive).
5. **Dates:** `date_created_from=YYYY-MM-DD`, `date_created_to=YYYY-MM-DD` are accepted on paginated list endpoints.
6. **File uploads:** always `multipart/form-data`; keep the field name exactly as specified in the endpoint notes (`files`, `image`, `identity`).
7. **Errors:** handle the two shapes separately — `{ status: 0, errors: { '[name="x"]': "..." } }` for form fields, vs `{ errors: ["..."] }` for generic HTTP errors.
8. **Single-object 404:** `output_single` returns `{}` + HTTP 404 for a missing record, not `null`.
9. **VAT:** product pricing is adjusted server-side when `core.price_with_vat` is false; the frontend should not re-apply VAT logic.
10. **Cart identity:** anonymous cart is tied to the `Authorization: Session <sesid>` value; persist the same sesid across page loads until login, then switch to Bearer.
