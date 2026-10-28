# Задача (бекенд): „Организация“ — данни за JSON-LD в devcore и админа

> **Еднократна задача.** Документът описва какво да се направи, за да заработи
> модулът. След като всичко по т.8 (тест) и т.9 (документация) е готово,
> **този файл се изтрива** от `frontend_docs`. Трайното описание на данните
> остава в `frontend_docs/core-v4-front.md` (т.9).

Дата: 2026-10-28. За агента в `devcore` + `admin.expozy.DEV`. Следва
`devcore/.claude/skills/devcore-conventions/SKILL.md` и модела на „AI видимост“
(`lib/classes/aiVersion/class.aiVersion.php`, `GET|POST /admin/ai_version`,
`admin/v4/settings.ai_version.php` + `templates/expozy/settings.ai_version.html`).
Двойникът на този документ за фронтенда е `core-organization-frontend-spec.md`;
общата им част е **т.6 — договорът на `GET /api/organization`**.
Промяна там се прави и в двата документа.

---

## 1. Цел

Фронтендът (`core/classes/class.geo_optimizer.php`) генерира JSON-LD на всяка
страница. За `Organization`, `WebSite` и `WebPage` му трябват фирмени данни, които
не могат да се извлекат от кода. Бекендът ги събира от местата, където вече ги
има, добавя няколкото липсващи полета, и ги връща **сглобени** в един обект от **отделен** endpoint
`GET /api/organization`. Не в `GET /api/settings`: сглобяването е около 15–20
заявки към базата, а `settings` се вика при всяка заявка на фронтенда. Фронтендът
кешира отговора сам (по език, ~1 час).

Пример за крайния резултат на фронтенда (само за ориентация, генерира го той):

```json
{"@context":"https://schema.org","@graph":[
 {"@type":"OnlineStore","@id":"https://site.bg/#organization","name":"Bids & Cars",
  "alternateName":["Bids&Cars","Бидс енд Карс"],"legalName":"БИДС ЕНД КАРС ЕООД","vatID":"BG206123456",
  "url":"https://site.bg/","logo":{"@type":"ImageObject","@id":"https://site.bg/#logo","url":"…"},
  "description":"…","email":"support@site.bg","telephone":"+359877629529",
  "address":{"@type":"PostalAddress","streetAddress":"…","addressLocality":"…","addressCountry":"BG"},
  "founder":{"@type":"Person","name":"…"},"foundingDate":"2023",
  "contactPoint":{"@type":"ContactPoint","contactType":"customer support","telephone":"…","email":"…","url":"https://site.bg/bg/contacts","areaServed":"BG","availableLanguage":["bg","en"]},
  "sameAs":["https://www.facebook.com/…","https://www.instagram.com/…"]},
 {"@type":"WebSite","@id":"https://site.bg/#website","url":"https://site.bg/","name":"…","inLanguage":"bg","publisher":{"@id":"https://site.bg/#organization"}},
 {"@type":"WebPage","@id":"https://site.bg/bg/#webpage","url":"…","name":"…","inLanguage":"bg","isPartOf":{"@id":"https://site.bg/#website"},"about":{"@id":"https://site.bg/#organization"}}
]}
```

## 2. Какво вече съществува — НЕ се дублира

Проверено в devcore на 2026-10-28:

