# Задача за бекенда (devcore) и админа: управление на „AI видимост“

Статус: за изпълнение. Фронтендът е готов да чете новите полета и до тогава работи с
резервните стойности от `ai_version/config.php`.

> **Когато приключиш работата и тестовете от т. 5 минат — изтрий този файл**
> (`frontend_docs/ai_version_backend_task.md`), комитни и пушни `frontend_docs`.
> Той е временна задача, не постоянна документация.

Преди да пишеш код в devcore — зареди скила `devcore-conventions` и спазвай
`devcore/CLAUDE.md` (без комити, миграции без FOREIGN KEY, тест с реална заявка
срещу https://devcore.myexpozy.com, POST — след питане).

---

## Контекст (накратко)

Фронтендът има модул `ai_version/`: AI ботовете (GPTBot, ClaudeBot…) получават
опростена PHP версия на страниците, защото не изпълняват JS. Разпознават се по
User-Agent. Готовият AI HTML се кешира за кратко.

Фронтендът вика `GET /api/settings` при **всяка** заявка (`FrontCore::get_settings()`)
и копира всички полета от отговора в `$core`. Затова новите настройки трябва просто да
се появят в този отговор — без нов endpoint и без нови заявки.

Целта: собственикът на сайта да включва/изключва модула и да сменя времето на кеша от
админа, и да вижда (без да редактира) списъка с ботове.

---

## 1. База — три нови колони в `settings`

Не в `modules_settings`: `Core::get_settings()` вече прави `SELECT * FROM settings
WHERE id = 1` при всяка API заявка и копира колоните в `$core`, а публичният
`/api/settings` връща `(array)$core` → новите колони идват **без нови заявки**.
(`modules_settings` би добавил заявка при всеки `settings`; фронтендът го вика 2 пъти
на страница.)

| Колона | Тип | По подразбиране | Смисъл |
|---|---|---|---|
| `ai_version_enabled` | `TINYINT(1) NOT NULL` | `1` | модулът е включен |
| `ai_cache_ttl_product` | `INT NOT NULL` | `300` | секунди кеш на AI HTML за продукт и категория (там са цените) |
| `ai_cache_ttl_default` | `INT NOT NULL` | `900` | секунди кеш за всички останали страници |

- Миграция за всеки проект (по конвенцията на devcore за миграции).
- Свойства в `Core` (v4, и v3, ако проектите на v3 също трябва да го имат), напр.
  `public int $ai_version_enabled = 1;` и т.н. — по образеца на `maintenance_mode`.
- Валидация при запис: TTL — цяло число от **0 до 3600** (0 = без кеш);
  `ai_version_enabled` — 0 или 1.

## 2. Списъкът с ботове — константа в кода, не в базата

В админа списъкът **само се показва**, не се редактира. Затова живее в кода на
devcore — един източник за всички проекти: нов AI бот се добавя на едно място.

Предложение: клас (напр. `AiVersion` в `core/v4/lib/classes/`) с:

```php
// Части от User-Agent (без значение главни/малки). Само AI ботове.
public const BOTS = [
    'GPTBot', 'OAI-SearchBot', 'ChatGPT-User',
    'ClaudeBot', 'Claude-SearchBot', 'Claude-User', 'anthropic-ai',
    'PerplexityBot', 'Perplexity-User',
    'CCBot',
    'Meta-ExternalAgent', 'Meta-ExternalFetcher',
    'Bytespider',
    'Amazonbot',
    'DuckAssistBot',
    'MistralAI-User',
    'cohere-ai',
    'YouBot',
];

// Тези НИКОГА не получават AI версията — изпълняват JS, а различно съдържание
// за тях е cloaking (вреди на SEO). Филтрират се и тук, и във фронтенда.
public const PROTECTED = [
    'Googlebot', 'Google-InspectionTool', 'GoogleOther', 'AdsBot-Google',
    'Mediapartners-Google', 'Storebot-Google',
    'bingbot', 'BingPreview', 'adidxbot',
    'Applebot',
];
```

Правило при връщане (функция, напр. `AiVersion::bots(): array`): махат се празни
низове, низове **под 4 символа** и всеки низ, който съвпада (без значение главни/малки,
като подниз в едната или другата посока) с някой от `PROTECTED`.

## 3. Публичният `GET /api/settings`

Файл: `core/v4/api/front/Get.php` → `settings()` (и v3, ако е нужно).

Освен трите колони (идват сами от `$core`) добави:

```php
$out = (array)$core;
$out['ai_version_bots'] = \AiVersion::bots();   // масив от низове
return Response::output($out);
```

Очакван отговор (само новите полета):

```json
{
  "ai_version_enabled": 1,
  "ai_cache_ttl_product": 300,
  "ai_cache_ttl_default": 900,
  "ai_version_bots": ["GPTBot", "OAI-SearchBot", "ChatGPT-User", "..."]
}
```

Типове: `ai_version_enabled` — цяло число 0/1 (фронтендът приема и bool);
TTL — цели числа; `ai_version_bots` — масив от низове.

**Това е договорът с фронтенда** — имената и типовете трябва да са точно тези.

## 4. Админ — App Store → AI → „AI видимост“

В `devadmin.expozy.com` → Settings → **App Store**, в групата **AI**, нов ред
**„AI видимост“**, който отваря страница с:

1. **Превключвател** „Активен модул“ → `settings.ai_version_enabled`.
2. **Кеш на AI страниците** (секунди, 0–3600, 0 = без кеш):
   - „Продукти и категории“ → `ai_cache_ttl_product`
   - „Всички останали страници“ → `ai_cache_ttl_default`
3. **AI ботове** — списъкът от `AiVersion::bots()`, **само за четене**.
4. Бележка: „Google, Bing и Apple изпълняват JavaScript и винаги получават нормалната
   страница — те не са в списъка и не могат да бъдат добавени.“
5. Кратко описание какво прави модулът: AI ботовете (ChatGPT, Claude, Perplexity…)
   получават съдържанието на страниците като обикновен HTML, защото не изпълняват JS.

Ако App Store изисква запис в `modules_new` / `AppStoreModules` (нов `ID_…`), за да
се покаже редът — направи го; стойностите обаче се пазят в колоните на `settings`.

Нужни admin endpoint-и (по образеца на останалите настройки в
`core/v4/api/admin/AdminGet.php` / `AdminPost.php`):
- четене: трите стойности + `AiVersion::bots()`;
- запис: трите стойности с валидацията от т. 1.

## 5. Тестове (срещу devcore.myexpozy.com)

- `GET /api/settings` с SAAS ключа на проект `ecommerce.myexpozy.com` → съдържа
  четирите полета с очакваните типове и стойности по подразбиране.
- `ai_version_bots` не съдържа нищо от `PROTECTED` и няма низове под 4 символа.
- Admin: запис на TTL 5000 → грешка; 0 и 3600 → приема; `ai_version_enabled` 0/1.
- След смяна в админа → `GET /api/settings` връща новите стойности.
- Документирай новите admin endpoint-и според правилата на devcore.

## 6. Какво прави фронтендът с тези полета (за справка — вече е направено)

- `ai_version_enabled = 0` → AI ботовете получават нормалната страница.
- TTL-ите заменят `page_cache_ttl` от `ai_version/config.php`.
- `ai_version_bots` замества списъка `bots` от `config.php`; фронтендът пак филтрира
  Google/Bing/Apple и низове под 4 символа.
- Липсващо поле (стар бекенд) → стойността от `config.php`.
- Промяна в админа се прилага при следващото отваряне на страница от сайта.

## 7. Накрая

Всичко от т. 1–5 е готово и тествано → **изтрий този файл**, комитни и пушни
`frontend_docs` (той е единственото repo, което се комитва само).
