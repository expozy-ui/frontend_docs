# Задача (фронтенд): JSON-LD `@graph` от данните „Организация“

> **Два вида съдържание.** Раздели 1–7 са **еднократна задача** в официалния
> фронтенд (`ecommerce`): когато е готова и тествана, те се изтриват. Раздел 8
> е **трайна конвенция** за всеки следващ проект и нов тип страница — при
> приключване се премества като **раздел 25 в `frontend_docs/frontend_framework.md`**,
> след което този файл се изтрива. Нищо от него не се пази на две места.

Дата: 2026-10-28. За агента във фронтенда (`ecommerce`, официалният Alpine
фронтенд). Двойникът за бекенда е `core-organization-backend-spec.md`; общата
част е **т.2 — договорът на `GET /api/organization`**. Промяна там се прави и в
двата документа. Бекендът може още да не е готов: фронтендът трябва да работи и
без този endpoint (т.5).

---

## 1. Цел

`core/classes/class.geo_optimizer.php` вече извежда JSON-LD (`Product`, `FAQPage`,
`Article`, минимален `Organization`, `CollectionPage`) като отделни `<script>`
елементи от `headTags()`, който се вика от `pages/header.php` и от
`ai_version/layout.php`. Задачата: един `@graph` с `@id` връзки, в който
`Organization`, `WebSite` и `WebPage` са на всяка страница и се пълнят от
данните, които администраторът въвежда в devcore, а страничните възли
(`Product`, `Article`…) сочат към тях вместо да повтарят името на фирмата.

Резултат на началната страница:

```json
{"@context":"https://schema.org","@graph":[
 {"@type":"OnlineStore","@id":"https://site.bg/#organization","name":"Bids & Cars",
  "alternateName":["Bids&Cars","Бидс енд Карс"],"legalName":"БИДС ЕНД КАРС ЕООД","vatID":"BG206123456",
  "url":"https://site.bg/","logo":{"@type":"ImageObject","@id":"https://site.bg/#logo","url":"https://r2…/logo.webp"},
  "image":{"@id":"https://site.bg/#logo"},"description":"…",
  "email":"support@site.bg","telephone":"+359877629529",
  "address":{"@type":"PostalAddress","streetAddress":"ул. …","addressLocality":"Кюстендил","addressCountry":"BG"},
  "areaServed":{"@type":"Country","name":"BG"},
  "founder":{"@type":"Person","name":"…"},"foundingDate":"2023",
  "contactPoint":{"@type":"ContactPoint","contactType":"customer support","telephone":"+359877629529","email":"support@site.bg","url":"https://site.bg/bg/contacts","areaServed":"BG","availableLanguage":["bg","en"]},
  "sameAs":["https://www.facebook.com/x","https://www.instagram.com/x"],
  "slogan":"…"},
 {"@type":"WebSite","@id":"https://site.bg/#website","url":"https://site.bg/","name":"Bids & Cars","description":"…","inLanguage":"bg","publisher":{"@id":"https://site.bg/#organization"}},
 {"@type":"WebPage","@id":"https://site.bg/bg/#webpage","url":"https://site.bg/bg/","name":"Bids & Cars — онлайн търгове…","description":"…","inLanguage":"bg","isPartOf":{"@id":"https://site.bg/#website"},"about":{"@id":"https://site.bg/#organization"},"primaryImageOfPage":{"@id":"https://site.bg/#logo"}}
]}
```

## 2. Вход: договорът `GET /api/organization`

Бекендът сглобява данните от „Данни за фактура“, „Контакти“, „Социални мрежи“ и
новата админ страница „Организация“ и ги връща в **отделен** endpoint
`GET /api/organization?lang=<код>` (само `saas_key`, без токен). Те **не** идват в
`GET /api/settings`: сглобяването е около 15–20 заявки към базата, а `settings`
се вика при всяка заявка.

**Кеш — във фронтенда.** Фронтендът вика endpoint-а през своя кеш (`Api::cache(true)`
или еквивалент), **по език**, с TTL около 1 час, и слага резултата в
`$core->organization` (масив) при зареждане на страницата. Промяна в админа се
вижда на сайта след изтичане на кеша — приемливо, без механизъм за инвалидиране.
Отговорът е самият обект (не обвит в `organization`):