| Данни | Къде са | Клас / ключ | Публичен endpoint | Админ |
|---|---|---|---|---|
| Име на фирмата (преводимо), ЕИК, ДДС номер, МОЛ, телефон, град (преводим), адрес (преводим), държава | таблица `company`, ред 1 | `ProjectCompany` (`lib/classes/class.project.company.php`): `name`, `eik`, `vat_number`, `mol`, `phone`, `city`, `address`, `country_id` → `Country` | `GET /api/company` | Settings → „Данни за фактура“ (`settings.invoice.html`) |
| Имейл, телефон, адрес (свободен текст), ЕИК, име за контакти | `modules_settings` | `AdminFunctions::MKEY_CONTACT_EMAIL` (`cmail`), `MKEY_CONTACT_PHONE` (`cphone`), `MKEY_CONTACT_ADDRESS` (`caddress`), `MKEY_COMPANY_EIK` (`ceik`), `MKEY_COMPANY_NAME` (`cname`) | `GET /api/contacts` (`AdminFunctions::get_contacts()`) | Settings (контакти) |
| Социални мрежи: Facebook, Twitter, YouTube, Instagram | `modules_settings` | `MKEY_LINK_FACEBOOK` … | `GET /api/settings_web` → `links{facebook,twitter,youtube,instagram}` | Settings → социални мрежи |
| „Schema organization“ — ръчен скрипт | `modules_settings` | `MKEY_SCRIPT_ORGANIZATION` (`script_organization`) | `GET /api/settings_web` → `scripts.organization` | Settings → скриптове („Добавете персонализирана Schema organization…“) |
| Име на сайта, URL, език по подразбиране | `settings`, ред 1 | `$core->site_name`, `site_url`, `lang` | `GET /api/settings` | общи настройки |
| Лого, favicon | `modules_settings` | `MKEY_LOGO`, `MKEY_FAVICON` | `GET /api/settings_web` → `logo` | общи настройки |

Извод: юридическо име, ЕИК, ДДС, телефон, имейл, адрес, социални профили и лого
**вече се въвеждат** от администратора. Не се правят втори полета за тях.

## 3. Какво липсва — новите полета

Домът им е таблицата **`company`** (ред 1) през `ProjectCompany`, защото това са
данни за фирмата и класът вече има Content-managed преводими полета (`name`,
`city`, `address`) — същият механизъм ползваме за описанието.

Колони в `company`:

| Колона | Тип | По подразбиране | Съдържание | schema.org |
|---|---|---|---|---|
| `org_type` | VARCHAR(32) | `'Organization'` | `Organization`, `OnlineStore`, `LocalBusiness` | `@type` |
| `org_founder` | VARCHAR(255) | `''` | име на основателя | `founder` → `Person.name` |
| `org_founding_date` | VARCHAR(10) | `''` | `YYYY` или `YYYY-MM-DD` | `foundingDate` |
| `org_contact_url` | VARCHAR(255) | `''` | път (`/bg/contacts`) или пълен URL | `contactPoint.url` |
| `org_same_as_extra` | TEXT | NULL | допълнителни профили, по един пълен URL на ред (LinkedIn, TikTok, Google Business, Wikipedia…) | добавят се към `sameAs[]` след четирите от `links` |
| `org_extra_jsonld` | TEXT | NULL | JSON обект с допълнителни свойства за `Organization` (виж по-долу) | слива се в `Organization` възела |

Преводими, през Content (`lang_content`, `table = 'company'`, `record_id = 1`),
точно както `name` / `city` / `address` в `ProjectCompany::loadObject()`:

| Ключ | Съдържание | schema.org |
|---|---|---|
| `org_description` | 1–3 изречения какво е организацията (до 1000 символа) | `Organization.description`, `WebSite.description` |
| `org_alternate_names` | алтернативни имена, разделени със запетая (латиница, кирилица, домейнът) | `alternateName[]` |

**`org_extra_jsonld` — разширение без миграция.** schema.org има над 60 свойства
за `Organization`; типизирана колона заслужават само тези, които всеки проект
ползва. Всичко друго (слоган, брой служители, ценови диапазон, награди, родителска
организация, втора точка за контакт, `knowsLanguage`…) се въвежда тук като JSON
обект и фронтендът го слива в `Organization` **след** типизираните полета
(типизираното печели при същия ключ). Пример:

```json
{"slogan":"Търгове за коли без посредник","numberOfEmployees":{"@type":"QuantitativeValue","value":12},"priceRange":"€€"}
```

Валидира се само структурно (т.5), не семантично. Това поле е наследникът на
ръчния „Schema organization“ скрипт (виж т.7, „Ръчният скрипт“).

## 4. Миграция

