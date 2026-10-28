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

**Кеш — във фронтенда.** Фронтендът вика endpoint-а през своя кеш, **по език**, с
TTL около 1 час. Промяна в админа се вижда на сайта след изтичане на кеша —
приемливо, без механизъм за инвалидиране. Как точно (решено при обсъждането):

- Зарежда се **при първа нужда** от `JsonLd::organization()` (т.3), не в
  `FrontCore` — иначе се вика и за заявки без HTML (llms.txt, sitemap, AJAX).
- Успешният отговор: `organization` се добавя в `Cache::ENDPOINTS`
  (`core/classes/class.cache.php`) и минава през `Api::cache(true)` — 1 час.
- Неуспешният: кешът на ядрото не пази грешки (само статус 200 с JSON), затова
  `JsonLd` пише маркер `cache/organization_fail_{lang}.php` (със защитния ред
  `<?php exit; ?>`) за 5 мин. — без него стар бекенд би получавал заявка на всяка
  страница. `Cache` не се променя за останалите endpoint-и.
- При `devMode = 1` кешът на ядрото е изключен — заявка на всяка страница;
  маркерът при грешка важи и тогава.

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

## 3. Нов клас `JsonLd`, промени в `GeoOptimizer` и `Page`

Разпределение (решено при обсъждането):

- **`core/classes/class.json_ld.php` → `JsonLd`** — целият граф: зарежда
  организацията (т.2), сглобява `Organization`, `ImageObject`, `WebSite`,
  `WebPage`, `BreadcrumbList` и възлите по тип страница (3.2), извежда **един**
  `<script type="application/ld+json">` и след него ръчния скрипт (т.6).
  Методите за възлите връщат масиви; само `render()` извежда.
- **`GeoOptimizer`** остава за продуктите: `normalizeProduct()`, `plainText()`,
  `generateProductFAQ()`, гео мета таговете, LLMO блокът. `JsonLd` ги ползва за
  `Product` и `FAQPage`.
- **`GeoOptimizer::headTags()`** остава входната точка (викат го `header.php` и
  `ai_version/layout.php` — те не се пипат, и в другите проекти) и само подава към
  `JsonLd`. На продуктова страница извежда графа **и** гео мета таговете от
  `generateGeoMetaTags()` — те не са JSON-LD и остават отделни `<meta>`, както
  сега. Скритият LLMO блок в `<body>` (`generateLLMOBlock()`) не се пипа.
- **`Page::canonical()`** (`core/classes/class.page.php`) — canonical е на
  страницата, не на JSON-LD (виж 3.1).
- Ръчно писаният JSON-LD извън `JsonLd` се маха — напр. `NewsArticle` в
  `$this->script` на `Page::load_page()` (в ecommerce вече го няма).

Без JSON-LD (`render()` не извежда нищо): `$page->error404`, страница `404`,
статус ≠ 200 (503 при поддръжка). Проверката е в `JsonLd`, защото `header.php`
вика `headTags()` винаги, а `layout.php` — само при 200; иначе двете версии се
разминават на 404.

### 3.1. Възли на всяка страница (статус 200)

| Възел | `@id` | Полета | Източник |
|---|---|---|---|
| `Organization` (или `OnlineStore` / `LocalBusiness` по `type`) | `{SITEURL}/#organization` | виж таблицата в 3.3 | `JsonLd::organization()` + лого |
| `ImageObject` (логото) | `{SITEURL}/#logo` | `url`, `caption` = `name`; `width`/`height`, ако са известни | `$core->web['logo']`; **само ако има лого** |
| `WebSite` | `{SITEURL}/#website` | `url` = `SITEURL/`, `name`, `alternateName` (първото от `alternate_names`, ако има), `description`, `inLanguage`, `publisher` → `#organization` | `organization`, `$lang` |
| `WebPage` | `{canonical}#webpage` | `url` = canonical, `name` = `$page->seo_title`, `description` = `$page->seo_description`, `inLanguage`, `isPartOf` → `#website`, `about` → `#organization` (само на началната), `primaryImageOfPage` → `#logo` или снимката на обекта, `breadcrumb` → `#breadcrumb` (ако има) | `$page`, `Page::canonical()` |

`WebPage` е на **всяка** страница със статус 200 — и на служебните (кошница,
вход). Нормалната версия не знае кои страници са noindex (списъкът е само в
`ai_version/config.php`), а `WebPage` там е безвреден; така графът е еднакъв в
двете версии.

Canonical: един и същ URL в нормалната и в AI версията, смятан на едно място —
`Page::canonical()`: `SITEURL` + пътя + параметрите, които сменят съдържанието, +
`?page=N` (N > 1). Параметрите са в константа на `Page` (по подразбиране само
`page`; проект добавя своите, напр. maevia `id` за `/estate?id=5`). Ползват го
`pages/header.php` (вместо днешния `REDIRECT_URL`, който губи `?page=` и `?id=`),
`JsonLd` и `AiRender::canonical()` (той го вика, ако `method_exists`, иначе
досегашната си логика — за проекти без новия `Page`). `cache_params` в
`ai_version/config.php` остава само за ключа на AI кеша.

