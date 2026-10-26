# План: ai_version — PHP версия на страниците за AI ботове

Статус: стъпки 1–8 изпълнени и тествани на ecommerce.myexpozy.com (2026-10-22). Кодът в ecommerce още не е комитнат.
Решенията от обсъждането (т. 1–24) са събрани в раздел „Решения“ най-долу.

---

## Структура

```
ai_version/
  .htaccess              Deny from all
  config.php             enabled, списък ботове, slug → шаблон, noindex slug-ове, лимити
  classes/
    class.ai_bot.php     AiBot::detect() — разпознаване по User-Agent
    class.ai_render.php  AiRender — поток, чистене на HTML от Alpine, линкове, лог
  bootstrap.php          входна точка (виж „Поток“)
  layout.php             <html>/<head>/навигация/footer около шаблона
  llms_txt_generator.php преместен от core/sitemaps/
  templates/
    products.php         категория
    product.php
    blog.php
    post.php
    page.php             fallback — страниците от редактора (вкл. началната); apiData блоковете стават списъци
    _product_list.php    частични шаблони (AiRender::partial), ползват се и от api_blocks
    _post_list.php
    _pagination.php
    404.php, maintenance.php
    nav_only.php         checkout, cart, login… — само навигация + noindex
  lang/bg.php, en.php    етикети
  logs/                  YYYY-MM.log
  README.md              инсталация в нов проект + как се добавя шаблон
```

Промени извън папката:
- `index.php` — един `require` (виж „Поток“).
- `.htaccess` — правилата за `llms.txt` / `llms-full.txt` сочат към `ai_version/`.
- `core/sitemaps/llms_txt_generator.php` — изтрива се (преместен).

## Поток (bootstrap.php)

Hook-ът е **преди** `$page->load_page()` в `index.php`, защото `load_page()` прави
redirect към `/404`, а за ботове искаме директен статус 404.

```
index.php:
  require_once 'ai_version/bootstrap.php';   // при AI бот: рендерира и exit
  $page->load_page();                        // нормалният поток, непроменен
```

1. `config.enabled` е false или User-Agent не е AI бот → `return` (нищо не се променя).
2. Maintenance mode → статус 503 + `Retry-After`, exit.
3. `$page->load_page(true, false)` — с кеш, без redirect.
4. `error404` → статус 404 + кратко съобщение + навигация.
5. Частна страница → оставяме поведението на `load_page()` (redirect към login).
6. Избор на шаблон по `config.templates[slug]` (по `$page->type` за product/post),
   иначе `page.php`; slug в `config.noindex` → `nav_only.php`.
7. Заглавки: `Content-Type: text/html; charset=UTF-8`, `Vary: User-Agent`,
   `Cache-Control: no-store`, `X-AI-Version: 1`; `noindex` при nav_only.
8. Рендер: `layout.php` + шаблон.
9. Лог ред: дата | бот | URL | статус | ms.
10. `exit`.

## Ред на изпълнение

Всяка стъпка завършва с тест, преди да започне следващата.

| # | Стъпка | Файлове | Готово, когато |
|---|---|---|---|
| 0 | Предварително | — | Cloudflare проверен (Block AI bots, cf-cache-status); стъпки 1 и 3 тествани и комитнати |
| 1 | Скелет | `.htaccess`, `config.php`, `class.ai_bot.php`, `bootstrap.php`, `layout.php`, `nav_only.php`, `lang/*`, ред в `index.php` | Нормален UA → страницата е байт по байт същата. GPTBot UA → `X-AI-Version: 1`, навигация с категории и footer линкове, hreflang, JSON-LD, лог ред. Непознат URL → 404 |
| 2 | Статични страници | `class.ai_render.php` (чистене), `page.php` | about / gdpr / contacts: текстът е налице, няма `x-`, `@`, `:` атрибути, `<template>`, `<script>` |
| 3 | Продукт | `product.php` | h1, цена + промо, вариации, наличност, описание, атрибути, ≤5 снимки в пълен размер, ≤10 отзива, FAQ. Цената съвпада с Alpine версията |
| 4 | Категория | `products.php` | 50 продукта/стр., линкове prev/next с `?page=N`, всеки с линк към продукта |
| 5 | Блог | `blog.php`, `post.php` | Списък с пагинация; статия с пълен текст, дата, снимка |
| 6 | Начална | `page.php` + `config.api_blocks` | apiData="get.products" / "get.blogPosts" блоковете от редактора се заменят със списъци на мястото на `<template x-for>`, със същите параметри — без отделен home.php |
| 7 | llms.txt + robots.txt | преместване на llms генератора + `.htaccess`; нов `core/sitemaps/robots_txt_generator.php` | `/llms.txt` и `/llms-full.txt` връщат същото като преди; `/robots.txt` има `Sitemap: https://{домейн}/sitemap` с домейна на текущия проект |
| 8 | Документация | `README.md`, конвенции | Виж „Конвенции“ |

## Тестове (за всяка стъпка)

```
curl -s  -A "Mozilla/5.0 (compatible; GPTBot/1.2; +https://openai.com/gptbot)" https://ДОМЕЙН/bg/product/6-x
curl -sI -A "...GPTBot..." https://ДОМЕЙН/bg/product/6-x        # X-AI-Version, статус
curl -s  https://ДОМЕЙН/bg/product/6-x                           # нормалната — без промяна
```