Файл `projects/<name>/migrations/2026_10_28_company_organization.sql` във
**всеки проект** (базата е на проекта, ядрото е общо), идемпотентна:

```sql
-- =========================================================
-- Migration: company — „Организация“ (данни за JSON-LD)
-- Date:      2026-10-28
-- Class:     ProjectCompany / Organization
-- Описание:  Полета за Organization схемата на фронтенда, които не съществуват
--            другаде. Преводимите (org_description, org_alternate_names) са в lang_content.
-- =========================================================
ALTER TABLE `company`
  ADD COLUMN IF NOT EXISTS `org_type`          VARCHAR(32)  NOT NULL DEFAULT 'Organization',
  ADD COLUMN IF NOT EXISTS `org_founder`       VARCHAR(255) NOT NULL DEFAULT '',
  ADD COLUMN IF NOT EXISTS `org_founding_date` VARCHAR(10)  NOT NULL DEFAULT '',
  ADD COLUMN IF NOT EXISTS `org_contact_url`   VARCHAR(255) NOT NULL DEFAULT '',
  ADD COLUMN IF NOT EXISTS `org_same_as_extra` TEXT         NULL,
  ADD COLUMN IF NOT EXISTS `org_extra_jsonld`  TEXT         NULL;
```

- Ако сървърът не приема `ADD COLUMN IF NOT EXISTS`, ползвай механизма за
  идемпотентност от миграцията за `ai_version_*`.
- **Преди пускане питай дали проектът има продъкшън** (обща база; известно:
  `supercare` — да). Миграцията е само добавяща, но правилото важи.
- **`ProjectCompany::DB_COLUMNS` е явен списък** (`['id','phone','country_id','mol','eik','vat_number','dop']`).
  Новите шест колони **трябва да се добавят в него** и като типизирани public
  properties, иначе `save()` тихо ги пропуска (виж т.5 от конвенциите, „POST —
  пробват се всички properties“). Преводимите се зареждат в `loadObject()` с
  `Content::get(self::DB_TABLE, 1, 'org_description', $lang->language)` и не
  влизат в `DB_COLUMNS`.

## 5. Клас `Organization`

Файл: `core/v4/lib/classes/organization/class.organization.php`. Статичен клас по
образеца на `AiVersion` — **сглобява** от съществуващите източници и записва само
новите полета. Без `$db`: четенето е през `ProjectCompany::get(1)`,
`AdminFunctions::get_contacts()`, `AdminFunctions::get_module_settings_multy([…links…])`,
`Country::get()`; записът е през `ProjectCompany` + `Content::add()`.

```
Organization::TYPES = ['Organization', 'OnlineStore', 'LocalBusiness']
Organization::public_fields(): array        // договорът от т.6, за езика на заявката
Organization::get_settings(): array         // за админа: новите полета за текущия език + public_fields() + TYPES
Organization::save_settings(array $data): array   // само новите полета → ['status'=>1,'obj'=>get_settings(),'msg'=>_SUCCESS_EDITED] / ['status'=>0,'errors'=>[…]]
```

Валидация в `save_settings()` (грешки по поле с ключ `[name="…"]`):

| Поле | Правило |
|---|---|
| `org_type` | в `TYPES`; празно → `Organization` |
| `org_founder` | `trim`, до 255 |
| `org_founding_date` | празно, `YYYY` (4 цифри) или `YYYY-MM-DD` (валидна дата) → `_ORG_ERROR_DATE` |
| `org_contact_url` | празно, път започващ с `/`, или пълен `http(s)://` URL → `_ORG_ERROR_URL` |
| `org_same_as_extra` | всеки непразен ред е пълен `http(s)://` URL (`FILTER_VALIDATE_URL`); `trim`, без дубли и празни; пази се по един на ред → `_ORG_ERROR_URL` |
| `org_extra_jsonld` | празно или валиден JSON **обект** (не списък), до 10 000 символа, без ключове `@context`, `@id`, `@type`, `@graph`; пази се нормализирано (`json_encode` с `JSON_UNESCAPED_UNICODE \| JSON_UNESCAPED_SLASHES`) → `_ORG_ERROR_JSON` |
| `org_description` | до 1000 символа; `Content::add($value, 'company', 1, 'org_description')` за **текущия език на админа** |
| `org_alternate_names` | до 255; `Content::add(…, 'org_alternate_names')` |

