
# Maevia — Custom API

Документация за къстъм endpoint-ите на maevia (`projects/maevia/api/front/*.php` в devcore). Ядрените endpoint-и (login, accounts…) са в `core-v4-front.md`.

Фронтендът НЯМА собствена база. Всичко минава по HTTP към ядрото (devcore).
Този документ описва какво връща ядрото и как се вика.

---

## 1. Свързване

```
База (дев):  https://devcore.myexpozy.com/api/
```

Всяка заявка носи хедър, който казва **кой проект** е:

```
Authentication: Basic <saas_key>
```

`saas_key` е един за проекта; НЕ се хардкодва в клиентски код и НЕ се показва в HTML —
държи се на сървъра на фронтенда и заявките минават през негов прокси слой.

За действия от името на потребител се добавя и:

```
Authorization: Bearer <token>
```

Токенът идва от `POST /login` (ядрен endpoint) и се пази в HttpOnly cookie.
Не го слагай в DOM, URL или localStorage.

### Формат на отговорите

- Списък без пагинация → чист масив: `[{...}, {...}]`
- Списък с пагинация → `{"pagination": {"total_results", "total_pages", "current_page", "results_per_page"}, "result": [...]}`
- Единичен запис → обект; несъществуващ → HTTP 404
- Запис/промяна → `{"status": 1, "obj": {...}, "msg": "..."}`
- Грешка по поле → `{"status": 0, "[name=\"price\"]": "Моля, въведете"}`
- Липсваща автентикация → `{"status": 0, "errors": ["Login first v4!"]}`

Пагинацията се управлява с `?page=2&limit=20`, сортирането с `?order_by=price&order_direction=ASC`.

---

## 2. Номенклатури (справочници)

Всички връщат масив от `{id, title, sort, active, date_created, date_updated}`,
подредени по `sort`, само активните (`active = 1`). Заглавията идват от системата
за съдържание, т.е. са преводими — при многоезичен сайт добави `?lang=en`.