Проверки: няма останали Alpine атрибути (`grep -c 'x-'` = 0), статус кодовете
(200 / 404 / 503), данните съвпадат с Alpine версията, лог редът е записан.

## Конвенции (стъпка 8)

- Нов раздел „24. AI ВЕРСИЯ“ в `frontend_docs/frontend_framework.md` (отделно repo):
  кога агентът пипа AI версията (таблица), задължителен ред в края на задачата
  „AI версия: засегната / не е засегната — защо“, AI версията като трето огледало
  до SEO двойката от раздел 23.
- `ai_version/README.md` — инсталация в нов проект и стъпки за нов шаблон.
- Нов тип страница (хотел, обява) = отделен шаблон по примера в `ai_version/README.md`
  („Нов тип страница“) + ред в `config.php`; списък от API = частичен шаблон + `api_blocks`.

## Среда

- Тестов домейн: https://ecommerce.myexpozy.com/ (NetBeans качва там; ecomerce.expozy.net е друг сървър)
- Всеки записан файл се качва автоматично → всяка стъпка е веднага на живо.
  Дев среда е, затова отделен тестов режим не е нужен (т. 29).

## TODO след това

- Логовете към ядрото (devcore) вместо файл.
- (последно) При запис от редактора да се чисти и кешът на списъците
  (`{endpoint}_0_…`), не само на обекта — един ред в `Editor::_refresh_cache()`.

Готово: Cloudflare (Security → Bots) проверен; footer линкът `/for-us` и примерните
текстове в Контакти — оправени в редактора; излишните файлове на сървъра изтрити.

---

## Решения

1. Само AI ботове получават PHP версията; Google / Bing / Apple — нормалната (изпълняват JS).
2. Съдържанието е същото като в Alpine версията (без допълнителна информация).
3. Тест с curl и бот User-Agent. 4. Лог на посещенията.
5. Всичко в `/ai_version/` в корена, за копиране между проекти.
6. Статичните страници — HTML от `static/pages/` в момента, изчистен; без копия.
7. Навигация — категории от API + линкове от footer.html; `get.pages` като fallback.
8. `llms_txt_generator.php` се мести; `class.geo_optimizer.php` остава в core.
9. Лог във файл; TODO към ядрото. 10. `geo_faq_api.php` остава.
11. FAQ от GeoOptimizer като видим h2/p.
13. Шаблони: home, products, product, blog, post, page; checkout/cart/login/… → nav_only + noindex.
14. Език от URL + hreflang. 15. Цена + промо от variations, списък вариации.
16. ≤5 снимки в пълен размер. 17. ≤10 отзива. 18. `Api::cache(true)` за допълнителните заявки.
19. Директен 404 статус. 20. Категория: 50/стр., `?page=N`. 21. Лог: дата, бот, URL, статус, ms; месечни файлове.
22. `Vary: User-Agent`, `Cache-Control: no-store`, `X-AI-Version: 1`.
23. JSON-LD от `GeoOptimizer::headTags()`. 24. `enabled` флаг в config.
Разпознаване само по User-Agent, без IP проверка.
25. Конвенциите — в `frontend_docs/frontend_framework.md`. 26. Отделен шаблон за всеки тип страница.
27. Тест само с curl. 28. Тестов домейн ecommerce.myexpozy.com (там качва NetBeans).
29. Без тестов режим — дев среда. 30. Конвенциите се пишат накрая (стъпка 8).
31. Планът живее във `frontend_docs/ai_version_plan.md`.
32. `robots.txt` се генерира от PHP (`core/sitemaps/`, не в ai_version) с ред `Sitemap:` за домейна на проекта.
33. Мерене на посещения от AI (utm_source=chatgpt.com и т.н.) — не за сега.

---

## Ниво 2: кратък кеш на AI HTML (предпазна мрежа)

Ниво 1 (направено): продуктът/статията/категорията се теглят веднъж на заявка
(`Page::$target`), без кеш на данните.

Ниво 2 — кеш на целия AI HTML за кратко, само като мрежа срещу масово обхождане.
Обхождащите ботове индексират на дни; за живите заявки (ChatGPT-User…) цена,
остаряла с минути, е същото, което показва Google.

Решения (т. 43–46):
- 43. TTL по тип: `product` и `products` — 5 мин., всичко останало — 15 мин.
  (`config.page_cache_ttl`; 0 = изключено).
- 44. Проверката е ПРЕДИ `core/autoload.php` (`ai_version/cache_hit.php`, първи ред в
  `index.php`) → при попадение 0 заявки към API-то. Компромис: DDoS проверката и
  режимът на поддръжка се пропускат за кешираните страници до изтичането им.
- 45. Запис от редактора (страница/статия) трие целия AI кеш.
- 46. Записано тук.

Как:
- Ключ = път + `?page=N`; другите параметри се игнорират (не могат да заобиколят кеша).
  User-Agent не влиза в ключа.
- Кешира се само 200. Файлове в `ai_version/cache/` (в .gitignore, достъп забранен).
- Атомичен запис (tmp + rename); изтекъл файл се трие при четене; на всеки ~100
  записа се трият всички изтекли.
- Заглавка `X-AI-Cache: HIT|MISS`; в лога — HIT/MISS.
- Не в Cloudflare: той не разделя кеша по `Vary: User-Agent`.