```json
{
  "type": "OnlineStore",
  "name": "Bids & Cars",
  "legal_name": "БИДС ЕНД КАРС ЕООД",
  "vat_id": "BG206123456",
  "tax_id": "206123456",
  "email": "support@bidsandcars.bg",
  "phone": "+359877629529",
  "street": "ул. Иларион Ловчански 16",
  "city": "Кюстендил",
  "postal_code": "",
  "country": "BG",
  "contact_url": "/bg/contacts",
  "same_as": ["https://www.facebook.com/bidsandcars", "https://www.instagram.com/bidsandcars"],
  "founder": "Кристиан Брусарски",
  "founding_date": "2023",
  "description": "Bids & Cars е българска платформа за …",
  "alternate_names": ["Bids&Cars", "Бидс енд Карс", "bidsandcars.bg"],
  "extra": {"slogan": "Търгове за коли без посредник"}
}
```

Правила: всички ключове са винаги налице; празно е `""`, `[]` или `{}`, никога
`null`; низовете са без HTML. Освен това фронтендът ползва, както и досега:
`$core->site_name`, `SITEURL`, `$core->web['logo']`, `$lang->language`,
`Api::cache(true)->get()->languages()` (кодовете на езиците) и
`$core->web['scripts']['organization']` (ръчният скрипт, т.6).

## 3. Какво се променя в `GeoOptimizer`

Структурата остава (един клас, `headTags()` от две места). Променя се начинът на
извеждане: методите връщат **възли** (масиви), `headTags()` ги събира в един
`@graph` и извежда **един** `<script type="application/ld+json">`.

### 3.1. Възли на всяка страница (статус 200)

| Възел | `@id` | Полета | Източник |
|---|---|---|---|
| `Organization` (или `OnlineStore` / `LocalBusiness` по `type`) | `{SITEURL}/#organization` | виж таблицата в 3.3 | `$core->organization` + лого |
| `ImageObject` (логото) | `{SITEURL}/#logo` | `url`, `caption` = `name`; `width`/`height`, ако са известни | `$core->web['logo']`; **само ако има лого** |
| `WebSite` | `{SITEURL}/#website` | `url` = `SITEURL/`, `name`, `alternateName` (първото от `alternate_names`, ако има), `description`, `inLanguage`, `publisher` → `#organization` | `organization`, `$lang` |
| `WebPage` | `{canonical}#webpage` | `url` = canonical, `name` = `$page->seo_title`, `description` = `$page->seo_description`, `inLanguage`, `isPartOf` → `#website`, `about` → `#organization` (само на началната), `primaryImageOfPage` → `#logo` или снимката на обекта, `breadcrumb` → `#breadcrumb` (ако има) | `$page`, canonical както го смята `header.php` / `AiRender::canonical()` |

Canonical: един и същ URL в нормалната и в AI версията (виж `AiRender::canonical()`
— път + `?page=N`, без други параметри). `headTags()` трябва да го получи по един
и същ начин и в двете; днес `header.php` ползва `REDIRECT_URL` — уеднакви.

### 3.2. Възли по тип страница

| Страница (`$page->type` / `slug`) | Допълнителни възли | Бележки |
|---|---|---|
| начална (`homepage`) | няма; `WebPage.about` → `#organization` | |
| `products` (категория) | `CollectionPage` **вместо** `WebPage` (същия `@id` и полета, `@type` сменен); `BreadcrumbList` (`#breadcrumb`): Начало → родители → категорията; `ItemList` с `itemListElement` = `ListItem{position, url}` на продуктите от страницата (без цени — те са на продуктовата страница) | данните са вече заредени в `products.php` / `$page->get_target('categories')` |
| `product` | `Product` (`@id` = `{url}#product`): `name`, `description`, `sku`, `image[]`, `brand` → `Brand{name}` (марката на продукта, **не** организацията), `offers` → `Offer{price, priceCurrency, availability, url, seller → #organization}`; `FAQPage` (както сега); `BreadcrumbList`; `WebPage.mainEntity` → `#product` | `normalizeProduct()` остава; продуктът от `$page->get_target('products')` |
| `post` | `Article` (`@id` = `{url}#article`): както сега, но `publisher` → `#organization` и `author` → `#organization` (освен ако статията има автор-човек); `BreadcrumbList`; `WebPage.mainEntity` → `#article` | |
| `blog` | `CollectionPage` + `BreadcrumbList` | |
| страница от редактора (`page.php`) | само `WebPage` | |
| `noindex` страници (кошница, вход…) и 404/503 | **нищо** (`headTags()` не се вика) | както сега в `layout.php` |

`BreadcrumbList`: `@id` = `{canonical}#breadcrumb`, `itemListElement` =
`ListItem{position, name, item}`; последният елемент без `item`.

### 3.3. `Organization` — съответствие и правила за извеждане