| Endpoint | Съдържание |
|---|---|
| `GET /maevia_types` | вид имот (Едностаен, Къща, Парцел…); има `type_category_id` и `fields` |
| `GET /maevia_types_categories` | групи видове (Жилищни / Бизнес / Други) — за optgroup в селекта |
| `GET /maevia_listing_types` | вид обява (Продава / Отдава под наем) |
| `GET /maevia_statuses` | статус на обявата (виж §5) |
| `GET /maevia_deal_statuses` | статус на сделката (Свободен / Резервиран / Продаден) |
| `GET /maevia_conditions` | състояние (Ремонтиран, За ремонт…) |
| `GET /maevia_constructions` | конструкция (Тухла, Панел, ЕПК…) |
| `GET /maevia_construction_stages` | етап на строеж (Кота 0, Акт 14/15/16…) |
| `GET /maevia_furnitures` | обзавеждане |
| `GET /maevia_overheads` | режийни |
| `GET /maevia_parkings` | паркинг |
| `GET /maevia_heatings` | отопление (мултиселект) |
| `GET /maevia_exteriors` | екстериор (мултиселект) |
| `GET /maevia_exhibitions` | изложение (мултиселект) |
| `GET /maevia_purposes` | предназначение (мултиселект) |
| `GET /maevia_protections` | охрана (мултиселект) |
| `GET /maevia_extras` | екстри (мултиселект) |
| `GET /maevia_equipments` | уреди и обзавеждане при наем (мултиселект); има и `icon` — име на иконата от набора на сайта, може да е празно |
| `GET /maevia_tenant_types` | „подходящ за" при наем — семейство, двойка, фирма, студенти (мултиселект) |
| `GET /maevia_rent_terms` | вид наем: `1` дългосрочен, `2` краткосрочен. Фиксиран списък, връща само `{id, title}` |
| `GET /maevia_views` | гледки — към планината, парк, града (мултиселект) |
| `GET /maevia_distance_kinds` | видове места в „Разстояния до важни места" — метро, спирка, училище… Има и `icon` |
| `GET /maevia_amenities` | удобства в комплекса — охрана, подземен паркинг, фитнес… Има и `icon` |
| `GET /maevia_languages` | езици, които говорят брокерите; `title` е самоназванието („English"), `code` — ISO 639-1 (`en`) за флагче |
| `GET /maevia_companies` | фирми — инвеститори, архитекти, строители: `{id, title, url, logo, sort, active}`; `logo` = `{id, url, url_10x10, …}` или празен обект (`id = 0`) |
| `GET /maevia_energy_classes` | енергиен клас: `1` A+, `2` A, `3` B … `8` G. Фиксиран списък, само `{id, title}` |
| `GET /maevia_finish_levels` | степен на завършеност: `1` тухла, `2` БДС — шпакловка и замазка, `3` до ключ. Фиксиран списък, само `{id, title}` |

Всеки от тях приема и `?id=5` за единичен запис.

**Кеширай ги.** Сменят се рядко; фронтендът не бива да ги тегли на всяко зареждане.

---

## 3. Адреси

Двустепенни: **град/област → квартал/село**. Обявата сочи към квартал.

```
GET /maevia_cities                        → 54 града и области
GET /maevia_neighborhoods?city_id=1372    → кварталите на града (пагинирано)
GET /maevia_neighborhoods?post_code=1000  → кварталите с този пощенски код
GET /maevia_post_codes?neighborhood_id=X  → кодовете на квартал
```

Кварталите са ~5850 общо — **никога не ги теглѝ наведнъж**. Селектът за квартал
се пълни след избор на град (каскада).

`GET /maevia_cities` връща само активните градове, подредени по `sort` (както
ги е наредил админът), не по име. За селекта в търсачката ползвай филтрите —
само градове, в които има какво да се намери:

| Параметър | Връща градовете, в които има… |
|---|---|
| `with_listings=1` | поне една активна обява **или** активен проект |
| `with_estates=1` | поне една активна обява |
| `with_complexes=1` | поне един активен проект — за /proekti |
| `listing_type_id=2` (заедно с `with_estates` / `with_listings`) | само обяви от този вид — напр. градовете с наеми за /naemi |

```
GET /maevia_cities?with_listings=1                    → търсачката на началната страница
GET /maevia_cities?with_estates=1&listing_type_id=2   → /naemi
GET /maevia_cities?with_complexes=1                   → /proekti
```

Без филтър връща всичките 54 записа.

„До мен“: геолокация → пощенски код (през геокодинг услуга на фронта) →
`GET /maevia_estates?near_post_code=1000`.

---

## 4. Обяви — търсачка и детайл

```
GET /maevia_estates            → активните обяви (пагинирано)
GET /maevia_estates/{id}       → детайл (само активна; иначе 404)
GET /maevia_estate_by_number?number=123  → {status, id, slug} за редирект
GET /maevia_estates_map        → леки данни за пиновете на картата
```

`/maevia_estates_map` приема същите филтри като търсачката (вкл. `sw_*`/`ne_*`
и `lat`/`lng`/`radius_km`), но връща само данните за пин — без описания и
мултиселекти. Максимум 1000 точки; при радиус са подредени по разстояние.

### Филтри

Всички се комбинират свободно и се съединяват с **И**. Списъчните филтри
(`*_ids`) са **ИЛИ** вътре в себе си: `type_ids=[1,2]` значи „вид 1 или 2“, но
заедно с `city_id=1` значи „(вид 1 или 2) И в този град“.

Списък се подава или като JSON — `type_ids=[1,2,3]` — или през запетая:
`type_ids=1,2,3`. И двете вървят.

**`only_ids` е изключението** — той приема само истински масив:
`only_ids[]=3&only_ids[]=5`. Подаден като `3,5` взима само първото id, а като
`[3,5]` връща празен резултат. Проверено на живо.

**Статусът е фиксиран.** Публичният endpoint сам налага „активна“; няма как
фронтът да поиска чакащи, свалени или архивирани обяви.

#### Класификация

| Параметър | Действие |
|---|---|
| `listing_type_id` | продажба / наем |
| `type_id` · `type_ids` | вид имот — единичен или списък |
| `condition_id` | състояние |
| `construction_id` | конструкция |
| `construction_stage_id` | етап на строителство |
| `furniture_id` · `furniture_ids` | обзавеждане |
| `overhead_id` | режийни |
| `parking_id` · `parking_ids` | паркиране |
| `deal_status_id` | свободен / резервиран / продаден |
| `broker_id` | обявите на един брокер |
| `only_ids` | точно тези обяви — за списък с любими |

#### Локация

| Параметър | Действие |
|---|---|
| `city_id` | всички обяви в града (през кварталите) |
| `neighborhood_id` · `neighborhood_ids` | квартал — единичен или списък |
| `complex_id` · `building_id` | обявите в комплекс / в конкретна сграда |
| `near_post_code` | обяви в квартали с този пощенски код („До мен“) |
| `with_coordinates` | само с координати — за картата |
| `sw_lat`, `sw_lng`, `ne_lat`, `ne_lng` | видимият правоъгълник на картата; подават се **и четирите** |
| `lat`, `lng`, `radius_km` | търсене в радиус; и трите заедно |

При радиус отговорът носи и `distance_km` за всяка обява, а подредбата
автоматично става „най-близките първо“, освен ако не подадеш свой `order_by`.
Радиусът се реже между 0.1 и **100 км** — по-голяма стойност се свежда до 100.

Координатите в отговора са **публичната точка**: при обява с „приблизителни
координати“ това е размитата точка, не истинската. Филтрите по карта и радиус
също гледат нея, така че двете винаги си съвпадат.

#### Числови диапазони

| Параметър | Действие |
|---|---|
| `price_from`, `price_to` | цена в евро — гледа само `price` |
| `effective_price_from`, `effective_price_to` | **реалната** цена: намалената, ако има, иначе редовната |
| `total_area_sqm_from`, `total_area_sqm_to` | квадратура |
| `year_of_construction_from`, `year_of_construction_to` | година на строеж |
| `floor_from`, `floor_to` | етаж от–до (приема и 0, и отрицателен за сутерен) |
| `rooms_min` | поне толкова стаи |

За ценовия плъзгач на търсачката ползвай `effective_price_*` — иначе обява с
намаление изпада от филтъра заради старата си цена.

#### Точни стойности

Всяка колона на обявата работи и като филтър по име: `rooms=3`, `bedrooms=2`,
`bathrooms=1`, `wc`, `balconies`, `garages`, `floor`, `floor_max`,
`year_of_construction`, `number`, `slug`, `post_code`.

**Внимание за текстовите колони** (`slug`, `street`, `block`, `post_code`):
съвпадението е частично, не точно — `street=Витоша` намира и „Витошка“.

#### Чипове (да/не)

Всеки от тези приема `1`:

`top_offer`, `new_construction`, `lux`, `for_investments`, `exclusive`,
`commissioned`, `no_buyer_renter_commission`, `no_seller_lessor_commission`,
`pets_allowed`, `for_advertising`, `deposited`, `management`,
`with_presentation`, `transparent`, `our_apartment`, `estate_key`,
`personal_offer`, `offer_from_other_agency`, `ideal_parts_flag`, `transitional`,
`weights`, `ceiling`, `warehouse`, `basement`, `technical_passport`,
`encumbrances`.

Плюс два, които не са колони:

| Параметър | Действие |
|---|---|
| `discounted=1` | само с намалена цена |
| `with_garage=1` | поне един гараж |
| `available_until=2026-12-01` | наем „свободен до дата": свободните веднага (`available_from` празно) и тези със `available_from` до подадената дата включително |

#### Мултиселект (през номенклатурните връзки)

`heating_ids`, `exterior_ids`, `exhibition_ids`, `purpose_ids`,
`protection_ids`, `extra_ids`, `equipment_ids`, `tenant_type_ids`, `view_ids` —
всеки приема списък: `extra_ids=[1,3]`.

Сингъл селектите по фиксирани списъци работят като точна стойност:
`energy_class_id=3`, `finish_level_id=3`, `rent_term_id=1`.

#### По дата

| Параметър | Действие |
|---|---|
| `created_after` | добавени след момент: `2026-09-01 00:00:00` |
| `date_created_from`, `date_created_to` | период по дата (границите са включително) |

### Подредба

`order_by` приема име на колона от обявата, `order_direction` е `ASC` или `DESC`.
Име, което не е колона, се игнорира мълчаливо.

```
GET /maevia_estates?order_by=price&order_direction=ASC
```

Без `order_by` подредбата е **топ оферта → най-нови**. При
търсене в радиус — най-близките първо.

### Пагинация

| Параметър | Действие |
|---|---|
| `page` | страница, от 1 |
| `limit` | брой на страница (сървърът има таван и го налага) |
| `no_limit=1` | без ограничение |
| `no_pagination=1` | връща гол списък, без обвивката с `pagination` |

Отговорът е `{ pagination: {...}, result: [...] }`, а `pagination` носи
`total_results`, `total_pages`, `current_page`, `results_per_page`.

### Типични комбинации

```
# търсачка: двустаен или тристаен в София, до 250 000 €, с гараж
GET /maevia_estates?city_id=1372&type_ids=2,3&effective_price_to=250000&with_garage=1

# резултати от картата, докато потребителят я влачи
GET /maevia_estates?sw_lat=42.6&sw_lng=23.2&ne_lat=42.8&ne_lng=23.5&with_coordinates=1

# „в радиус 5 км от мен“, най-близките първо
GET /maevia_estates?lat=42.6790&lng=23.3200&radius_km=5

# страница 2 от подредени по цена възходящо
GET /maevia_estates?order_by=price&order_direction=ASC&page=2&limit=12

# страницата „Любими“ — id-тата ги пази фронтът
GET /maevia_estates?only_ids[]=3&only_ids[]=5

# „нови през последната седмица“
GET /maevia_estates?created_after=2026-09-17 00:00:00

# офертите с намаление, за отделна секция на началната
GET /maevia_estates?discounted=1&limit=8
```

### Какво връща обявата

Числово и текстово: `id, number, slug, title, description, description_short,
seo_title, seo_description, image_alt, price, price_discounted, price_no_vat, price_per_sqm,
total_area_sqm, net_area_sqm, ideal_parts_sqm, rooms, bedrooms, bathrooms, wc,
bathroom_and_wc, balconies, garages, floor, floor_max, year_of_construction`.

`image_alt` е **един** alt текст за всички снимки на обявата — обявата няма alt на
отделна снимка. Слагай го в `alt` на всеки `<img>` от галерията. Може да е празен;
тогава остави `alt=""`, а не измислен текст.

Връзки по id: `listing_type_id, type_id, status_id, deal_status_id, condition_id,
construction_id, construction_stage_id, furniture_id, overhead_id, parking_id,
neighborhood_id, broker_id, user_id, complex_id, building_id` — разгъват се срещу номенклатурите, които вече си кеширал.

Адрес: `neighborhood_id, post_code, street, street_number, block, entrance,
apartment_number, latitude_approx, longitude_approx, approximate_coordinates`.

Промо: `top_offer` (0/1 — таг „Топ оферта" и първи в подредбата), `priority`.

> **Breaking (17.10.2026):** ВИП отпада изцяло — полето `vip_until` вече не
> съществува. Промотирането е само чрез `top_offer`.

**Изчислени полета** (идват и в списъка, и в детайла — не ги смятай във фронта):

| Поле | Тип | Какво е |
|---|---|---|
| `kind` | string | вид обява: `rent` при `listing_type_id = 2`; иначе `unit` при `complex_id > 0` (имот от проект); иначе `sale` |
| `price_effective` | float | реалната цена: `price_discounted`, ако е > 0, иначе `price` |
| `discount_percent` | int | `round((1 − price_discounted / price) × 100)`; `0` = няма намаление |
| `price_per_sqm` | float | вече се смята от сървъра при запис: `price_effective / total_area_sqm`, 2 знака. Само при обява без площ или без цена остава ръчно въведената стойност |

```json
{ "id": 4, "kind": "sale", "price": 129500, "price_discounted": 122000,
  "price_effective": 122000, "discount_percent": 6, "price_per_sqm": 1967.74 }
```

25 булеви полета (0/1): `new_construction, lux, for_investments, exclusive,
no_buyer_renter_commission, no_seller_lessor_commission, pets_allowed, transitional,
weights, encumbrances, ceiling, warehouse, basement, technical_passport, estate_key,
ideal_parts_flag, deposited, commissioned, management, with_presentation, transparent,
our_apartment, personal_offer, offer_from_other_agency, for_advertising`.

**Условия за наем** (идват при всяка обява; смислени са при `kind = "rent"`):

| Поле | Тип | Какво е |
|---|---|---|
| `deposit_months` | int | депозит в брой наеми; `0` = не е посочен. Първоначално плащане = `price × (1 + deposit_months)` |
| `min_term_months` | int | минимален срок на договора в месеци; `0` = не е посочен |
| `rent_term_id` | int | вид наем (`GET /maevia_rent_terms`); `0` = не е посочен |
| `rent_term` | object\|null | същото, разгънато: `{"id": 1, "title": "Дългосрочен"}`; `null` при `rent_term_id = 0` |
| `available_from` | date\|null | `"2026-12-01"`; `null` или минала дата = свободен веднага → таг „Свободен", иначе „Свободен от 01.12" |
| `smoking_allowed` | 0/1 | пушенето е разрешено |
| `utilities_estimate` | int | ориентировъчни режийни, € на месец; `0` = не е посочено |
| `notarized_contract` | 0/1 | договор с нотариална заверка |

Комисионата за наемателя не е публична и не идва в отговора.

**Сградата, броячи и таг „Ново"** (при всяка обява):

| Поле | Тип | Какво е |
|---|---|---|
| `energy_class_id` | int | енергиен клас (`GET /maevia_energy_classes`); `0` = не е посочен |
| `energy_class` | object\|null | същото, разгънато: `{"id": 3, "title": "B"}` |
| `finish_level_id` | int | степен на завършеност (`GET /maevia_finish_levels`); `0` = не е посочена |
| `finish_level` | object\|null | `{"id": 3, "title": "До ключ"}` |
| `views_count` | int | отваряния на публичния детайл. Всяко `GET /maevia_estates/{id}` го вдига с 1 (и отговорът вече носи новата стойност); списъците не броят. Под `views_threshold` от `GET /maevia_settings` не показвай нито него, нито `inquiries_30d` |
| `is_new` | 0/1 | `1`, ако обявата е добавена преди по-малко от `new_days` дни (настройка, по подразбиране 14) — за тага „Ново" и в списъците. Сървърът го смята сам |

Само в детайла (`/{id}`): `complex` и `building` (обектите, ако обявата
е в комплекс), `photos[]`, `ai_images[]`, `architectures[]`, `price_list`,
`heating_ids[]`, `exterior_ids[]`, `exhibition_ids[]`, `purpose_ids[]`,
`protection_ids[]`, `extra_ids[]`, `equipment_ids[]`, `tenant_type_ids[]`,
`view_ids[]`, `highlights[]`, `distances[]`, `inquiries_30d`, `documents[]`.

`documents[]` — публичните документи на обявата (произволен брой, PDF, Word,
снимки…): `{id, object_id, filename, url, path, date_created}`. Показвай
`filename` като текст на линка към `url`; празен масив = няма документи.

```json
"documents": [
  { "id": 3, "object_id": 6, "filename": "skica.pdf",
    "url": "https://r2.expozy.com/maevia/maevia/estates_documents/6/skica.pdf" }
]
```

**Имот от проект** (`complex_id > 0`): вложеният `complex` носи основните полета
на проекта плюс `developer`, `architect`, `builder`, `milestones[]`,
`payment_schemes[]` и `brochure` (същият формат като в детайла на проекта, §5) —
за блоковете „Схеми на плащане", „Етапи", инвеститор/архитект/строител и
брошурата, без втора заявка към проекта.

| Поле | Тип | Какво е |
|---|---|---|
| `highlights[]` | array | акценти под описанието, в реда за показване: `{id, estate_id, sort, title}` |
| `distances[]` | array | разстояния до важни места: `{id, estate_id, kind_id, kind, label, meters, sort}`; `kind` = `{id, title, icon, …}` от `/maevia_distance_kinds`; `label` е надпис по желание („Метростанция Опълченска") — ако е празен, показвай `kind.title` |
| `inquiries_30d` | int | запитвания („Искам инфо" + „Искам оглед") за последните 30 дни. Броят се от 16.10.2026 нататък — по-старите запитвания не са вързани за обява |

```json
{
  "energy_class": {"id": 3, "title": "B"},
  "finish_level": {"id": 3, "title": "До ключ"},
  "views_count": 4, "is_new": 0, "inquiries_30d": 0,
  "view_ids": [3],
  "highlights": [
    {"id": 7, "estate_id": 6, "sort": 1, "title": "Офис сграда клас А"}
  ],
  "distances": [
    {"id": 3, "estate_id": 6, "kind_id": 1, "label": "Метростанция Опълченска", "meters": 450, "sort": 1,
     "kind": {"id": 1, "title": "Метро", "icon": "", "sort": 1, "active": 1}}
  ]
}
```

`ai_images[]` са генерираните с „AI интериор" визуализации (виж §11) — стоят
ОТДЕЛНО от `photos[]`, за да може фронтът да ги отдели, ако потрябва. Носят
надпис „AI-generated" върху самото изображение; показвай ги като илюстрация,
не като снимка на реалното състояние.

Снимка: `{id, object_id, filename, url, url_10x10}`; има и готови размери
`360x240/` и `640x480/` в същата папка на `url`.

**Вътрешните полета не се връщат публично** (цена на собственика, комисионни,
бележки, данни за ключа, точни координати). Публично са само приблизителните
координати — на картата се показват те.

### История на цената

```
GET /maevia_price_histories?estate_id=4
```

Само `saas_key`, без токен. Връща масив от най-старата към най-новата цена:

```json
[
  { "price": 129500, "date_created": "2026-09-04 14:44:33" }
]
```

| Поле | Тип | Какво е |
|---|---|---|
| `price` | float | реалната цена към момента на промяната (намалената, ако има, иначе редовната), в евро |
| `date_created` | datetime | кога е записана |

Запис се прави при създаване на обявата и при всяка смяна на реалната цена —
т.е. и при ново намаление. Същите записи пускат и имейлите „цената е
променена" до потребителите с обявата в любими.

Грешки: без `estate_id` → `{"status": 0, "errors": {"[name=\"estate_id\"]": "Моля, въведете"}}`;
неактивна или несъществуваща обява → HTTP 404 `{"status": 0}`.

---

## 5. Комплекси

Комплексът е самостоятелен обект със собствена страница и галерия.
Съдържа **сгради**, а всяка сграда — **входове** със собствен брой етажи.

```
GET /maevia_complexes                     → видимите комплекси (пагинирано)
GET /maevia_complexes?city_id=1372        → комплексите в град (през кварталите)
GET /maevia_complexes?with_coordinates=1  → за картата
GET /maevia_complexes?amenity_ids=1,4     → с тези удобства (мултиселект; също heating_ids)
GET /maevia_complexes/{id}                → детайл със сгради, входове и снимки
GET /maevia_estates?complex_id={id}       → обявите в този комплекс
GET /maevia_building_classes              → клас на сградата: 1 стандарт, 2 висок стандарт, 3 лукс (фиксиран списък)
```

Неактивен комплекс (`active = 0`) връща **404** на детайла и не излиза в списъка.

Полета: `id, slug, title, description, description_short, seo_title,
seo_description, neighborhood_id, street, street_number, block, post_code,
latitude, longitude, broker_id, construction_id, construction_stage_id,
year_of_construction, website, youtube_link, virtual_tour_link, sort`, плюс:

| Поле | Тип | Какво е |
|---|---|---|
| `no_buyer_commission` | 0/1 | таг „Без комисиона" |
| `exclusive` | 0/1 | таг „Ексклузивно" |
| `gross_area_sqm` | int | РЗП, м²; `0` = не е посочено |
| `elevators_per_entrance` | int | асансьори на вход |
| `parking_underground`, `parking_outdoor` | int | брой паркоместа подземни / на открито |
| `building_class_id` + `building_class` | int + object\|null | `{"id": 2, "title": "Висок стандарт"}` |
| `energy_class_id` + `energy_class` | int + object\|null | енергиен клас (`/maevia_energy_classes`) |
| `finish_level_id` + `finish_level` | int + object\|null | степен на завършеност (`/maevia_finish_levels`) |

**Агрегати от активните имоти в проекта** — идват и в списъка, и в детайла,
така че картите на /proekti не искат отделна заявка за всеки проект:

| Поле | Тип | Какво е |
|---|---|---|
| `price_min` | float | най-ниската реална цена (намалената, ако има) сред имотите с цена > 0 |
| `price_sqm_min`, `price_sqm_max` | float | цена на м² от–до (реалната цена / `total_area_sqm`) |
| `area_min`, `area_max` | float | площ от–до, м² |
| `units_total` | int | активни имоти в проекта |
| `units_free`, `units_reserved`, `units_sold` | int | по статус на сделката (1 свободен, 2 резервиран, 3 продаден) — за лентата |

Без имоти всички са `0`.

**Наследяване в имота.** Имот от проекта без свой енергиен клас или степен на
завършеност показва тези на проекта: в детайла на имота `energy_class` /
`finish_level` идват от проекта, а `energy_class_id` / `finish_level_id`
остават `0` (така се вижда, че стойността е наследена).

Само в детайла:

- `photos[]` — галерията (същият формат като при обявите)
- `estates_count` — брой активни обяви в комплекса
- `buildings[]` — `{id, complex_id, title, entrances_count, sort, entrances[]}`,
  а всеки вход е `{id, building_id, title, floors_count, sort}`
- `heating_ids[]`, `amenity_ids[]` — отопление и удобства (`/maevia_heatings`, `/maevia_amenities`)
- `highlights[]` — „Ключови предимства": `{id, complex_id, sort, title}`
- `distances[]` — `{id, complex_id, kind_id, kind, label, meters, sort}`, като при имота
- `brochure`, `price_list` — `{id, object_id, filename, url, path, date_created}`;
  без файл `id = 0` и `url = ""`
- `architectures[]` — архитектурни планове, същият формат
- `progress_photos[]` — снимки от строежа, най-новата първа:
  `{id, object_id, filename, url, url_10x10, taken_on, caption, sort_order, date_created}`
- `progress_updated` — датата на най-новата снимка (`"2026-10-10"`) за
  „Последна актуализация"; `null`, ако няма снимки
- `contact_office` — само при проект **без брокер** (`broker_id = 0`): офисът,
  отбелязан като „Отдел ново строителство" (формат като `GET /maevia_offices`).
  `null`, ако проектът има брокер или няма отбелязан офис

- `developer_id`, `architect_id`, `builder_id` + разгънати `developer`,
  `architect`, `builder` — фирмата по роля от `/maevia_companies`
  (`{id, title, url, logo}`) или `null`. Една фирма може да е в няколко роли
- `milestones[]` — етапите с дати, в реда на номенклатурата:
  `{id, complex_id, construction_stage_id, stage {id, title}, date, done}`.
  `date` е `"2025-05-01"` или `null`; `done` = 0/1. Текущият етап на проекта
  (`construction_stage_id`) е последният с `done = 1`. Показвай месец и година
- `payment_schemes[]` — схемите на плащане за целия проект:

```json
[
  { "id": 1, "title": "С банков кредит", "description": "Депозит и кредит след Акт 14", "sort": 1,
    "steps": [
      { "label": "депозит", "percent": 0, "amount": 3000, "construction_stage_id": 0, "stage": null, "sort": 1 },
      { "label": "при предварителен договор", "percent": 30, "amount": 0, "construction_stage_id": 0, "stage": null, "sort": 2 },
      { "label": "кредит след Акт 14", "percent": 70, "amount": 0, "construction_stage_id": 4,
        "stage": { "id": 4, "title": "След акт 14" }, "sort": 3 }
    ] }
]
```

Стъпката е **процент ИЛИ сума**: `percent > 0` → процент от цената, иначе
`amount` в евро. Процентите в една схема дават точно 100 — рисувай ги като
пропорционална лента; сумите (депозит) не влизат в лентата. `label` може да е
празен — тогава ползвай `stage.title`.

`entrances_count` е декларираният брой входове по проект — може да е
по-голям от броя записи в `entrances[]`, ако не всички са описани.

Комплекс от вили няма сгради — тогава обявата има `complex_id`,
но `building_id = 0`. Входът, етажът и апартаментът на конкретния имот
са полета на самата обява (`entrance`, `floor`, `apartment_number`).

---

## 6. Статуси

`status_id`: **1** чака одобрение, **2** активна, **3** неактивна, **4** архивирана.
Публичните endpoint-и връщат само активни. Потребителската обява влиза с 1 и
става 2 след одобрение от админ (собственикът получава имейл).

---

## 7. Потребителски функции (искат Bearer токен)

### Как се познава брокер

В обекта на потребителя (`login`, `GET /accounts`) **няма** поле `is_broker`.
Ролята е в `userlevel`: брокерът е **точно `14`**.

```js
const isBroker = data.user.userlevel == 14;
```

Не ползвай праг (`>= 14`) — нивата не са йерархия: 99 и 100 са админи, 13 е
съвсем друга роля.

`userlevel == 14` казва само, че човекът **има брокерска роля** — не че е
одобрен. Одобрението стои в записа му като брокер и в обекта на потребителя го
няма. Затова действията, които искат одобрен брокер (AI интериор, телефоните в
„Търся имот“), могат да върнат **403** и на потребител с ниво 14 — фронтът
трябва да го покаже като „профилът ви още не е одобрен“, а не като грешка.

### Кой може да подава обява

`POST /maevia_estates` е отворен за **всеки логнат потребител** — това е
нарочно, частни продавачи също подават обяви:

| Кой | Какво става с обявата |
|---|---|
| обикновен потребител | влиза като „чака одобрение“ и не се вижда, докато админ не я одобри |
| одобрен брокер | става активна веднага |
| брокер, който още не е одобрен или е спрян | отказ: `Access denied` |

Тоест сървърът вече пази същественото — обикновеният потребител не може да
публикува нищо без админ.

```
GET  /my_maevia_estates            → моите обяви (всички статуси, пагинирано)
POST /maevia_estates               → нова обява (влиза "чака одобрение")
POST /maevia_estates/{id}          → редакция на СВОЯ обява
DELETE /maevia_estates/{id}        → архивиране на своя обява
POST /maevia_estates_restore/{id}  → връщане на своя архивирана обява

GET  /my_maevia_favorites          → любими (пагинирано)
POST /maevia_favorites             → toggle: {estate_id} → {status, favorite: 0|1}

POST /maevia_price_offers          → {estate_id, price} ценово предложение (upsert + имейл до админ)

GET  /my_maevia_search_logs        → запазени търсения
POST /maevia_search_logs           → {name, filters (JSON), request_uri}
DELETE /maevia_search_logs/{id}    → изтриване на свое
```

**Връщане от архив** (`POST /maevia_estates_restore/{id}`, без тяло): само за собственика
и само ако обявата е архивирана (`status_id` 4). Одобрен брокер я получава веднага
активна (2), всички останали — „чака одобрение“ (1), т.е. минава отново през модерация.
Грешки: `Access denied` (не е твоя или не съществува), `Not archived` (не е в архива).
Успехът връща `{status: 1, obj}` с новия `status_id`. Редакцията през
`POST /maevia_estates/{id}` НЕ променя статуса — архивирана обява си остава архивирана.

**Подаване на обява:** приемат се само полетата от белия списък — вид имот и сделка,
квартал, площи, стаи, етаж, година, цена, адресни полета, YouTube/виртуална разходка,
заглавие, описание, мултиселектите и условията за наем (`deposit_months`,
`min_term_months`, `rent_term_id`, `available_from` във формат `YYYY-MM-DD` или празно,
`smoking_allowed`, `utilities_estimate`, `notarized_contract`, `equipment_ids`,
`tenant_type_ids`), както и `energy_class_id`, `finish_level_id`, `view_ids`,
`highlights[]` (списък текстове; празните се пропускат, до 10) и `distances`
(`distances[0][kind_id]`, `distances[0][label]`, `distances[0][meters]` …; ред без
вид или без метри се пропуска). `highlights` / `distances` заменят изцяло
досегашните; празен низ ги изчиства, а липсващ ключ ги оставя непроменени.
`views_count` не може да се пише отвън. Всичко останало (статус, брокер, топ оферта,
вътрешни цени) се игнорира, дори да го пратиш.

Грешки по условията за наем: непознат `rent_term_id` или невалидна дата в
`available_from` → `{"status": 0, "[name=\"rent_term_id\"]": "..."}` (съответно
`available_from`); отрицателни `deposit_months` / `min_term_months` /
`utilities_estimate` → същата грешка по полето. Непознат `energy_class_id` или
`finish_level_id` → `{"status": 0, "[name=\"energy_class_id\"]": "..."}` (съответно).

Задължителни при създаване: `type_id`, `listing_type_id`, `neighborhood_id`.
Ако видът имот има конфигурация `fields`, тя добавя свои задължителни полета
и скрива неприложимите — виж §8.

**Запазени търсения:** `filters` е JSON със същите параметри като търсачката;
дневен крон известява потребителя, ако са излезли нови обяви по тях.

---

## 8. „Търся имот" — заявки на купувачи (обратен пазар)

Купувачът описва какво търси; дневният крон намира съвпадащите активни
обяви и известява брокера на всяка от тях (всяка двойка — само веднъж).

```
GET    /maevia_wanted_statuses     → номенклатурата (1 Активна, 2 Намерен имот,
                                     3 Изтекла, 4 Затворена)

За купувача (Bearer токен):
GET    /my_maevia_wanted           → моите заявки (пагинирано)
GET    /my_maevia_wanted?id=3      → една моя заявка с мултиселектите
GET    /my_maevia_wanted_matches?id=3  → съвпадащите обяви сега (без вътрешните полета)
POST   /maevia_wanted              → нова заявка
POST   /maevia_wanted?id=3         → редакция на СВОЯ заявка
DELETE /maevia_wanted/{id}         → затваря заявката (статус 4, не се трие)

Само за ОДОБРЕН брокер (иначе 403):
GET    /maevia_wanted              → активните заявки с телефоните за връзка
GET    /maevia_wanted_matches?id=3 → съвпадащите обяви по заявката
```

**Полета на заявката** (всички освен `phone` и `listing_type_id` са по желание):
`phone` (задължителен, мин. 6 знака — брокерът звъни на него), `listing_type_id`
(задължителен), `city_id`, `price_from`, `price_to`, `area_from`, `area_to`,
`rooms_min`, `floor_from`, `floor_to`, `new_construction` (0/1), `garage` (0/1),
`note`, `status_id` и мултиселектите `type_ids`, `neighborhood_ids`,
`furniture_ids`, `parking_ids` (масив или JSON стринг).

**Как се съпоставя:** празен критерий не стеснява търсенето; попълненият
трябва да е изпълнен точно, без толеранс. Избраните квартали изместват града,
а цената на обявата се взима като намалената, когато има такава.

**Срок:** новата заявка е активна 90 дни (`date_expires`); кронът я прехвърля в
„Изтекла". Връщането в статус „Активна" мести срока напред със същите 90 дни.

Заявките НЕ са публични — телефоните се виждат само от одобрени брокери.

---

## 8а. Заявки за оглед

„Искам оглед" в детайла на обявата (`POST /maevia_inquiry` с `kind=viewing`, виж §9)
освен имейла записва и **заявка за оглед**. Тя се закача за брокера, който държи
обявата в момента на заявката. Ако после обявата смени брокера, заявката остава
при стария. Брокерът вижда своите заявки, сменя им статуса, датата и пише
забележка. Клиентът, ако е бил логнат при подаването, вижда своите заявки и
статуса им.

```
GET  /maevia_viewing_statuses     → номенклатурата (1 Активна, 2 Неактивна, 3 Отминала)

Само за ОДОБРЕН брокер (Bearer токен; иначе 403):
GET  /maevia_viewings             → заявките по моите обяви (пагинирано, най-новите първо)
GET  /maevia_viewings?id=7        → една моя заявка (чужда/несъществуваща → 404)
POST /maevia_viewings/7           → {status_id?, viewing_date?, broker_note?} обновява моя заявка

За клиента (Bearer токен):
GET  /my_maevia_viewings          → моите заявки (пагинирано, най-новите първо)
GET  /my_maevia_viewings?id=7     → една моя заявка (чужда → 404)
```

**Статуси:** новата заявка е винаги **1 Активна**. **2 Неактивна** я слага брокерът
(отказ, не се е състоял, дублиран). **3 Отминала** я слага дневният крон: всяка
Активна с попълнена `viewing_date`, която вече е минала. Заявка без дата никога не
става Отминала сама. Ако брокерът премести датата напред, статусът не се връща
сам. Трябва изрично да прати и `status_id=1`.

**Обект „заявка за оглед":**

| Поле | Тип | Какво е |
|---|---|---|
| `id` | int | |
| `estate_id` | int | обявата |
| `estate` | object\|null | обявата накратко: `{id, number, slug, title, image}`; `image` = URL на първата снимка или `null`; `null`, ако обявата е изтрита |
| `broker_id` | int | брокерът на обявата при подаването; `0` = обявата е без брокер |
| `user_id` | int | клиентът, ако е бил логнат; `0` = подадена без логин |
| `status_id` | int | 1 / 2 / 3, виж по-горе |
| `status` | object | `{id, title, sort, active}` |
| `names`, `phone`, `email`, `message` | string | данните от формата |
| `viewing_date` | string\|null | `"YYYY-MM-DD HH:MM:SS"`: желаната от клиента или уговорената от брокера дата; `null` = няма |
| `broker_note` | string | забележка на брокера, до 1000 знака. **Само в `/maevia_viewings`**: клиентът не я получава |
| `date_created`, `date_updated` | string | |

Пример (`GET /maevia_viewings?id=7`):

```json
{
  "id": 7, "estate_id": 9, "broker_id": 3, "user_id": 9, "status_id": 1,
  "names": "Иван Петров", "phone": "0888123456", "email": "ivan@example.com",
  "message": "Удобно ми е следобед", "viewing_date": "2026-10-28 15:00:00",
  "broker_note": "Ключът е у портиера",
  "date_created": "2026-10-23 11:02:15", "date_updated": null,
  "status": {"id": 1, "title": "Активна", "sort": 1, "active": 1},
  "estate": {"id": 9, "number": 9, "slug": "dvustaen-lozenets", "title": "Двустаен в Лозенец", "image": "https://…/1.jpg"}
}
```

**Филтри в списъците:** `status_id`, `estate_id`, `viewing_date_from` /
`viewing_date_to` (`YYYY-MM-DD`, по датата на огледа), плюс стандартните `page`,
`limit`, `order_by` / `order_direction`.

**`POST /maevia_viewings/{id}`** приема само тези три полета, всичко друго се пренебрегва.
Подава се само това, което се сменя:

| Поле | Какво |
|---|---|
| `status_id` | 1 / 2 / 3; несъществуващ → грешка по полето |
| `viewing_date` | `YYYY-MM-DD HH:MM` (може и `YYYY-MM-DD` или с секунди); празно = маха датата. Брокерът може да впише и минала дата (състоял се оглед) |
| `broker_note` | свободен текст, режe се до 1000 знака |

Отговор: `{status: 1, obj: <заявката>}`. Грешки: `Access denied` (не е твоя
или не си одобрен брокер), `[name="status_id"]` / `[name="viewing_date"]`
(невалидна стойност).

---

## 9. Лийд форми (публични, без токен)

```
POST /maevia_inquiry           {estate_id, kind: info|viewing, names, phone, email?, message?, viewing_date?, follow?, similar?}
POST /maevia_consultation      {names, phone, email?, message?}
POST /maevia_job_application   {city: sofia|varna|burgas, names, phone, email?, message?, cv (файл)}
```

**Кой получава запитването (`maevia_inquiry`, и инфо, и оглед):** имейлът отива
при брокера на обявата, ако е одобрен и има имейл. Иначе (обява без брокер,
спрян брокер) отива при кол центъра.

**„Искам оглед" (`kind=viewing`)** приема и `viewing_date`: желаната дата и час,
по желание. Формат е `YYYY-MM-DD HH:MM` (или само `YYYY-MM-DD`) и не може да е
в миналото. Невалидна дата връща `{status: 0, "[name=\"viewing_date\"]": …}` и
тогава нищо не се записва и не се праща. При успех заявката се записва
(виж §8а) и отговорът носи `viewing_id`. Ако заявката е подадена с Bearer
токен, клиентът после я вижда в `GET /my_maevia_viewings`. Затова за огледа
е добре формата да праща токена винаги, когато потребителят е логнат.

**„Следи имота" и „Подобни оферти" — само за регистрирани.** Чекбоксовете във
формата за запитване се показват само на логнат потребител; заявката тогава
носи и `Authorization: Bearer <token>`:

| Поле | Какво прави |
|---|---|
| `follow=1` | добавя имота в любимите (ако вече е там — нищо). Дневният крон праща имейл при промяна на цената, включително намаление |
| `similar=1` | създава запазено търсене „Подобни на „…"" със същия вид имот и вид обява, същия квартал, цена и площ ±15 %, поне толкова стаи. Кронът праща имейл за новите обяви по него. Вижда се и се трие от `GET /my_maevia_search_logs` |

Отговорът тогава има и `follow: 1` / `similar_search_id: <id>`. Без токен
двата флага се пренебрегват — запитването минава нормално.

Свързаните бутони на сайта, също само за логнат:

- **„Следете цената"** в детайла = `POST /maevia_favorites {estate_id}` (вече съществува)
- **„Известия за проекта"** = `POST /maevia_search_logs` с
  `{"name": "Проект …", "filters": "{\"complex_id\": 5}"}` — кронът известява за
  новите имоти в проекта

Всяка форма записва брояч в статистиките и праща имейл. CV-то се праща като
прикачен файл (multipart) и не се съхранява.

Други публични:

```
GET /maevia_brokers         → брокери (видимите, по приоритет)
GET /maevia_brokers?id=5    → един брокер
GET /maevia_offices         → активните офиси (по sort)
GET /maevia_offices?id=2    → един офис; неактивен/несъществуващ → 404
GET /maevia_offices?is_new_construction=1  → „Отдел ново строителство"
```

**Брокер:** `{id, name, job_title, email, phone, image, office_id, office,
language_ids, languages}`.

| Поле | Тип | Какво е |
|---|---|---|
| `office_id` | int | офисът на брокера; `0` = без офис |
| `office` | object\|null | офисът, разгънат (формат като по-долу) — за секция „Консултант" |
| `language_ids` | int[] | езиците, които говори (`GET /maevia_languages`) |
| `languages` | array | същите, разгънати: `[{"id": 1, "title": "Български", "code": "bg", …}, {"id": 2, "title": "English", "code": "en", …}]` — за реда „Български · English" |

**Офис:**

| Поле | Тип | Какво е |
|---|---|---|
| `id`, `title` | int, string | заглавие („Офис София") |
| `address` | string | адрес |
| `latitude`, `longitude` | float | координати за карта; `0` = не са въведени |
| `working_hours` | string | работното време, както се изписва („Пон – Нед · 09:00 – 19:00 …") |
| `phone`, `email` | string | контакти — резервните, когато обявата няма брокер |
| `is_new_construction` | 0/1 | „Отдел ново строителство" — най-много един офис е `1` |
| `sort`, `active` | int | подредба; публично излизат само активните |

Обявата сочи към брокера с `broker_id`; офисът и езиците идват от брокера.
Проект без брокер носи контактите на отдела в `contact_office` (виж §5).

---

## 9а. „Бърза оценка" (само за логнати)

Клиентът описва имота си, а админът отговаря с ценови диапазон „от–до".

```
GET  /maevia_valuation_statuses     → 1 Нова, 2 В процес, 3 Отговорена, 4 Отказана
GET  /my_maevia_valuations          → моите заявки
GET  /my_maevia_valuations?id=5     → една моя заявка с отговора и снимките
POST /maevia_valuations             → нова заявка (+ файлове `photos[]`, до 10)
POST /maevia_valuations?id=5        → редакция на своя заявка, докато няма отговор
```

**Полета:** `names, phone, email, neighborhood_id, street, street_number, block,
type_id, condition_id, construction_id, total_area_sqm, rooms, floor, floor_max,
year_of_construction, note`.

**Отговор (попълва се от админа):** `status_id, price_from, price_to, answer,
date_answered`. В обекта идват и хидратирани `status`, `type`, `neighborhood`,
`photos[]`.

След отговор заявката не може да се редактира. Админът получава имейл при нова
заявка, клиентът — при статус „Отговорена".

---

## 10. Условни полета по вид имот

`GET /maevia_types` връща и колона `fields` — JSON конфигурация:

```json
{"hidden": ["floor", "bedrooms"], "required": ["total_area_sqm", "price"]}
```

Формата за подаване на обява трябва да я спазва: скритите полета не се показват,
изискваните са задължителни. Празно = показвай всичко. Сървърът валидира същото,
така че разминаване води до отказ при запис.

---

## 11. „AI интериор" — обзавеждане на празна стая (само за одобрен брокер)

Брокерът качва снимка на празна стая към своя обява и външен доставчик връща
три варианта с интериорен дизайн. Всеки генериран файл носи **изгорен надпис
„AI-generated"** — етикетът е част от изображението, за да оцелее при сваляне и
при експорт към портали.

```
GET    /maevia_interior_styles                 → стилове (Скандинавски, Модерен…)
GET    /maevia_room_types                      → видове стаи (Хол, Спалня…)
GET    /maevia_estates_ai_images?estate_id=5   → снимките на своята обява
POST   /maevia_estates_ai_images               → {estate_id} + файл `image` (изходната)
POST   /maevia_estates_ai_generate             → {id, style_id, room_type_id}
DELETE /maevia_estates_ai_images/{id}          → триене
```

**Достъп:** всички искат Bearer токен на **одобрен брокер** и работят само върху
негова обява (същата проверка като при редакция). Обикновен потребител получава 403.
Двата списъка връщат `prompt` празен — текстът към доставчика е вътрешна настройка.

**Как се ползва:** първо се качва изходната снимка (`POST /maevia_estates_ai_images`),
после с върнатото `obj.id` се вика генерирането. Двете са отделни, за да може едно
и също изходно изображение да се пуска пак — с друг стил или след грешка.

**Структура на отговора.** Всички снимки са в една таблица и се различават по
`original_id`:

```
original_id = 0    → изходната снимка (празната стая); тя носи статуса на
                     последното генериране: 1 чака, 2 генерира, 3 готова, 4 грешка,
                     плюс `error` с текста от доставчика
original_id = <id> → генериран вариант; носи `style_id`, `room_type_id`,
                     `provider` и `model`, с които е направен
```

`GET /maevia_estates_ai_images` връща изходните снимки, всяка с вариантите си в
`generated` и с хидратирани `style` и `room_type`.

**В детайла на обявата** генерираните варианти идват в отделен масив `ai_images`
(виж §4) — НЕ са смесени с `photos`. Изходната снимка не излиза на сайта.

**Бавно е.** Генерирането трае десетки секунди и заявката е синхронна — покажи
изчакване и заключи бутона, иначе брокерът ще натисне три пъти и ще плати три пъти.

---

## 11а. Настройки за фронта

```
GET /maevia_settings
```

Само `saas_key`. Връща настройките, които админът сменя от „Настройки на
maevia" и които фронтът ползва. Кеширай ги за кратко (минути) — сменят се рядко.

```json
{ "mortgage_rate": 2.9, "mortgage_down_payment": 20, "mortgage_years": 25,
  "new_days": 14, "views_threshold": 20 }
```

| Поле | Тип | Какво е |
|---|---|---|
| `mortgage_rate` | float | лихва по подразбиране в ипотечния калкулатор, % годишно |
| `mortgage_down_payment` | float | самоучастие по подразбиране, % от цената |
| `mortgage_years` | int | срок на кредита по подразбиране, години |
| `new_days` | int | колко дни обявата е „Нова" — само за информация, `is_new` вече идва готово |
| `views_threshold` | int | под толкова `views_count` не показвай „N прегледа" и „N запитвания за 30 дни" |

Калкулаторът е анюитет: кредит = цена × (1 − самоучастие / 100), месечна лихва =
лихва / 12 / 100, вноска = кредит × r / (1 − (1 + r)^(−месеци)).

---

## 12. Правила за фронтенда

- **saas_key никога не стига до браузъра.** Заявките минават през сървърен прокси
  с бял списък на позволените endpoint-и; без него всеки може да удря админ API-то.
- **Ескейпвай всичко от API-то** преди вкарване в HTML — данните са въведени от
  админи и потребители.
- Кеширай номенклатурите; не кеширай списъка с обяви за дълго (цени и статуси се менят).
- Не теглѝ всички квартали/обяви наведнъж — винаги с филтър и пагинация.
- Картата: една заявка с `with_coordinates=1` и същите филтри като списъка;
  ползвай `latitude_approx`/`longitude_approx`.
- Детайлът на неактивна обява връща 404 — покажи нормална 404 страница, не празен изглед.
- Числата са в евро; `price_discounted > 0` означава намаление спрямо `price`.
