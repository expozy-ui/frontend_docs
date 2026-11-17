# Демо Магазин (demo-shop) — Custom API

Документация за къстъм endpoint-ите на demo-shop (`projects/demo-shop/api/front/*.php` в devcore).
Ядрените endpoint-и (login, accounts…) са в `core-v4-front.md`.

Демото е само на български.

---

## 1. Свързване

```
База (дев):  https://devcore.myexpozy.com/api/
```

Всяка заявка носи хедър, който казва **кой проект** е:

```
Authentication: Basic <saas_key>
```

`saas_key` може и като `?saas_key=...`. Endpoint-ите по-долу не искат потребителски токен.

### Формат на грешките

- Грешка по поле → `{"status": 0, "errors": {"[name=\"email\"]": "текст"}}` — ключът е CSS селектор на полето във формата
- Обща грешка → `{"status": 0, "error": "текст", "errors": ["текст"]}`

Текстовете са на български и са готови за показване на потребителя.

---

## 2. Регистрация за демото

### `POST /demo_signup`

> **Промяна (2026-11-17, breaking):** до тази дата endpoint-ът се казваше `POST /demo_<проект>_signup`. Сега е общ за всички демо проекти — `POST /demo_signup`; проектът се определя от `saas_key`. Параметрите и отговорът са същите.

Създава админ акаунт за демото (ниво Админ), праща мейл до админа на проекта с данните (без паролата)
и приветствен мейл до потребителя. Връща токен за директен вход в админ панела.

Достъп: само `saas_key`, без токен.

Тяло: `multipart/form-data` или `application/x-www-form-urlencoded`.

| Параметър | Тип | Задължителен | Описание |
|---|---|---|---|
| `first_name` | string | да | Име |
| `last_name` | string | да | Фамилия |
| `email` | string | да | Имейл — става потребителското име за вход в админа. Трябва да е валиден и да не е регистриран |
| `password` | string | да | Парола, минимум 8 символа |
| `website` | string | не | **Honeypot срещу ботове.** Сложи го във формата като скрито поле (`display:none`, `tabindex="-1"`, `autocomplete="off"`) и го праща празно. Попълнено → регистрацията се отказва с обща грешка |

Ограничение: най-много 5 регистрации от едно IP за час.

#### Отговор при успех

| Поле | Тип | Описание |
|---|---|---|
| `status` | int | `1` |
| `user_id` | int | id на новия потребител |
| `token` | string | Токен за вход в админа (64 знака, може да съдържа `@ # $ ! / \|`) |
| `project_name` | string | `demo-shop` — подава се при входа с токен |
| `ip` | string | IP на клиента, както го вижда сървърът — подава се при входа с токен |
| `admin_url` | string | Страницата за вход с имейл и парола: `https://devadmin.expozy.com/login/demo-shop` |
| `login_url` | string | Адресът за директен вход с токена: `https://devadmin.expozy.com/admin/login_demo.php` |

```json
{
    "status": 1,
    "user_id": 10000,
    "token": "<64 знака>",
    "project_name": "demo-shop",
    "ip": "<IP>",
    "admin_url": "https://devadmin.expozy.com/login/demo-shop",
    "login_url": "https://devadmin.expozy.com/admin/login_demo.php"
}
```

#### Грешки

| Кога | Отговор |
|---|---|
| Липсва поле | `errors` по поле: `Моля, въведете име.` / `Моля, въведете фамилия.` / `Моля, въведете Email адрес.` / `Моля, въведете парола.` |
| Невалиден имейл | `errors["[name=\"email\"]"]` = `Въведеният Email адрес е невалиден.` |
| Имейлът вече е регистриран | `errors["[name=\"email\"]"]` = `Този Email е вече регистриран.` |
| Парола под 8 символа | `errors["[name=\"password\"]"]` = `Паролата е твърде кратка - минимум 8 символа.` |
| Попълнен `website` | `error` = `Грешка` |
| Над 5 регистрации от едно IP за час | `error` = `Твърде много регистрации от този адрес. Опитайте отново след час.` |

```json
{
    "status": 0,
    "errors": {
        "[name=\"email\"]": "Въведеният Email адрес е невалиден.",
        "[name=\"password\"]": "Паролата е твърде кратка - минимум 8 символа."
    }
}
```

### Вход в админа след регистрацията

Входът с токен е **POST към `login_url`**, не линк — затова след успешен отговор фронтът създава скрита форма
и я изпраща сам. Браузърът отива в админа вече влязъл.

```html
<form id="demoLogin" method="POST" action="https://devadmin.expozy.com/admin/login_demo.php">
    <input type="hidden" name="token" value="">
    <input type="hidden" name="project_name" value="">
    <input type="hidden" name="ip" value="">
</form>
```

```js
const f = document.getElementById('demoLogin');
f.action = res.login_url;
f.token.value = res.token;
f.project_name.value = res.project_name;
f.ip.value = res.ip;
f.submit();
```

- Адресите винаги от отговора (`login_url`, `admin_url`), не хардкоднати — на 17.11.2026 се смениха на `devadmin.expozy.com` и `login_demo.php` (води направо в таблото на админа).
- Стойностите се слагат през `.value`, не с конкатенация в HTML — токенът съдържа специални знаци.
- Ако директният вход не мине, пращай потребителя на `admin_url` — там влиза с имейла и паролата си.
- Покажи на потребителя, че демото **се възстановява всяка нощ**: промените по данните се губят, акаунтът остава.
