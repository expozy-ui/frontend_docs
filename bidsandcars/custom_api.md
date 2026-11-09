# bidsandcars — Custom API (front)

Документация за къстъм front endpoint-ите на проекта —
`projects/bidsandcars/api/front/{Get,Post,Delete,Put}.php` в devcore.
Ядрените endpoint-и (вход, профил, блог и т.н.) са в `core-v4-front.md` и тук не се повтарят.
Админските (`api/admin/*`) не са част от този документ.

Сайт: `https://bidsandcars.bg` · език на текстовете в отговорите: **български**.

> Документът започва с endpoint-ите, добавени след 2026-11-09. Останалите
> къстъм endpoint-и още не са описани тук.

---

## Съдържание

1. [Общи конвенции](#1-общи-конвенции)
2. [Аналитика — сървърни евенти](#2-аналитика--сървърни-евенти)

---

## 1. Общи конвенции

**Base URL:** `https://devcore.myexpozy.com/api`. Името на PHP метода = пътят.

| Хедър | Кога | Описание |
|---|---|---|
| `Authentication: Basic <saas_key>` | винаги | Ключът на проекта (публичен — `const SAAS_KEY` в HTML-а на сайта). Може и `?saas_key=`. |
| `Authorization: Bearer <token>` | 🔒 endpoint-ите | Токенът на логнатия потребител. |

🔓 — публичен (само `saas_key`), 🔒 — изисква логнат потребител.

POST тялото е `application/x-www-form-urlencoded` или `multipart/form-data`.

| Случай | Тяло |
|---|---|
| Успешен запис | `{ "status": 1 }` |
| Грешка | HTTP **400** + `{ "status": 0, "error": "текст" }` |

---

## 2. Аналитика — сървърни евенти

Три евента се пращат **от сървъра**, не от браузъра, защото се случват без
потребителят непременно да е на сайта:

| Евент (GA4) | Meta | Кога |
|---|---|---|
| `kyc_completed` | `SubmitApplication` | админът одобрява верификацията на купувача (само първото одобрение) |
| `auction_deposit` | `AddPaymentInfo` | Stripe потвърждава платен депозит |
| `auction_won` | `Purchase` | купувачът печели — край на аукциона над резерва, „Купи сега“ или приета насрещна оферта |

**Фронтът НЕ ги праща** — нито през `track()`, нито през `analytics.js`.
Иначе ще има дубли. Единственото, което фронтът трябва да прави, е да подава
съгласието и бисквитките през endpoint-а по-долу.

Сървърът праща евент към GA4 само при съгласие `analytics`, а към Meta — само
при `targeting`. Ако потребителят никога не е подал данните си, евент не се праща.

### 2.1. `POST /analytics_identity` 🔒

Записва избора от cookie банера и аналитичните бисквитки на **влезлия**
потребител. Сървърът пази последните подадени стойности (по един запис на
потребител). IP адресът и user agent-ът се взимат от самата заявка — не се подават.

**Кога да се вика:**
- при всяко зареждане на страница, ако потребителят е влязъл (`LOGGED_IN == 1`) —
  `_fbc` се сменя, когато човек дойде от нова реклама;
- веднага след всяка промяна в банера (събитието `cookie_consent_update` /
  `applyConsent()`), ако потребителят е влязъл;
- веднага след успешен вход или регистрация.

Достатъчно е да се вика веднъж на зареждане (не при всяка SPA навигация).
Може да се пропусне, ако стойностите не са се променили от последното
изпращане (напр. кеш в `sessionStorage`).

**Параметри:**

| Име | Тип | Задълж. | Описание |
|---|---|---|---|
| `consent_analytics` | int 0/1 | да | 1 ако в банера е приета категорията `analytics` |
| `consent_targeting` | int 0/1 | да | 1 ако в банера е приета категорията `targeting` |
| `ga_client_id` | string | не | стойността на бисквитката `_ga` както е (`GA1.1.123456789.1700000000`) — сървърът сам вади client_id |
| `fbp` | string | не | стойността на бисквитката `_fbp` |
| `fbc` | string | не | стойността на бисквитката `_fbc` |

Празна или липсваща бисквитка не трие вече записаната стойност. Съгласието
обаче винаги се презаписва с подаденото — при отказ подавайте `0`.

Ако потребителят е отказал категорията, съответните бисквитки няма да ги има —
това е нормално, подайте само съгласието.

**Пример:**

```js
function cookie(name) {
	const m = document.cookie.match(new RegExp('(?:^|;\\s*)' + name + '=([^;]*)'));
	return m ? decodeURIComponent(m[1]) : '';
}

function sendAnalyticsIdentity(categories) {
	if (LOGGED_IN != 1) return;
	categories = categories || [];

	const body = new URLSearchParams({
		consent_analytics: categories.includes('analytics') ? 1 : 0,
		consent_targeting: categories.includes('targeting') ? 1 : 0,
		ga_client_id: cookie('_ga'),
		fbp: cookie('_fbp'),
		fbc: cookie('_fbc')
	});

	// + Authorization: Bearer <токена на потребителя>, както при останалите 🔒 заявки
	fetch(COREURL + 'analytics_identity', {
		method: 'POST',
		headers: { 'Authentication': 'Basic ' + SAAS_KEY },
		body
	}).catch(() => {});
}
```

Използвайте помощния механизъм за заявки, който вече ползвате на сайта
(той слага хедърите сам) — горното е само за илюстрация.

**Отговор (успех):**

```json
{ "status": 1 }
```

**Грешки:**

| Кога | Отговор |
|---|---|
| няма вход / невалиден токен | HTTP 400, `{ "status": 0, "errors": ["Login first v4!"] }` |

### 2.2. Задача за фронта — как да се вържe

Целта: при всеки известен избор в банера, ако потребителят е влязъл, да се
извика `POST /analytics_identity`. Нищо друго — евентите `auction_deposit`,
`kyc_completed` и `auction_won` **не се добавят** в `analytics.js` и не се пращат
през `track()`.

**1. Една функция в `pages/header.php`**, до `window.applyConsent` (в блока
„Google Consent Mode v2“):

```js
window.sendAnalyticsIdentity = function (categories) {
	// LOGGED_IN / COREURL се дефинират в по-късен <script> блок — при
	// синхронното извикване по-горе в header.php още ги няма.
	if (typeof LOGGED_IN === 'undefined' || LOGGED_IN != 1) return;
	categories = categories || [];

	function cookie(name) {
		var m = document.cookie.match(new RegExp('(?:^|;\\s*)' + name + '=([^;]*)'));
		if (!m) return '';
		try { return decodeURIComponent(m[1]); } catch (e) { return m[1]; }
	}

	var payload = {
		consent_analytics: categories.indexOf('analytics') > -1 ? 1 : 0,
		consent_targeting: categories.indexOf('targeting') > -1 ? 1 : 0,
		ga_client_id: cookie('_ga'),
		fbp: cookie('_fbp'),
		fbc: cookie('_fbc')
	};

	// Без повторно пращане в рамките на сесията, ако нищо не се е променило.
	var key = JSON.stringify(payload);
	try { if (sessionStorage.getItem('bc_ai') === key) return; } catch (e) {}

	// POST към COREURL + 'analytics_identity' с payload-а — през същия механизъм
	// за заявки, с който се пращат останалите 🔒 POST-ове (напр. web_notifications_read),
	// за да тръгнат хедърите Authentication + Authorization. Fire-and-forget:
	// грешка тук не бива да се показва на потребителя.
	// При успех: try { sessionStorage.setItem('bc_ai', key); } catch (e) {}
};
```

**2. Извикване от `window.applyConsent`** — то вече се вика и при първия избор,
и при всяко зареждане със запазен избор (`onAccept`), и при промяна (`onChange`)
в `coockieconsent_init.js`. Така едно място покрива всички случаи:

```js
window.applyConsent = function (categories) {
	gtag('consent', 'update', window.consentState(categories));
	if (window.sendAnalyticsIdentity) window.sendAnalyticsIdentity(categories);
};
```

`applyConsent` се вика веднъж и синхронно в самия `header.php` (за връщащ се
посетител, преди GTM). Тогава `_ga`/`_fbp` може още да не са поставени —
това е нормално: празните стойности не трият записаните, а `onAccept` на
`window.load` ще прати отново с бисквитките (ключът в `sessionStorage` е
различен, защото стойностите са различни).

**3. След успешен вход / регистрация** (имейл, Google, Facebook — в `user.js`,
там, където сега се вика `analytics('login', …)` / `analytics('sign_up', …)`):
ако страницата не се презарежда след входа, извикайте

```js
window.sendAnalyticsIdentity && window.sendAnalyticsIdentity(savedCategories());
```

където `savedCategories()` чете `cc_cookie` (както прави блокът в `header.php`)
и връща `saved.categories` или `[]`. Ако след вход страницата се презарежда,
стъпка 3 не е нужна — `onAccept` ще го направи.

**Проверка:** влезте, приемете всички бисквитки → в DevTools → Network трябва
да има една заявка `analytics_identity` с отговор `{"status":1}`. При откажи
всички → заявка с `consent_analytics=0&consent_targeting=0`. Без вход —
никаква заявка.