Сглобяване в `public_fields()` — приоритет на източниците:

| Ключ в `organization` | Източник (първият непразен печели) | Нормализация |
|---|---|---|
| `type` | `company.org_type` | |
| `name` | `$core->site_name` | |
| `legal_name` | `company.name` (преводимо) → `cname` | |
| `vat_id` | `company.vat_number` | `trim`, главни букви |
| `tax_id` | `company.eik` → `ceik` | `trim` |
| `email` | `cmail` | `trim`; невалиден → `''` |
| `phone` | `cphone` → `company.phone` | само `+` и цифри (махат се интервали, скоби, тирета); без `+` в началото → остава както е |
| `street` | `company.address` (преводимо) → `caddress` | |
| `city` | `company.city` (преводимо) | |
| `postal_code` | `''` (няма източник; остава за `org_extra_jsonld` или втори етап) | |
| `country` | `Country::get(company.country_id)` → ISO alpha-2 с главни букви (`BG`) | провери кое поле на `Country` е кодът; ако няма, `''` |
| `contact_url` | `company.org_contact_url` | |
| `same_as` | `links.facebook`, `twitter`, `youtube`, `instagram` + редовете на `org_same_as_extra` | масив, само пълни `http(s)://` URL-и, без дубли и празни |
| `founder` | `company.org_founder` | |
| `founding_date` | `company.org_founding_date` | |
| `description` | Content `org_description` (език на заявката, fallback езика по подразбиране — вградено в `Content::get`) | |
| `alternate_names` | Content `org_alternate_names` → разделено по запетая | масив, `trim`, без празни и дубли |
| `extra` | `company.org_extra_jsonld` декодиран | обект; `{}` при празно |

Логото, URL-ът на сайта и ръчният скрипт **не** влизат тук — фронтендът ги има
от `settings_web` (`logo`, `scripts.organization`) и `settings` (`site_url`).

## 6. Договор: `GET /api/organization`

Нов публичен endpoint в `core/v4/api/front/Get.php` (само `saas_key`, без токен):

```php
public function organization(){
	load_class('organization');
	return Response::output(\Organization::public_fields());
}
```

`GET /api/settings` **не** се променя. Езикът е този на заявката (`?lang=`).
Отговорът е самият обект:

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

Правила на договора:
- Всички ключове са **винаги налице**; празното е `""`, `[]` или `{}`, никога
  липсващ ключ и никога `null`.
- Низовете са без HTML (`strip_tags`, `html_entity_decode`), `trim`-нати.
- Нов ключ се добавя в края и се отразява в двата документа и в `core-v4-front.md`.
- Кешът е във фронтенда (по език, ~1 час); бекендът не кешира и не версионира.
- Стар бекенд без endpoint-а → фронтендът пада на сегашното си поведение
  (име, URL, лого).

Админ endpoint-и (в `AdminGet.php` / `AdminPost.php`, до `ai_version()`, с docblock):

```
GET  /api/admin/organization   → Required::admin(); Response::output(\Organization::get_settings())
POST /api/admin/organization   → Required::admin(); Response::output(\Organization::save_settings(Request::getAllData()))
```

`get_settings()` връща: `fields` (шестте нови + двете преводими за текущия език
на админа, сурови — `org_same_as_extra` и `org_extra_jsonld` като текст),
`organization` (резултатът на `public_fields()`, за прегледа), `types`,
`custom_script_present` (bool — има ли нещо в `scripts.organization`), и
`missing` (виж т.7, индикатор за пълнота).

## 7. Админ страница „Организация“