### 3.2. Възли по тип страница

| Страница (`$page->type` / `slug`) | Допълнителни възли | Бележки |
|---|---|---|
| начална (`homepage`) | няма; `WebPage.about` → `#organization` | |
| `products` (категория) | `CollectionPage` **вместо** `WebPage` (същия `@id` и полета, `@type` сменен); `BreadcrumbList` (`#breadcrumb`): Начало → родители → категорията. **Без `ItemList`** (решено: `products` не е в `Cache::ENDPOINTS` — в нормалната версия всяко отваряне на категория би правило още една заявка без кеш) | категорията от `$page->get_target('categories')`; родителите — от дървото `get.categories` (в кеша на ядрото) |
| `product` | `Product` (`@id` = `{url}#product`): `name`, `description`, `sku`, `image[]`, `brand` → `Brand{name}` (марката на продукта, **не** организацията), `offers` → `Offer{price, priceCurrency, availability, url, seller → #organization}`; `FAQPage` (както сега); `BreadcrumbList`: Начало → веригата категории → продуктът; `WebPage.mainEntity` → `#product`. Гео мета таговете — отделно, извън графа (т.3) | `normalizeProduct()` остава; продуктът от `$page->get_target('products')`; веригата категории — от дървото `get.categories` (продуктът носи само своята категория) |
| `post` | `Article` (`@id` = `{url}#article`): както сега, но `publisher` → `#organization` и `author` → `#organization` (освен ако статията има автор-човек); `BreadcrumbList`; `WebPage.mainEntity` → `#article` | |
| `blog` | `CollectionPage` + `BreadcrumbList` | |
| страница от редактора (`page.php`) | само `WebPage` | |
| служебни страници (кошница, вход…) | само `WebPage` | виж 3.1 |
| 404 / 503 | **нищо** | проверката е в `JsonLd` (т.3) |

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
fallback към измислени стойности. Празна организация (или липсващ endpoint) дава
валиден възел с `name` (= `$core->site_name`), `url` и лого — точно колкото
излиза и днес.

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
  `JsonLd::organization()` връща празен масив и поведението е от днес: `name`,
  `url`, `logo`. Нито warning, нито празни възли. Неуспешният отговор се помни
  5 мин. (маркерът от т.2), за да не чака час след обновяване на бекенда.
- Към 2026-10-28 devcore още отговаря 400 на `GET /api/organization` — фронтендът
  се прави и тества първо в този режим, а с реални данни — когато бекендът е готов.
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
   полета са допустими (как — раздел 8 → „ТЕСТ“).
   Състояние 2026-10-28 (ecommerce.myexpozy.com): schema.org — 0 грешки / 0
   предупреждения (начална, категория, продукт, статия); Rich Results — Articles,
   Product snippets и Breadcrumbs валидни; Merchant listings — „Missing field image“,
   защото тестовите продукти нямат снимки → да се повтори на продукт със снимка.
3. Празна „Организация“ (или бекенд без `GET /api/organization`) → страницата се
   рендерира, възелът е минимален, без PHP notice.
4. Частични данни: само град без държава → без `address`; само `contact_url` без
   телефон/имейл → без `contactPoint`.
5. `extra` с ключ `name` → името от `organization.name` остава (типизираното
   печели); `extra` с `@type` → игнорира се.
6. Ръчен скрипт непразен → извежда се след графа, непроменен.
7. Бекенд с грешка: след първата заявка се появява `cache/organization_fail_{lang}.php`
   и следващите страници в рамките на 5 мин. не викат endpoint-а.
8. 404 (несъществуващ продукт/статия) → никакъв `application/ld+json` и в двете версии.
9. Продукт → графът плюс гео мета таговете, както преди; LLMO блокът в `<body>` е на мястото си.
10. Canonical: `/bg/blog?page=2` → с `?page=2` и в двете версии; сравни `<link rel="canonical">`
    с `WebPage.url`.
11. Ред в края на задачата по раздел 24: `AI версия: засегната — layout.php вика
    същия headTags(); canonical от Page::canonical(); проверено с curl като бот`.

---

## 8. Конвенция (трайна → раздел 25 на `frontend_framework.md`)

### 25. JSON-LD (структурирани данни) — ЕДИН @graph, ЕДИН ИЗТОЧНИК