| Ключ в `organization` | schema.org | Правило |
|---|---|---|
| `type` | `@type` | `Organization` / `OnlineStore` / `LocalBusiness`; непознато → `Organization` |
| `name` | `name` | винаги |
| `alternate_names` | `alternateName` | масив; само ако непразен |
| `legal_name` | `legalName` | ако непразно |
| `vat_id` | `vatID` | ако непразно |
| `tax_id` | `taxID` | ако непразно и различно от `vat_id` без буквите |
| `description` | `description` | ако непразно; същото и в `WebSite.description` |
| `email` | `email` | ако непразно |
| `phone` | `telephone` | ако непразно |
| `street`, `city`, `postal_code`, `country` | `address` → `PostalAddress` | **само в цялост:** `country` непразно и поне едно от `street` / `city`; иначе без `address` |
| `country` | `areaServed` → `Country{name}` | ако непразно |
| `founder` | `founder` → `Person{name}` | ако непразно |
| `founding_date` | `foundingDate` | ако непразно |
| `email`, `phone`, `contact_url`, `country`, езици | `contactPoint` → `ContactPoint{contactType:"customer support", telephone, email, url, areaServed, availableLanguage}` | **само ако има телефон или имейл**; `url` = `SITEURL + contact_url` (ако е път) или както е (ако е пълен); `availableLanguage` = кодовете на езиците на сайта |
| `same_as` | `sameAs` | масив; само ако непразен |
| `extra` | свойства на `Organization` | слива се **след** всичко по-горе; типизираното печели при същия ключ; `@context`, `@id`, `@type`, `@graph` от `extra` се игнорират |
| лого | `logo` → `{@id: #logo}`, `image` → `{@id: #logo}` | само ако има лого |
| — | `url` | винаги `SITEURL/` |

Общо правило: **пропусни, не измисляй.** Празно поле не се извежда; няма
fallback към измислени стойности. Празен `organization` дава валиден възел с
`name`, `url` и лого — точно колкото излиза и днес.

## 4. Къде се извежда

- `pages/header.php` (нормалната версия) и `ai_version/layout.php` (AI версията)
  викат същия `headTags()` — **не се променя**. Съдържанието е еднакво и за
  хора, и за ботове (парност, раздел 24 на `frontend_framework.md`).
- Един `<script type="application/ld+json">`, `JSON_UNESCAPED_UNICODE |
  JSON_UNESCAPED_SLASHES`, без `JSON_PRETTY_PRINT` на продукция (размер);
  `</script>` вътре в низ се escape-ва (`JSON_HEX_TAG`).
- SPA навигацията (`page.js → updateMeta()`) **не** обновява JSON-LD. Това е
  приемливо: ботовете и Rich Results четат сървърния HTML. Не добавяй клиентско
  генериране.
- Кешът на AI версията (`ai_version/cache/`, до 15 мин.) и кешът на
  `GET /api/organization` (около 1 час) не се чистят при промяна в „Организация“.
  Приемливо: данните се сменят рядко.

## 5. Съвместимост

- `GET /api/organization` липсва или връща грешка (стар бекенд) →
  `$core->organization` остава празен и поведението е от днес: `name`, `url`,
  `logo`. Нито warning, нито празни възли. Неуспешният отговор не се кешира за
  дълго (напр. 5 мин.), за да не чака час след обновяване на бекенда.
- Ключ липсва в `organization` (по-стар договор) → третира се като празен.
- `normalizeProduct()`, `plainText()`, `generateProductFAQ()` остават; шаблонът
  `ai_version/templates/product.php` ползва `generateProductFAQ()` директно и не
  се пипа.

## 6. Ръчният скрипт „Schema organization“

В админа (Настройки → Скриптове) има поле „Schema organization“, което идва във
фронтенда като `$core->web['scripts']['organization']`. **Днес фронтендът не го
извежда никъде.** Правило оттук нататък: ако е непразно, се извежда **както е**,
веднага след генерирания `@graph`, без обработка. Администраторът е предупреден в
„Организация“, че това дублира данни, и е насърчен да ги прехвърли в полетата.
Не парсвай скрипта и не се опитвай да го сливаш с генерирания възел.

## 7. Тест

1. `curl` на начална, категория, продукт, статия, страница от редактора, кошница
   — нормален User-Agent и `-A "…GPTBot…"`: точно един `application/ld+json`
   (плюс ръчният, ако има) и **еднакъв** в двете версии (сравни с `diff`).
2. Валидация: https://validator.schema.org/ и https://search.google.com/test/rich-results
   на продукт и статия — без грешки; предупрежденията за липсващи незадължителни
   полета са допустими.
3. Празна „Организация“ (или бекенд без `GET /api/organization`) → страницата се
   рендерира, възелът е минимален, без PHP notice.