В `admin.expozy.DEV/admin/v4/`, по образеца на „AI видимост“:
`settings.organization.php` (`$values['org'] = AdminApi::cache(false)->get()->organization();`)
и `templates/expozy/settings.organization.html`, бутон „Запази“ с
`alpineListeners('post.organization', $event)` и `@success="window.location.reload()"`.

**Място:** Settings → ред „Организация“ в `templates/expozy/settings.html`
(до „AI видимост“, ред ~228). Не в App Store.

**Съдържание на страницата, в този ред:**

1. **Индикатор за пълнота** (от `missing`): „Липсват: юридическо име, адрес,
   телефон, описание, социални профили“ — само препоръчителните празни полета
   (`legal_name`, `street`+`city`+`country`, `phone`, `email`, `description`,
   `same_as`). Нищо не се блокира; това е подсказка. Когато всичко е попълнено:
   „Всички препоръчителни данни са попълнени“.
2. **Общите данни — редактируеми тук, записват се в оригиналните места**
   (решение 2026-10-28: всичко за фирмата на едно място, без копия на данните):

   | Поле | Записва се в | През |
   |---|---|---|
   | Юридическо име, ЕИК, ДДС номер, адрес, град, държава | `company` („Данни за фактура“) | `ProjectCompany` |
   | Имейл, телефон | контакти (`cmail`, `cphone`) | `AdminFunctions` |
   | Facebook, Instagram, YouTube, Twitter | социални мрежи (`linkfb`…) | `AdminFunctions` |

   - При полета с два източника се редактира **основният** (този, който печели в
     т.5): телефон → контакти; име и адрес → `company`. Резервният не се пипа.
   - Юридическо име, ЕИК, ДДС и адрес се печатат и **на фактурите** — бележка в
     лейбъла на тези полета.
   - Преводимите (име, град, адрес) — за текущия език на админа, както в „Данни за фактура“.
   - Записът минава през методите на тези класове (без директни заявки), за да важат
     същите проверки. Старите страници остават.
3. **Нови полета (формата):**
   - Тип (select от `types`): „Организация“, „Онлайн магазин“, „Местен бизнес“.
   - Описание (textarea, преводимо — за текущия език на админа, както при
     другите преводими полета; бележка „за всеки език поотделно“).
   - Алтернативни имена (text, преводимо; „разделени със запетая“).
   - Страница за контакти (`/bg/contacts`).
   - Допълнителни профили (textarea, „по един пълен адрес на ред“ — за мрежи извън
     четирите в „Социални мрежи“).
   - Година или дата на основаване.
   - Основател (по желание).
   - Допълнителни свойства (JSON) — textarea с моноширинен шрифт, свита зад линк
     „Разширени“; подсказка „JSON обект със свойства на schema.org Organization;
     за напреднали“. Грешката от валидацията под полето.
4. **Ръчният скрипт.** Ако `custom_script_present` е true — предупреждение:
   „В Настройки → Скриптове има ръчен „Schema organization“ скрипт. Фронтендът го
   извежда както е, в допълнение към генерираната схема; за да няма двойни данни,
   прехвърлете съдържанието му в полетата тук (и в „Допълнителни свойства“) и го
   изтрийте.“ Полето не се пипа от тази страница.
5. Бележка под формата: „Данните се показват в структурираните данни (JSON-LD) на
   всяка страница. Име, лого, адрес на сайта и езици се взимат от общите настройки.“

Езикови константи `_ORGANIZATION`, `_ORGANIZATION_DESC`, `_ORG_TYPE`, …,
`_ORG_MISSING`, `_ORG_COMPLETE`, `_ORG_CUSTOM_SCRIPT_WARNING`, `_ORG_EDIT_IN`,
`_ORG_ERROR_DATE`, `_ORG_ERROR_URL`, `_ORG_ERROR_JSON` във **всички**
`admin/v4/core/lang/*.lang.php`, както са добавени `_AI_VERSION_*` (bg и en с
реален текст, останалите с английския). В `admin/v4/core/classes/class.core.php`
нищо ново — полетата не са в `settings`.