Всяка страница със статус 200 извежда един `<script type="application/ld+json">`
с `@graph`, сглобен от `core/classes/class.json_ld.php` (`JsonLd`). Входната точка е
`GeoOptimizer::headTags()` — викат го `pages/header.php` и `ai_version/layout.php`,
съдържанието е **еднакво** за хора и за AI ботове. На 404/503 — нищо.
`GeoOptimizer` остава за продуктите (FAQ, гео мета тагове, LLMO блок).
Не пиши JSON-LD на ръка в `.html` на страница, в `static/pages/*`, в шаблон на
`ai_version/`, в `scripts.header` или в `$this->script` на `Page::load_page()`.
Вече написан на ръка JSON-LD се маха, когато `JsonLd` дойде в проекта.

Проект без `class.json_ld.php` не извежда графа, докато класът не бъде пренесен
(първо ecommerce, после проект по проект).

── ОТКЪДЕ ИДВАТ ДАННИТЕ ──

  Organization / WebSite / WebPage   → `JsonLd::organization()` от `GET /api/organization`
                                        (кеширан във фронтенда по език, ~1 час;
                                        грешка — маркер за 5 мин.)
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
- Canonical — само от `Page::canonical()` (път + параметрите, които сменят
  съдържанието, + ?page=N), един и същ в нормалната и в AI версията и в
  `WebPage.url`. Нов параметър със собствено съдържание (?id=, ?vehicle=) →
  в константата на `Page`, не само в `ai_version/config.php`.
- `WebPage` е на всяка страница със статус 200, и на служебните.
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
       обява за имот → RealEstateListing с offers → Offer (наем: цена на месец)
       жилищен комплекс → ApartmentComplex / Residence
       събитие → Event
       услуга → Service с provider → #organization
     При съмнение — проверка в https://schema.org/docs/full.html и Google
     Search Central → Structured data → поддържаните типове.
  2) В `JsonLd` — нов клон по `$page->slug`, който взима обекта с
     `$page->get_target('<endpoint>', $id)` (id от пътя или от ?id=) и връща възел
     с `@id` = `{url}#<тип>`; `WebPage.mainEntity` сочи към него; `BreadcrumbList`
     ако страницата има родител. Без нова заявка за обекта.
  3) Полетата: само това, което потребителят вижда на страницата (парност с
     Alpine и с AI шаблона). Цени → `Offer` с `priceCurrency` от `$currency`.
  4) Списъчна страница на типа (всички хотели) → `CollectionPage`. `ItemList` само
     ако списъкът вече е зареден на сървъра или endpoint-ът е в `Cache::ENDPOINTS` —
     не добавяй некеширана заявка на всяко отваряне на страницата.
  5) Тест — задължителен, по „ТЕСТ“ по-долу.

── ТЕСТ (ЗАДЪЛЖИТЕЛЕН СЛЕД ВСЯКА ПРОМЯНА В JSON-LD) ──

След генериране на JSON-LD за нова страница (или промяна в `JsonLd` /
`GeoOptimizer`) — всяка засегната страница през трите проверки:

  1) curl като човек и като бот (`-A "ExpozyAIPreview"`) → точно един
     `application/ld+json` и еднакъв граф в двете версии (`diff`); на 404 — нито един.

  2) Schema.org Validator — синтаксис и речник на schema.org:
       https://validator.schema.org/#url=<URL-encoded адрес на страницата>
     Резултат: 0 ГРЕШКИ и 0 ПРЕДУПРЕЖДЕНИЯ. Възлите, вързани с @id, се показват
     вложени (Product в WebPage.mainEntity, Organization в seller/author) — това е
     правилно, не липса.
     Смяна само на #url= в същия раздел НЕ пуска нов тест — зареди друга страница
     между два теста или ползвай „Стартиране на нов тест“.

  3) Google Rich Results Test — дали Google може да покаже обогатен резултат:
       https://search.google.com/test/rich-results?url=<URL-encoded адрес>
     Тестът отнема около минута. Резултат: всеки открит тип (Product snippets,
     Merchant listings, Articles, Breadcrumbs…) без „critical issue“. Забележки
     „(optional)“ са допустими, ако данните ги няма в API-то (напр. shippingDetails,
     hasMerchantReturnPolicy за магазин) — запиши ги в отчета.
     Критична грешка заради липсващи данни в теста (напр. продукт без снимка →
     Missing field "image") не е грешка в кода — повтори теста на обект с пълни
     данни. НЕ попълвай липсващо поле с измислена стойност (лого вместо снимка и т.н.).

  Тествай обект с реални, пълни данни (снимка, цена, наличност, дата), не празен
  тестов запис. Страницата трябва да е публично достъпна (без парола / Basic Auth) —
  иначе и двата валидатора получават 401.

── ЗАДЪЛЖИТЕЛЕН РЕД В КРАЯ НА ЗАДАЧА С НОВ ТИП СТРАНИЦА ──

  JSON-LD: {тип на възела и @id} — validator.schema.org: {грешки/предупреждения},
           Rich Results: {типове и статус}
  или
  JSON-LD: не е засегнат — {защо}