4. Частични данни: само град без държава → без `address`; само `contact_url` без
   телефон/имейл → без `contactPoint`.
5. `extra` с ключ `name` → името от `organization.name` остава (типизираното
   печели); `extra` с `@type` → игнорира се.
6. Ръчен скрипт непразен → извежда се след графа, непроменен.
7. Ред в края на задачата по раздел 24: `AI версия: засегната — layout.php вика
   същия headTags(); проверено с curl като бот`.

---

## 8. Конвенция (трайна → раздел 25 на `frontend_framework.md`)

### 25. JSON-LD (структурирани данни) — ЕДИН @graph, ЕДИН ИЗТОЧНИК

Всяка страница извежда един `<script type="application/ld+json">` с `@graph`,
генериран от `core/classes/class.geo_optimizer.php → headTags()`. Викат го
`pages/header.php` и `ai_version/layout.php` — съдържанието е **еднакво** за
хора и за AI ботове. Не пиши JSON-LD на ръка в `.html` на страница, в
`static/pages/*`, в шаблон на `ai_version/` или в `scripts.header`.

── ОТКЪДЕ ИДВАТ ДАННИТЕ ──

  Organization / WebSite / WebPage   → `$core->organization` от `GET /api/organization`
                                        (кеширан във фронтенда по език, ~1 час)
                                        (администраторът: devcore → Settings →
                                        „Организация“, „Данни за фактура“, „Контакти“,
                                        „Социални мрежи“). Фронтендът НЕ ги знае и
                                        НЕ ги измисля. Празно поле не се извежда.
  Product / Article / Collection…    → обектът, който `Page::load_page()` вече е
                                        заредил (`$page->get_target()`), без втора
                                        заявка.
  Лого, URL, езици                   → `$core->web['logo']`, `SITEURL`, `languages()`.

── @id КОНВЕНЦИЯ ──

  {SITEURL}/#organization   {SITEURL}/#website   {SITEURL}/#logo
  {canonical}#webpage       {canonical}#breadcrumb
  {url}#product             {url}#article        {url}#<тип>  за нови обекти

  Страничните възли сочат към организацията с {"@id": ".../#organization"}
  (publisher, seller, provider, about) вместо да повтарят името ѝ.

── ПРАВИЛА ──

- Пропусни, не измисляй: няма fallback към измислени стойности; адресът и
  точката за контакт се извеждат само в цялост.
- Canonical е един и същ в нормалната и в AI версията (път + ?page=N).
- `organization.extra` се слива в Organization след типизираните полета.
- Ръчният „Schema organization“ скрипт от админа се извежда след графа както е.
- JSON-LD се генерира само на сървъра; SPA навигацията не го обновява и това е
  нарочно.

── НОВ ТИП СТРАНИЦА (хотел, обява, аукцион, дилър…) ──

JSON-LD е ЧЕТВЪРТОТО огледало на динамичната страница, до трите от раздели
23 и 24 (page.js ↔ class.page.php ↔ ai_version шаблон). При нов slug с ID в URL-а:

  1) Избери schema.org тип за обекта и за страницата:
       хотел → LodgingBusiness / Hotel + WebPage
       обява за кола → Car (Vehicle) с offers → Offer + WebPage
       аукцион → Product с offers → Offer (price = текуща/начална цена,
                 availability, validThrough = край на търга) + WebPage
       дилър/партньор → Organization (отделен @id {url}#dealer, НЕ #organization)
       събитие → Event
       услуга → Service с provider → #organization
     При съмнение — проверка в https://schema.org/docs/full.html и Google
     Search Central → Structured data → поддържаните типове.
  2) В `GeoOptimizer::headTags()` — нов клон по `$page->slug`, който взима
     обекта с `$page->get_target('<endpoint>', $page->target_id)` и връща възел
     с `@id` = `{url}#<тип>`; `WebPage.mainEntity` сочи към него; `BreadcrumbList`
     ако страницата има родител.
  3) Полетата: само това, което потребителят вижда на страницата (парност с
     Alpine и с AI шаблона). Цени → `Offer` с `priceCurrency` от `$currency`.
  4) Списъчна страница на типа (всички хотели) → `CollectionPage` + `ItemList`.
  5) Тест: curl като човек и като бот → един и същ граф; validator.schema.org
     без грешки.

── ЗАДЪЛЖИТЕЛЕН РЕД В КРАЯ НА ЗАДАЧА С НОВ ТИП СТРАНИЦА ──

  JSON-LD: {тип на възела и @id} / не е засегнат — {защо}