## 8. Тест (задължителен, по т.5 от конвенциите)

Срещу `https://devcore.myexpozy.com`, никога срещу продукционен домейн. GET без
питане, POST след потвърждение.

1. `GET /api/admin/organization` без токен → отказ. С токен → `fields`,
   `organization`, `types`, `custom_script_present`, `missing`.
2. `POST /api/admin/organization` с **всички** нови полета и различими стойности;
   после `GET` и сравнение поле по поле, вкл. че `org_same_as_extra` е без дубли и
   празни редове и че `org_extra_jsonld` е нормализиран.
3. Отрицателни: дата `23`, контактен URL `contacts` (без `/`), ред
   `linkedin.com/x` (без схема), JSON счупен, JSON списък `[1,2]`, JSON с ключ
   `@type` → `status: 0` и грешка по поле, не 500.
4. Преводими: `org_description` с `?lang=bg` и с `?lang=en`, после
   `GET /api/organization?lang=bg` и `?lang=en` връщат различен `description`;
   език без запис връща езика по подразбиране.
5. Сглобяване: промени телефон в „Контакти“ и име в „Данни за фактура“ → `GET
   /api/organization` ги отразява в `phone` / `legal_name` без нищо друго.
   `same_as` съдържа четирите от `links` плюс допълнителните, без дубли.
6. `GET /api/organization` (само `saas_key`): всички ключове от т.6 са налице с
   правилните типове и при напълно празна „Организация“. `GET /api/settings` е
   непроменен.
7. HTML вместо JSON = fatal error, чете се.

## 9. Документация

- `frontend_docs/core-v4-front.md`: нов раздел `GET /?route=organization` с обекта
  (таблицата от т.6 с ключове, типове и източници) — това е **трайното** място на
  договора.
- Admin endpoint-ите **не** се документират.
- `frontend_docs` се комитва и пушва; `devcore` и админ темплейтът **не**.
- **Накрая този файл (`core-organization-backend-spec.md`) се изтрива** от
  `frontend_docs` — задачата е приключена, договорът живее в `core-v4-front.md`.

## 10. Втори етап (не се прави сега; записано, за да е ясно защо липсва)

| Група | Полета | schema.org | Кога |
|---|---|---|---|
| **Пощенски код** | `postal_code` в `company` | `address.postalCode` | Когато „Данни за фактура“ получи поле за него; дотогава `org_extra_jsonld` → `"address"` не се ползва (адресът е типизиран), оставя се празно. |
| **Политика за връщане** | дни, държава, кой плаща, условия | `hasMerchantReturnPolicy` → `MerchantReturnPolicy` | Магазини; Google merchant listings, AI препоръки. |
| **Доставка** | региони, срок от–до, цена, праг за безплатна | `OfferShippingDetails` на ниво `Offer` | Магазини; данните са в Еконт/Спиди настройките — обединяване, не ново въвеждане. |
| **Физически обект** | работно време, координати, карта | `LocalBusiness` → `openingHoursSpecification`, `geo`, `hasMap` | Само с посещаван обект; тогава `org_type` = подтип (`Store`, `AutoDealer`…). |
| **Няколко точки за контакт** | тип, телефон, имейл, езици | `contactPoint[]` | Дотогава втората — в `org_extra_jsonld`. |
| **Оценки** | рейтинг и брой отзиви за организацията | `aggregateRating` | Само с реален източник; иначе спам за Google. |

Правило: типизирана колона само ако поне два проекта го ползват или има
валидация, която свободният JSON не дава. Иначе — `org_extra_jsonld`.

## 11. За уточнение преди код

1. Кои проекти имат продъкшън (за миграцията)? Известно: `supercare`.
2. `LocalBusiness` още сега или само `Organization` / `OnlineStore`?
3. Описанието преводимо (Content, както е предложено) или само на езика по
   подразбиране?
4. Има ли `Country` поле с ISO alpha-2 код? Ако не — откъде се взима `country`.
