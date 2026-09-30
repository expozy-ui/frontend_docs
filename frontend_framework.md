---
name: expozy-alpine
description: Конвенции за писане на HTML страници за Expozy SPA платформата (Alpine.js + Tailwind + custom framework layer) - window.data, apiData, alpineListeners(), callModal(), href(), пагинация, @success/@error callbacks, is-section структура, API endpoints. Използвай ЗАДЪЛЖИТЕЛНО при създаване или редакция на .html страници в Expozy проект (public_html/static/pages/**), при въпроси за alpineListeners, apiData, callModal, corePage, is-section, или когато проектът съдържа Alpine.js SPA router с #main контейнер.
---
Ти генерираш HTML страници за SPA платформа, базирана на Alpine.js с custom framework layer.
Спазвай СТРИКТНО описаните програмни конвенции.

═══════════════════════════════════════════════════════════
1. ТЕХНОЛОГИЧЕН СТАК
═══════════════════════════════════════════════════════════

- Alpine.js — реактивност, състояние, условен рендер, цикли
- Tailwind CSS — utility-first стилове (НЕ пиши custom CSS, НЕ ползвай <style>)
- Font Awesome 6 — икони (fa-solid, fa-regular, fa-brands)
- Чист HTML фрагмент — без <html>, <head>, <body>
- Страницата се зарежда в #main контейнер от SPA router-а

═══════════════════════════════════════════════════════════
2. ГЛОБАЛЕН data ОБЕКТ
═══════════════════════════════════════════════════════════

Цялото състояние живее в `window.data` (Alpine.reactive).
Достъпен е навсякъде в HTML чрез Alpine expressions.

Вградени ключове:

  data.corePage          — { id, target_id } — метаданни за текущата страница

  data.user              — текущ потребител:
    {
      id: number,
      email: string,
      names: string,              // пълно име
      first_name: string,
      last_name: string,
      phone: string,
      phone_code: string,
      userlevel: number,          // 1=обикновен, 13=админ и т.н.
      logged_in: true|false,
      token: string,
      balance: number,
      date_created: string,       // "2026-02-03 10:02:29"
      addresses: null|array,
      company: null|object,
      rental_host: null|object
    }

  data.pageUrl            — обект с URL параметрите
                            напр. data.pageUrl.id, data.pageUrl.speciality_id
                            Празен {} ако няма параметри

  data.settings           — { logo: string, social: { facebook, twitter, youtube, instagram } }

  data.modals             — масив с активни модали (toggle чрез callModal)

  data.cities             — обект с градове (зарежда се с apiData="get.cities"):
    {
      "0": { id, country_id, post_code, important, region_id, title },
      "1": { ... },
      ...
    }
    Достъп: Object.values(data.cities).reverse()

  data.regions            — обект с региони (зарежда се с apiData="get.regions"):
    {
      "0": { id, country_id, title },
      ...
    }

  UI състояния:
    data.openCart           — boolean
    data.openMobileMenu     — boolean
    data.openLogin          — boolean
    data.openRegistration   — boolean
    data.openProduct        — boolean
    data.openForgotten      — boolean
    data.openGeolocation    — boolean
    data.mobileFilters      — boolean (за мобилен филтър drawer)
    data.darkMode           — boolean (от localStorage)
    data.screenWidth        — number (viewport ширина)
    data.scrollPosition     — number (текущ scroll Y)
    data.location           — array (геолокация)

Динамични ключове се създават от API отговорите чрез keyName атрибут.
Например: `keyName="doctors"` → `data.doctors` съдържа отговора.

═══════════════════════════════════════════════════════════
3. ЗАРЕЖДАНЕ НА ДАННИ (apiData)
═══════════════════════════════════════════════════════════

За автоматично зареждане на данни при рендер, добави `apiData` атрибут
на ВСЕКИ HTML елемент:

```html
<div apiData="get.cities">
    <!-- data.cities е достъпен тук -->
    <template x-for="city in Object.values(data.cities).reverse()" :key="city.id">
        <span x-text="city.title"></span>
    </template>
</div>
С custom keyName:


<section apiData="get.blogPosts" data-limit="3" keyName="featurePosts">
    <template x-for="post in data.featurePosts.result" :key="post.id">
        <div x-text="post.title"></div>
    </template>
</section>
Правила:

Без keyName → ключът е последната част от endpoint ("cities" от "get.cities")
С keyName → данните се записват в data[keyName]
Зареждането е автоматично при init на елемента

ВАЖНО: apiData сканира ВСИЧКИ data-* атрибути на елемента и ги изпраща
като параметри към заявката. Атрибутът data-limit="3" се изпраща като
?limit=3 в GET заявката. Всеки data-* атрибут = параметър.

Примери:
  data-limit="3"          → limit=3
  data-page="2"           → page=2
  data-order_by="date"    → order_by=date
  data-tag="news"         → tag=news
  :data-id="item.id"      → id=динамична стойност (Alpine binding)
═══════════════════════════════════════════════════════════
4. alpineListeners() — ГЛАВЕН МЕХАНИЗЪМ ЗА ЗАЯВКИ
═══════════════════════════════════════════════════════════

Сигнатура: alpineListeners(method, element)

Това е ЕДИНСТВЕНАТА функция за API повиквания и локални действия.
Извиква се при потребителски действия (@click, @change, @submit).

4.1 API заявки (CRUD)
Методът определя HTTP verb-а чрез префикс:

"get.doctors"              → GET /doctors
"post.doctors_ratings"     → POST /doctors_ratings
"put.doctors"              → PUT /doctors
"delete.doctors_appointment" → DELETE /doctors_appointment


<!-- Зареждане при промяна на филтър -->
<select name="speciality_id"
        @change="alpineListeners('get.doctors', $el)">
    <option value="empty">Всички</option>
</select>

<!-- Изтриване с data атрибут -->
<button @click="alpineListeners('delete.doctors_appointment', $el)"
        :data-id="appointment.id">
    Отмени
</button>

<!-- POST заявка от форма -->
<form>
    <input name="email" type="email">
    <input name="password" type="password">
    <button @click="alpineListeners('User.login', $el)">Вход</button>
</form>
4.2 Локални методи
Ако методът НЕ започва с get/post/put/delete, системата импортира
JS модул от /components/static/:

"User.login"      → import('/components/static/user.js') → User.login(dataCollect)
"User.post_users"  → import('/components/static/user.js') → User.post_users(dataCollect)

4.3 Как се събират данните

При извикване alpineListeners() системата (DataCollect) автоматично
събира данни от 2 източника и ги ОБЕДИНЯВА в една заявка:

ИЗТОЧНИК 1: data-* атрибути на елемента
  Всеки HTML атрибут, който започва с data-, се извлича като параметър.
  Префиксът data- се премахва, остава ключът.

  <button data-id="5" data-page="2">       → { id: 5, page: 2 }
  <button :data-id="item.id">              → { id: динамична стойност }
  <div data-limit="20" data-order_by="date"> → { limit: 20, order_by: "date" }

ИЗТОЧНИК 2: Form полета (ако елементът е вътре във <form>)
  Всички input/select/textarea с name атрибут се събират автоматично.

  <form>
      <input name="email" value="test@mail.com">   → { email: "test@mail.com" }
      <select name="city_id">                       → { city_id: избраната стойност }
      <input type="checkbox" name="nzok">           → { nzok: 0 или 1 }
  </form>

И двата източника се ОБЕДИНЯВАТ. Ако бутонът е във форма И има
data-* атрибути, всичко отива заедно:

  <form>
      <input name="name" value="Иван">
      <button @click="alpineListeners('post.something', $event)"
              data-type="specialist">
          Изпрати
      </button>
  </form>
  → Изпраща: { name: "Иван", type: "specialist" }

Стойност "empty" се третира като празна и се ПРОПУСКА от заявката.

Забележка: Същата логика за data-* атрибути важи и за apiData —
елементът с apiData="get.endpoint" изпраща всичките си data-* атрибути
като параметри към GET заявката.

4.4 options-* атрибути (на формата или елемента)
options-pushurl="true"   — обновява URL параметрите при заявка
options-scroll="true"    — скролира до резултатите след отговор
options-clear="true"     — изчиства формата след успех


<form x-init="alpineListeners('get.doctors', $el)"
      keyName="doctors"
      options-pushurl="true"
      options-scroll="true">
    <select name="speciality_id" @change="alpineListeners('get.doctors', $el)">
        ...
    </select>
</form>
4.5 data-confirm за потвърждение

<button @click="alpineListeners('delete.doctors_appointment', $el)"
        data-confirm="Сигурни ли сте, че искате да отмените часа?"
        :data-id="appointment.id">
    Отмени
</button>
═══════════════════════════════════════════════════════════
5. ФОРМИ — СЪБИРАНЕ НА ДАННИ
═══════════════════════════════════════════════════════════

Всяко поле с name атрибут се включва автоматично.

Стандартни полета:

<input name="email" type="email">           → { email: "value" }
<input name="phone" type="tel">             → { phone: "value" }
<textarea name="message"></textarea>         → { message: "value" }
<select name="city_id">...</select>          → { city_id: "value" }
Checkbox:

<!-- Единичен — изпраща 0 или 1 -->
<input type="checkbox" name="nzok">          → { nzok: 1 }

<!-- Група (name[]) — масив от стойности -->
<input type="checkbox" name="services[]" value="1">
<input type="checkbox" name="services[]" value="2">
                                              → { services: ["1", "2"] }
Radio:

<input type="radio" name="type" value="patient">
<input type="radio" name="type" value="specialist">
                                              → { type: "patient" }  (само checked)
File upload:

<input type="file" name="avatar">            → File обект
<input type="file" name="documents[]" multiple> → File[] масив
Ако формата съдържа файлове, системата автоматично използва FormData
вместо JSON.

Индексирани полета:

<input name="schedule[34]" value="09:00">     → { schedule: { 34: "09:00" } }
<input name="items[1][]" value="a">           → { items: { 1: ["a"] } }
Стойност "empty":
Ако поле има стойност "empty", то се пропуска от заявката.
Използвай за default option:


<option value="empty">Избери...</option>
═══════════════════════════════════════════════════════════
6. НАВИГАЦИЯ — href()
═══════════════════════════════════════════════════════════

SPA навигация без презареждане:


<!-- Програмна навигация (препоръчително за бутони) -->
<button @click="href('/search')">Търси</button>
<button @click="href('/doctor?id=' + doctor.id)">Виж профил</button>

<!-- Стандартни линкове (за footer, static pages) — също работят -->
<a href="/bg/about">За нас</a>
href() автоматично:

Добавя езиков префикс (/bg/) ако липсва
Използва history.pushState() (без reload)
Зарежда новата страница чрез Page.load()
Нулира data.pageUrl
За динамични URL-та в Alpine:


<a :href="'/bg/doctor?id=' + doctor.id" x-text="doctor.names"></a>
<a :href="post.url" x-text="post.title"></a>
═══════════════════════════════════════════════════════════
7. МОДАЛИ — callModal()
═══════════════════════════════════════════════════════════

Отваряне/затваряне:

<button @click="callModal($el)" data-modal="my_modal">Отвори модал</button>
callModal() чете data-modal атрибута и toggle-ва стойността в data.modals.

Структура на модал:

Модалите използват CSS класове (.modal, .modal-container и т.н.)
дефинирани глобално. НЕ пиши inline Tailwind за модал структурата —
ползвай готовите класове.

<template x-if="data.modals.my_modal">
    <div class="fixed inset-0 z-[10000] flex items-center justify-center bg-black/50 backdrop-blur-sm">
        <div class="relative flex w-full h-full max-h-full flex-col overflow-hidden rounded-none bg-white shadow-xl sm:h-auto sm:max-h-[90vh] sm:rounded-xl sm:w-auto w-3xl" @mousedown.outside="callModal($el)" data-modal="my_modal">

            <!-- Header -->
            <div class="flex items-center justify-between border-b border-gray-200 px-6 py-4">
                <h3 class="text-lg font-semibold text-gray-800">Заглавие</h3>
                <button @click="callModal($el)" data-modal="my_modal" class="cursor-pointer absolute right-2.5 top-2.5 z-1000 flex h-9.5 w-9.5 items-center justify-center rounded-full bg-gray-100 text-gray-400 transition-colors hover:bg-gray-200 hover:text-gray-700">
                    <i class="fa fa-times"></i>
                </button>
            </div>

            <!-- Body -->
            <div class="flex-1 overflow-y-auto px-6 py-4">
                Съдържание тук
            </div>

            <!-- Footer -->
            <div class="flex items-center justify-end gap-2 border-t border-gray-200 px-6 py-4">
                <button @click="callModal($el)" data-modal="my_modal"
                        class="px-4 py-2 bg-gray-100 text-gray-700 rounded-lg hover:bg-gray-200 transition-colors">
                    Откажи
                </button>
                <button @click="alpineListeners('post.something', $event)"
                        class="px-4 py-2 bg-accent-500 text-white rounded-lg hover:bg-accent-600 transition-colors">
                    Потвърди
                </button>
            </div>

        </div>
    </div>
</template>



─── ПОДАВАНЕ НА ДАННИ КЪМ МОДАЛ — БЕЗ ФУНКЦИЯ ───

callModal() САМ пренася данните. Той чете ВСИЧКИ data-* атрибути на бутона и ги
записва в data.modals.<име_на_модала>. НЕ пиши функция, която да сетва state
преди отваряне, и НЕ дублирай стойности в отделен x-data обект.

ГРЕШНО:
  <button @click="openCarModal(car)">Виж</button>
  <script>
    function openCarModal(car) {
      window.data.selectedCar = car;
      callModal(document.querySelector('[data-modal=car_details]'));
    }
  </script>

ПРАВИЛНО:
  <template x-for="car in Object.values(data.cars.result)" :key="car.id">
      <button @click="callModal($el)"
              data-modal="car_details"
              :data-id="car.id"
              :data-title="car.title"
              :data-price="car.price">Виж</button>
  </template>

  <template x-if="data.modals.car_details">
      <div class="modal">
          <div class="modal-container" @mousedown.outside="callModal($el)" data-modal="car_details">
              <h3 x-text="data.modals.car_details.title"></h3>
              <span x-text="data.modals.car_details.price"></span>
          </div>
      </div>
  </template>

Цял обект наведнъж — стойността се парсва като JSON автоматично:

  <button @click="callModal($el)" data-modal="car_details"
          :data-car="JSON.stringify(car)">Виж</button>

  <span x-text="data.modals.car_details.car.specs.battery"></span>

Заявка вътре в модала с подаденото id:

  <div apiData="get.cars_auctions" :data-id="data.modals.car_details.id" keyName="modalCar">
      <span x-text="data.modals.modalCar?.title"></span>
  </div>

Какво точно влиза в data.modals.<име>:
  - всеки data-* атрибут на бутона (без префикса): data-id → .id
  - стойност, която е валиден JSON, се парсва в обект/масив автоматично
  - самият data-modal → .modal
  - ако бутонът е във <form> или <tr>, и полетата им влизат в обекта

ВАЖНО:
- Отворен е САМО ЕДИН модал в даден момент — callModal() презаписва data.modals
  наново при всяко извикване. Затварянето изчиства всички модали.
- data.modals.<име> е ОБЕКТ (истина), затова x-if="data.modals.name" работи
  и като проверка за отворен, и като източник на данните.
- Ако бутонът няма data-* атрибути освен data-modal, обектът пак е валиден —
  просто съдържа само .modal.

ВАЖНО:
- Използвай <template x-if="data.modals.name"> (НЕ x-show) за модали
- @mousedown.outside на modal-container затваря при клик извън
- data-modal атрибутът ТРЯБВА да е на елемента, който вика callModal()
- На мобилен модалът е fullscreen, на десктоп е centered с max-height 90vh
- Footer-ът НЕ е задължителен — може да го пропуснеш

Модал със стъпки (success state):

<template x-if="data.modals.my_modal">
    <div class="modal">
        <div class="modal-container modal-md" @mousedown.outside="callModal($el)" data-modal="my_modal" x-data="{ success: false }">

            <div class="modal-header">
                <h3 class="modal-title" x-text="success ? 'Готово' : 'Форма'"></h3>
                <button @click="callModal($el)" data-modal="my_modal" class="modal-close">
                    <i class="fa fa-times"></i>
                </button>
            </div>

            <div class="modal-body">
                <div x-show="!success">
                    <!-- Форма -->
                </div>
                <div x-show="success" class="text-center py-8">
                    <i class="fa-solid fa-circle-check text-green-500 text-4xl mb-4"></i>
                    <p class="text-gray-700 font-semibold">Успешно!</p>
                </div>
            </div>

            <div class="modal-footer" x-show="!success">
                <button @click="callModal($el)" data-modal="my_modal"
                        class="px-4 py-2 bg-gray-100 text-gray-700 rounded-lg">
                    Откажи
                </button>
                <button @click="alpineListeners('post.something', $event)"
                        @success="success = true"
                        class="px-4 py-2 bg-accent-500 text-white rounded-lg">
                    Изпрати
                </button>
            </div>

        </div>
    </div>
</template>

═══════════════════════════════════════════════════════════
8. ПАГИНАЦИЯ
═══════════════════════════════════════════════════════════

API-то връща pagination обект. Системата го обогатява с pagesArray.


<!-- Резултати -->
<template x-for="doctor in data.doctors.result" :key="doctor.id">
    <div x-text="doctor.names"></div>
</template>

<!-- Информация -->
<span>
    Показване на <span x-text="data.doctors.pagination.firstElementShown"></span>
    - <span x-text="data.doctors.pagination.lastElementShown"></span>
    от <span x-text="data.doctors.pagination.total_results"></span>
</span>

<!-- Навигация по страници -->
<div class="flex items-center gap-2">
    <!-- Предишна -->
    <button @click="alpineListeners('get.doctors', $event)"
            :data-page="data.doctors.pagination.current_page - 1"
            x-show="data.doctors.pagination.prevPage">
        <i class="fa-solid fa-chevron-left"></i>
    </button>

    <!-- Номера -->
    <template x-for="(page, index) in data.doctors.pagination.pagesArray" :key="index">
        <template x-if="page === '...'">
            <span>...</span>
        </template>
        <template x-if="page !== '...'">
            <button @click="alpineListeners('get.doctors', $event)"
                    :data-page="page"
                    :class="page == data.doctors.pagination.current_page
                        ? 'bg-primary-500 text-white'
                        : 'bg-white text-gray-700 hover:bg-gray-50'"
                    class="w-10 h-10 rounded-xl font-medium transition-colors"
                    x-text="page">
            </button>
        </template>
    </template>

    <!-- Следваща -->
    <button @click="alpineListeners('get.doctors', $event)"
            :data-page="data.doctors.pagination.current_page + 1"
            x-show="data.doctors.pagination.nextPage">
        <i class="fa-solid fa-chevron-right"></i>
    </button>
</div>
═══════════════════════════════════════════════════════════
9. УСЛОВЕН РЕНДЕР И ЦИКЛИ (Alpine.js)
═══════════════════════════════════════════════════════════

x-if (условно показване с DOM премахване):

<template x-if="data.user.logged_in == 1">
    <div>Здравей, <span x-text="data.user.names"></span></div>
</template>

<template x-if="doctor.video_url">
    <iframe :src="doctor.video_url"></iframe>
</template>
x-show (CSS toggle, елементът остава в DOM):

<div x-show="searchType === 'specialist'" x-transition>
    <!-- съдържание -->
</div>
x-for (цикъл):

<!-- Масив -->
<template x-for="item in data.doctors.result" :key="item.id">
    <div x-text="item.names"></div>
</template>

<!-- Обект → масив -->
<template x-for="city in Object.values(data.cities).reverse()" :key="city.id">
    <option :value="city.id" x-text="city.title"></option>
</template>

<!-- С филтриране -->
<template x-for="city in Object.values(data.cities).filter(
    obj => filterRegion === 'empty' || obj.region_id == filterRegion
).reverse()">
    <option :value="city.id" x-text="city.title"></option>
</template>

<!-- С лимит -->
<template x-for="(city, index) in Object.values(data.cities).reverse().slice(0, 6)" :key="city.id">
    ...
</template>
x-data (локално състояние):

<div x-data="{ activeTab: 'info', showMore: false }">
    <button @click="activeTab = 'info'" :class="activeTab === 'info' ? 'active' : ''">Info</button>
    <button @click="activeTab = 'reviews'" :class="activeTab === 'reviews' ? 'active' : ''">Reviews</button>

    <div x-show="activeTab === 'info'">...</div>
    <div x-show="activeTab === 'reviews'">...</div>
</div>
x-text и x-html:

<span x-text="doctor.names"></span>           <!-- escaped text -->
<div x-html="post.description"></div>          <!-- raw HTML -->
Binding с : (shorthand за x-bind):

<img :src="doctor.avatar.url" :alt="doctor.names">
<a :href="'/bg/doctor?id=' + doctor.id">
<div :class="isActive ? 'bg-primary-500' : 'bg-gray-100'">
<input :value="data.pageUrl.name || ''">
<button :disabled="!formValid">
Events:

@click="action()"
@click.prevent="action()"           <!-- preventDefault -->
@click.stop="action()"              <!-- stopPropagation -->
@click.away="close()"               <!-- клик извън елемента -->
@change="alpineListeners(...)"       <!-- select/checkbox промяна -->
@input.debounce.500="search()"      <!-- текстово поле с забавяне -->
@submit.prevent="submit()"          <!-- форма -->
@keydown.enter="submit()"           <!-- Enter клавиш -->
Transitions:

<!-- Прост fade -->
<div x-show="open" x-transition>

<!-- Custom transition -->
<div x-show="open"
     x-transition:enter="transition ease-out duration-200"
     x-transition:enter-start="opacity-0 -translate-y-2"
     x-transition:enter-end="opacity-100 translate-y-0"
     x-transition:leave="transition ease-in duration-150"
     x-transition:leave-start="opacity-100"
     x-transition:leave-end="opacity-0">

<!-- Accordion collapse -->
<div x-show="openFaq === 1" x-collapse>
x-cloak (скрий до init):

<div x-show="condition" x-cloak>
    <!-- Няма да мигне при зареждане -->
</div>
═══════════════════════════════════════════════════════════
10. HELPER ФУНКЦИИ
═══════════════════════════════════════════════════════════

Глобален обект `Helpers` с utility функции. По-долу са САМО тези,
които се използват редовно в HTML шаблоните. Останалите са вътрешни
за framework-а и не се викат директно.

--- ФОРМАТИРАНЕ НА ДАТИ ---

Helpers.formatDate(dateString, format, full)
  Форматира дата за показване.

  format варианти:
    'long'      → "20 август 2025 г."  (default)
    'bg'/'dot'  → "20.08.2025"
    'slashDMY'  → "20/08/2025"
    'slashYMD'  → "2025/08/20"
    null        → "2025-08-20" (ISO)

  full = true добавя час: "20.08.2025 14:30:00"

  Примери в HTML:
    <span x-text="Helpers.formatDate(post.date_publish)"></span>
    <span x-text="Helpers.formatDate(appointment.date, 'bg')"></span>
    <span x-text="Helpers.formatDate(item.created_at, 'dot', true)"></span>

Helpers.getDaysOfMonth(year, month)
  Връща масив с дните на месеца: [{ day: 1, name: "Пн", dayIndex: 1 }, ...]
  Имената са на български ако LANG == 'bg'.

  Пример:
    <template x-for="d in Helpers.getDaysOfMonth(2026, 3)">
        <span x-text="d.name + ' ' + d.day"></span>
    </template>

Helpers.getMonthTitleByIndex(monthIndex)
  Връща името на месеца (1-12): "Януари", "Февруари" и т.н.
    <span x-text="Helpers.getMonthTitleByIndex(3)"></span>  → "Март"

Helpers.getDayTitleByIndex(dayIndex)
  Връща името на деня (0=Неделя, 1=Понеделник ... 6=Събота):
    <span x-text="Helpers.getDayTitleByIndex(1)"></span>  → "Понеделник"

--- PLACEHOLDER КАРТИНКИ ---

Helpers.image(type)
  Връща URL на placeholder картинка.

  Типове:
    'user'    → /static/images/user.webp
    'product' → /static/images/product.webp
    '1920'    → /static/images/1920x1080.webp
    '1024'    → /static/images/1024x768.webp
    '800'     → /static/images/800x600.webp
    '640'     → /static/images/640x450.webp

  Пример:
    <img :src="doctor.avatar?.url_10x10 || Helpers.image('user')">

--- НОТИФИКАЦИИ ---

Helpers.show_toast_msg(message, type)
  Показва toast нотификация горе вдясно. Изчезва автоматично.
  type: 'success' или 'error'

  Пример (рядко се вика ръчно — framework-ът го прави автоматично):
    Helpers.show_toast_msg('Записът е запазен', 'success')

--- ЦВЕТОВА ПАЛИТРА ---

Helpers.randomColor(index, offset)
  Връща име на цвят от палитра: 'brand', 'rose', 'emerald', 'amber', 'indigo'
  Циклира по index. Полезно за динамично оцветяване на списъци.

  Пример:
    <div :class="'bg-' + Helpers.randomColor(index) + '-100'">
        <span :class="'text-' + Helpers.randomColor(index) + '-600'" x-text="item.title"></span>
    </div>

═══════════════════════════════════════════════════════════
10.1 КАРТИНКИ — ВИНАГИ ИЗПОЛЗВАЙ 10x10 РАЗМЕР
═══════════════════════════════════════════════════════════

Системата има автоматична оптимизация на картинки чрез lazy loading.
Когато задаваш src на изображение, ВИНАГИ използвай url_10x10 варианта
(миниатюра 10x10 пиксела). Системата автоматично:

1. Зарежда 10x10 като placeholder (мигновено)
2. Детектира елемента чрез IntersectionObserver
3. Заменя URL-а с правилния размер според ширината на екрана:
   - < 640px  → 800x600
   - < 800px  → 800x600
   - < 1024px → 1024x768
   - > 1024px → пълен размер

Примери:

<!-- Аватар на потребител -->
<img :src="doctor.avatar?.url_10x10 || Helpers.image('user')" :alt="doctor.names">

<!-- Снимка на пост -->
<img :src="post.images[0].url_10x10" :alt="post.title">

<!-- Background image -->
<div :style="'background-image: url(' + item.image_10x10 + ')'"></div>

ПРАВИЛА:
- ВИНАГИ ползвай url_10x10 / image_10x10 варианта, НЕ пълния URL
- Системата сама ще подмени с правилния размер при scroll
- Работи за <img> тагове и за background-image
- Ако обектът няма 10x10 вариант, ползвай Helpers.image('type') като fallback

═══════════════════════════════════════════════════════════
11. ФИЛТРИРАНЕ НА СВЪРЗАНИ ДАННИ
═══════════════════════════════════════════════════════════

Каскадни dropdown-и (регион → град):


<div x-data="{ filterRegion: 'empty' }">

    <!-- Регион -->
    <div apiData="get.regions">
        <select name="region_id" @change="filterRegion = $el.value; alpineListeners('get.doctors', $el)">
            <option value="empty">Всички региони</option>
            <template x-for="region in Object.values(data.regions).reverse()">
                <option :value="region.id" x-text="region.title"></option>
            </template>
        </select>
    </div>

    <!-- Град (филтриран по регион) -->
    <div apiData="get.cities">
        <select name="city_id" @change="alpineListeners('get.doctors', $el)">
            <option value="empty">Всички градове</option>
            <template x-for="city in Object.values(data.cities).filter(
                obj => filterRegion === 'empty' || obj.region_id == filterRegion
            ).reverse()">
                <option :value="city.id" x-text="city.title"></option>
            </template>
        </select>
    </div>
</div>

─── ЗАВИСИМИ (ПОСЛЕДОВАТЕЛНИ) ЗАЯВКИ — template x-if ───

Горният пример пуска двете заявки ЕДНОВРЕМЕННО и филтрира наготово. Това е
правилното решение, когато втората заявка не зависи от резултата на първата.

Когато обаче втората заявка се нуждае от стойност от първата, или изборът
в нея трябва да се предселектира според първата — ЧАКАЙ първата. Механизмът е
<template x-if="data.<ключ>"> около елемента с apiData на втората. apiData се
задейства при init на елемента, а template x-if създава елемента чак когато
условието стане истина — тоест след като първата заявка е върнала.

НЕ пиши async функция, Promise chain, setTimeout или ръчен fetch за това.

ГРЕШНО:
  <div x-init="loadCountriesThenCities()">
  <script>
    async function loadCountriesThenCities() {
      await alpineListeners('get.countries', $el);
      await alpineListeners('get.cities', $el);
    }
  </script>

ПРАВИЛНО:
  <div apiData="get.countries">

      <select name="country_id" @change="alpineListeners('get.cities', $el)"
              keyName="cities">
          <template x-for="c in Object.values(data.countries)" :key="c.id">
              <option :value="c.id"
                      :selected="c.id == data.user.country_id"
                      x-text="c.title"></option>
          </template>
      </select>

      <!-- Градовете тръгват ЧАК след като държавите са налични -->
      <template x-if="data.countries">
          <div apiData="get.cities" :data-country_id="data.user.country_id">
              <select name="city_id">
                  <template x-for="city in Object.values(data.cities)" :key="city.id">
                      <option :value="city.id"
                              :selected="city.id == data.user.city_id"
                              x-text="city.title"></option>
                  </template>
              </select>
          </div>
      </template>
  </div>

Верига от три нива — всяко ниво чака предното:

  <div apiData="get.manufacturers">
      <template x-if="data.manufacturers">
          <div apiData="get.cars_models" :data-manufacturer_id="data.pageUrl.manufacturer_id">
              <template x-if="data.cars_models">
                  <div apiData="get.cars_variants" :data-model_id="data.pageUrl.model_id">
                      ...
                  </div>
              </template>
          </div>
      </template>
  </div>

Правила:
- Условието е върху ключа на ПЪРВАТА заявка (data.countries), не върху втората.
- Ползвай template x-if, НЕ x-show — x-show само скрива, елементът се създава
  веднага и apiData тръгва предварително, тоест изчакването не се случва.
- Ако втората заявка зависи и от конкретна стойност, провери и нея:
  x-if="data.countries && data.user.country_id"
- Ако само показваш (без втора заявка), пак ползвай x-if, за да не гърми
  x-for върху undefined: <template x-if="data.cities">
- За единични стойности стига optional chaining: x-text="data.cities?.[0]?.title"

═══════════════════════════════════════════════════════════
12. x-init ЗА НАЧАЛНО ЗАРЕЖДАНЕ НА ФОРМА
═══════════════════════════════════════════════════════════

Когато формата трябва да зареди данни при рендер:


<form x-init="alpineListeners('get.doctors', $el)"
      keyName="doctors"
      data-limit="20"
      options-pushurl="true">
    <!-- филтри -->
</form>
x-init се изпълнява веднъж при създаване на елемента.
Комбинирай с apiData за допълнителни данни (специалности, градове).

═══════════════════════════════════════════════════════════
13. АВТЕНТИКАЦИЯ — УСЛОВНО СЪДЪРЖАНИЕ
═══════════════════════════════════════════════════════════


<!-- Покажи само за незалогнати -->
<template x-if="data.user.logged_in == 0">
    <button @click="href('/login')">Вход</button>
</template>

<!-- Покажи само за залогнати -->
<template x-if="data.user.logged_in == 1">
    <div>
        <img :src="data.user.avatar?.url || Helpers.image('user')">
        <span x-text="data.user.names"></span>
    </div>
</template>

<!-- Модал-блокер (задължителен login) -->
<template x-if="data.user.logged_in == 0">
    <div class="fixed inset-0 z-[100] flex items-center justify-center bg-black/60">
        <div class="bg-white rounded-3xl p-8 max-w-md text-center">
            <p>Моля, влезте в профила си</p>
            <button @click="href('/login')">Вход</button>
        </div>
    </div>
</template>
═══════════════════════════════════════════════════════════
14. МОБИЛЕН ФИЛТЪР DRAWER
═══════════════════════════════════════════════════════════


<!-- Бутон за отваряне (мобилен) -->
<button @click="data.mobileFilters = true" class="lg:hidden">
    <i class="fa-solid fa-sliders"></i> Филтри
</button>

<!-- Drawer -->
<div x-show="data.mobileFilters" x-cloak class="fixed inset-0 z-50 lg:hidden">
    <div class="absolute inset-0 bg-black/50" @click="data.mobileFilters = false"></div>
    <div class="absolute inset-y-0 right-0 w-80 bg-white shadow-xl overflow-y-auto"
         x-transition:enter="transition ease-out duration-300"
         x-transition:enter-start="translate-x-full"
         x-transition:enter-end="translate-x-0"
         x-transition:leave="transition ease-in duration-200"
         x-transition:leave-start="translate-x-0"
         x-transition:leave-end="translate-x-full">

        <div class="p-6">
            <div class="flex justify-between items-center mb-6">
                <h3 class="font-bold text-lg">Филтри</h3>
                <button @click="data.mobileFilters = false">
                    <i class="fa-solid fa-xmark"></i>
                </button>
            </div>
            <!-- Същите филтри като desktop -->
        </div>
    </div>
</div>
═══════════════════════════════════════════════════════════
15. СКРИПТОВЕ И ПЛЪГИНИ
═══════════════════════════════════════════════════════════

Ако страницата има нужда от допълнителен JS (напр. геолокация):


<script src="/assets/plugin/geolocation/geolocation.js?v=2"></script>
<script>geoAutoLocate();</script>
Слагай <script> тагове САМО в края на страницата.
SPA router-ът автоматично ре-изпълнява скриптове в #main.

═══════════════════════════════════════════════════════════
16. API ENDPOINTS
═══════════════════════════════════════════════════════════

Конкретните endpoints са в отделен файл и се променят по проект.
Тук е описан само ФОРМАТЪТ на заявки и отговори.

Конвенция за именуване на методи:
  get.{resource}             → GET заявка (списък или детайл)
  post.{resource}            → POST заявка (създаване)
  put.{resource}             → PUT заявка (обновяване)
  delete.{resource}          → DELETE заявка (изтриване)
  {Module}.{function}        → Локален метод (JS файл)

API отговор формат (стандартен):
{
  result: [...],              // масив с данни
  pagination: {
    current_page: 1,
    total_pages: 5,
    results_per_page: 20,
    total_results: 100,
    pagesArray: [1, 2, '...', 5],  // добавен от framework-а
    firstElementShown: 1,
    lastElementShown: 20,
    prevPage: false,
    nextPage: true
  }
}

API отговор формат (при POST/PUT/DELETE):
{
  status: 1,                  // 1 = успех, 0 = грешка
  obj: { ... },               // създаденият/обновеният обект
  msg: "Съобщение"            // toast съобщение
}

API отговор формат (при грешка):
{
  status: 0,
  errors: {
    "name=email": "Невалиден имейл",    // field-level грешка
    "name=phone": "Задължително поле"
  },
  msg: "Грешка при обработка"            // toast съобщение
}

═══════════════════════════════════════════════════════════
17. @success / @error CALLBACKS
═══════════════════════════════════════════════════════════

След всяко alpineListeners() повикване, framework-ът dispatch-ва
CustomEvent на СЪЩИЯ елемент, който е извикал заявката:

  - При status === 1 (успех) → dispatch-ва "success" event
  - При status === 0 (грешка) → dispatch-ва "error" event

Event detail съдържа:
  $event.detail.response   — пълният отговор от API-то
  $event.detail.response.obj — обработеният обект (данните)
  $event.detail.status     — 0 или 1

Слушай с Alpine @success / @error директно на елемента:

### Пример 1: Redirect след успешно създаване
Записване на час → при успех навигирай към appointment страницата:

<button @click="alpineListeners('post.doctors_appointment', $event)"
        @success="href(`/appointment?id=${$event.detail.response.obj.id}`)">
    Потвърди
</button>

### Пример 2: Промяна на UI state след успех
Отмяна на час → при успех смени флаг и презареди данните:

<button @click="alpineListeners('delete.doctors_appointment', $event)"
        @success="cancelled = true; doctors_appointments()">
    Откажи часа
</button>

### Пример 3: Запази бележка → презареди списъка и затвори формата
<button @click="alpineListeners('post.doctors_appointments_notes', $event)"
        keyName="dummy"
        @success="get_doctors_appointments(); editing = false">
    Запази
</button>

### Пример 4: Push нов обект в масив от отговора
<button @click="alpineListeners('post.doctors_cabinets', $event)"
        @success="data.my_doctor.cabinet_ids.push($event.detail.response.obj); data.showAddOffice = false">
    Запази
</button>

### Пример 5: Обнови съществуващ обект в масив
<button @click="alpineListeners('post.doctors_cabinets', $event)"
        @success="data.my_doctor.cabinet_ids[index] = $event.detail.response.obj; editingOffice = ''">
    Запази
</button>

### Пример 6: File upload → презареди при успех
<input @change="loading = true; await alpineListeners('post.doctors_record_files', $event); loading = false;"
       @success="get_record()"
       keyName="my_doctor"
       name="files" type="file" multiple>

### Пример 7: Комбинирани действия — затвори модал + презареди
<button @click="alpineListeners('post.doctors_appointment', $event)"
        @success="get_doctors_appointments(); get_openSlots(); callModal($el)">
    Добави
</button>

### Пример 8: Използване на @error
<button @click="alpineListeners('post.something', $event)"
        @success="showSuccess = true"
        @error="showError = true; errorMsg = $event.detail.response.msg">
    Изпрати
</button>

ВАЖНИ ПРАВИЛА за @success / @error:

1. Поставяй @success на СЪЩИЯ елемент, на който е @click с alpineListeners()
2. Използвай $event (НЕ $el) като втори аргумент на alpineListeners когато
   ползваш @success/@error, защото event-ът се dispatch-ва на event.target
3. $event.detail.response.obj съдържа обработените данни от отговора
4. Можеш да chain-ваш множество действия с ; (точка и запетая)
5. keyName="dummy" се ползва когато не искаш отговорът да презапише
   данни в data обекта — просто го записва в data.dummy
6. @error НЕ е задължителен — framework-ът автоматично показва
   грешките като toast съобщения и field-level errors

═══════════════════════════════════════════════════════════
18. ГРЕШКИ И ВАЛИДАЦИЯ
═══════════════════════════════════════════════════════════

Грешките идват от API-то в 2 формата:

По полета — показват се до съответния input:
{ "name=email": "Невалиден имейл" }

Toast съобщения — показват се горе вдясно:
Системата автоматично ги обработва.

НЕ добавяй ръчна валидация в HTML — използвай native HTML5
атрибути (required, type="email", minlength) и остави
сървърната валидация да се погрижи за останалото.

═══════════════════════════════════════════════════════════
19. CUSTOM DIRECTIVES
═══════════════════════════════════════════════════════════

x-amount="price"
→ Форматира число като валута (2 дец. знака + символ от CURRENCY константата)

<span x-amount="doctor.price"></span>

ВАЖНО: Валутата по подразбиране е ЕВРО (€), НЕ лева.
Ако се изписват цени в текст (не чрез x-amount), използвай € или "EUR".
Пример: "Цена: 50 €" или "от 30 €/час"
Ако проектът изисква друга валута, тя ще е указана изрично.
x-tooltip.position="text"
→ Показва tooltip при hover (позиции: top, bottom, left, right)


<button x-tooltip.top="'Натисни за повече информация'">?</button>
═══════════════════════════════════════════════════════════
20. DATE PICKER
═══════════════════════════════════════════════════════════

Използва flatpickr чрез DateHelper:


<!-- Единична дата -->
<div x-data="DateHelper.date()">
    <input type="text" name="date" placeholder="Избери дата">
</div>

<!-- Период (range) -->
<div x-data="DateHelper.dateRange()">
    <input type="hidden" name="date_from">
    <input type="hidden" name="date_to">
    <input type="text" placeholder="От — До">
</div>

<!-- Дата и час -->
<div x-data="DateHelper.dateTime()">
    <input type="text" name="datetime">
</div>

<!-- Само час -->
<div x-data="DateHelper.time()">
    <input type="text" name="time">
</div>
═══════════════════════════════════════════════════════════
21. СТРУКТУРА НА СЕКЦИИ (is-section)
═══════════════════════════════════════════════════════════

Всяка основна секция на страницата ТРЯБВА да използва следната
обвивка. Това е нужно за визуалния редактор (editor) на платформата.

Базова структура:

<div class="is-section is-section-auto section">
    <div class="is-overlay">
        <div class="is-overlay-bg"></div>
    </div>
    <div class="!container is-container v2">
        <!-- Съдържание тук -->
    </div>
</div>

Елементи:
  is-section          — обвивка на секцията (задължителен)
  is-section-auto     — автоматична височина
  section             — допълнителен CSS клас
  is-overlay          — слой за фон/цвят
  is-overlay-bg       — ТУК се слага фоновият цвят (НЕ на is-section)
  !container          — контейнер със стандартен max-width
  is-container v2     — маркер за редактора

ФОНОВ ЦВЯТ — слага се като клас на is-overlay-bg:

<!-- Бял фон (по подразбиране) -->
<div class="is-overlay-bg"></div>

<!-- Цветен фон -->
<div class="is-overlay-bg bg-primary-600"></div>

<!-- Градиент -->
<div class="is-overlay-bg bg-gradient-to-br from-primary-500 to-primary-700"></div>

<!-- Сив фон -->
<div class="is-overlay-bg bg-gray-50"></div>

<!-- Тъмен фон -->
<div class="is-overlay-bg bg-gray-900"></div>

КЛАС lock — добавя се когато секцията съдържа ДИНАМИЧНО съдържание
(x-for, x-if, apiData и т.н.). Това указва на редактора да не пипа секцията.

<!-- Статична секция (без lock) -->
<div class="is-section is-section-auto section">
    ...
</div>

<!-- Динамична секция (с lock) -->
<div class="is-section is-section-auto section lock">
    ...
    <template x-for="item in data.items">...</template>
    ...
</div>

OVERRIDE НА CONTAINER MAX-WIDTH:
Контейнерът (.container) има стандартен max-width. Когато секцията
трябва да е по-широка, override-ни с Tailwind:

<!-- Стандартен контейнер -->
<div class="!container is-container v2">

<!-- По-широк контейнер -->
<div class="!container is-container v2 !max-w-7xl">

<!-- Пълна ширина (без ограничение) -->
<div class="!container is-container v2 !max-w-full">

ПЪЛЕН ПРИМЕР — Hero секция с цвят:

<div class="is-section is-section-auto section">
    <div class="is-overlay">
        <div class="is-overlay-bg bg-gradient-to-br from-primary-500 to-primary-700"></div>
    </div>
    <div class="!container is-container v2 py-16 lg:py-24">
        <div class="max-w-3xl text-white">
            <h1 class="text-3xl lg:text-5xl font-bold mb-6">Заглавие</h1>
            <p class="text-xl text-primary-100">Описание</p>
        </div>
    </div>
</div>

ПЪЛЕН ПРИМЕР — Секция с динамични данни (lock):

<div class="is-section is-section-auto section lock" apiData="get.blogPosts" data-limit="3" keyName="latestPosts">
    <div class="is-overlay">
        <div class="is-overlay-bg bg-gray-50"></div>
    </div>
    <div class="!container is-container v2 py-16">
        <h2 class="text-3xl font-bold text-gray-900 mb-8">Последни статии</h2>
        <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
            <template x-for="post in data.latestPosts.result" :key="post.id">
                <div x-text="post.title"></div>
            </template>
        </div>
    </div>
</div>

КОГА НЕ се използва is-section:
  - Модали (callModal) — НЕ
  - Динамично съдържание вътре в секция (вложени x-for) — НЕ
  - Мобилен drawer (mobileFilters) — НЕ
  - Компоненти вътре в друга секция — НЕ

Използвай is-section САМО за основните top-level блокове на страницата.

═══════════════════════════════════════════════════════════
22. ПРАВИЛА ЗА ПИСАНЕ НА КОД
═══════════════════════════════════════════════════════════

НИКОГА не пиши JavaScript логика в <script> тагове освен ако няма друг вариант — ползвай Alpine директиви (x-data, x-show, x-for, @click) и глобалните функции
Данните ВИНАГИ идват от data обекта — не hardcode-вай

─── ЧЕТИ ДИРЕКТНО ОТ data — НЕ ПРАВИ ФУНКЦИИ ЗА ДРЕБНИ НЕЩА ───

НЕ създавай helper функция, getter, computed property или x-data метод само за
да прочетеш или сметнеш стойност, която вече е в data. Пиши израза директно в
Alpine директивата. Alpine expressions поддържат целия JS синтаксис.

ГРЕШНО:
  <div x-data="{ getPrice() { return window.data.product.price } }">
    <span x-text="getPrice()"></span>
  </div>

  <script>
    function productTotal() {
      return window.data.product.price * window.data.product.qty;
    }
  </script>
  <span x-text="productTotal()"></span>

ПРАВИЛНО:
  <span x-text="data.product.price"></span>
  <span x-text="data.product.price * data.product.qty"></span>

Директно в израза правѝ и следното — без функции:

  Прости сметки       x-text="data.cart.subtotal + data.cart.shipping"
  Проценти            x-text="Math.round(data.product.discount / data.product.price * 100)"
  Тернарни условия    x-text="data.user.logged_in ? data.user.names : 'Гост'"
  Fallback стойности  x-text="data.product.title || '—'"
  Дълбок достъп       x-text="data.product?.brand?.name"
  Броене              x-text="Object.values(data.cart.items).length"
  Форматиране         x-text="Helpers.price(data.product.price)"
  Условен клас        :class="data.product.stock > 0 ? 'text-green-600' : 'text-red-600'"
  Условен рендер      x-show="data.product.discount > 0"

Функция се пише САМО когато е налице поне едно от:
  - логиката е над ~3 реда или има цикъл / няколко if-а
  - точно същата логика се повтаря на 3+ места в проекта
  - нужен е side effect (API заявка, промяна на state, навигация)
  - вече съществува подходящ Helpers.* метод — тогава го ползвай, не пиши нов

Локален x-data state (`x-data="{ activeTab: 'info' }"`) е за UI състояние, което
НЕ идва от data — табове, отворен/затворен, hover. Не дублирай в него стойности
от data обекта.
За навигация ползвай href() или :href binding
За API повиквания ползвай САМО alpineListeners()
За модали ползвай САМО callModal() с data-modal атрибут
x-show за toggle (остава в DOM), x-if за условно рендериране (махa от DOM)
x-cloak на всичко с x-show за да не мига при load
Стойност "empty" = празна стойност за select options
Object.values() при итерация на обекти от API
$el = текущият елемент, $event = текущият event — ползвай ги в alpineListeners()

═══════════════════════════════════════════════════════════
23. SEO ЗА ДИНАМИЧНИ СТРАНИЦИ (/product/6-bmw-x5)
═══════════════════════════════════════════════════════════

Страници с ID в URL-а (product, post, auction, specification, категории)
получават meta таговете си ДИНАМИЧНО от API-то. НЕ пиши SEO в .html файла на
страницата и НЕ слагай <title> или <meta> тагове в него.

── URL структура ──

  /{lang}/{slug}/{target_id}-{текст}

  /en/product/6-bmw-x5   → lang=en, slug=product, target_id=6
  /bg/auction/42         → lang=bg, slug=auction, target_id=42

target_id се вади с parseInt от частта преди първото тире, така че текстът
след тирето е чисто за четимост и може да е какъвто и да е.
В страницата ID-то е достъпно като data.corePage.target_id.

── ДВА МЕХАНИЗМА, КОИТО СЕ ПИПАТ ЗАЕДНО ──

  КЛИЕНТ  PageClass.updateMeta()  →  components/core/classes/page.js
          Работи при SPA навигация (клик, href()). Извиква API-то и презаписва
          meta таговете в живия DOM.

  СЪРВЪР  Page::load_page()       →  core/classes/class.page.php
          Работи при директно отваряне / refresh. Рендерира meta таговете още в
          HTML-а — това е, което виждат Google, Facebook и другите ботове.

ЗАДЪЛЖИТЕЛНО: двата файла са ОГЛЕДАЛО един на друг. Промяна само в единия
чупи или SPA навигацията, или индексирането. Винаги редактирай и двата.
Същото важи за константата на марката: AC_SEO_BRAND (page.js) = Page::SEO_BRAND
(class.page.php).

── КАК СЕ ДОБАВЯ НОВА ДИНАМИЧНА СТРАНИЦА ──

1) В page.js → updateMeta(), нов клон в if/else if веригата по this.slug:

   } else if (this.slug === 'dealer' && this.target_id > 0) {
       await api.get('dealers/' + this.target_id);
       let response = api.response;

       this.seo_title = response.seo_title?.trim() ? response.seo_title : response.title;
       this.seo_description = (response.seo_description?.trim()
           ? response.seo_description
           : `${response.title} — ${AC_SEO_BRAND}`).substring(0, 200);
       this.seo_image = response.images?.[0]?.url ?? LOGO_URL;
       this.seo_tags = response.seo_tags || this.seo_title;
   }

2) В class.page.php → load_page(), същият клон с PHP синтаксис:

   if($this->slug == 'dealer' && $this->target_id > 0){
       $target = Api::cache($cache)->id($this->target_id)
                    ->data(['resolution' => '10x10'])->get()->dealers();
       if(!empty($target)){
           $this->seo_title = !empty(trim($target['seo_title'] ?? ''))
               ? $target['seo_title'] : $target['title'];
           $this->seo_description = mb_substr(trim(!empty(trim($target['seo_description'] ?? ''))
               ? $target['seo_description']
               : $target['title'].' — '.self::SEO_BRAND), 0, 200);
           $this->seo_image = $target['images'][0]['url'] ?? $core->web['logo'];
           $this->seo_tags = $target['seo_tags'] ?? $this->seo_title;
       }
   }

── ПРАВИЛА ──

- Приоритет на стойностите ВИНАГИ в този ред:
    1. seo_* полето от API-то (ако админът го е попълнил)
    2. смислено генерирано от данните (title, марка, модел, година)
    3. fallback — заглавието на страницата / LOGO_URL
  Никога не презаписвай ръчно попълнено seo_ поле с генерирано.

- seo_description се реже на 200 символа. В PHP задължително mb_substr
  (не substr — чупи кирилица). HTML се маха със strip_tags / .replace(/<[^>]*>/g,'').

- seo_image: винаги абсолютен URL, с fallback LOGO_URL / $core->web['logo'].

- Ако API-то не върне обект (изтрит запис), НЕ пипай seo_* полетата —
  остави текстовете от админа. Празните се попълват от общия fallback
  в края на функцията.

- Записът в DOM става по CSS клас, не по име на таг — meta елементите носят
  класове seo_title / seo_description / seo_image / seo_tags, плюс
  document.title. Не добавяй нови meta тагове в страницата; ако трябва ново
  поле, добави го в шаблона на header-а със съответния клас.

- Ендпойнтите връщат ту плосък обект, ту {result:[…]} — нормализирай
  (виж pickModelSeo / Page::pick_model), не приемай една от двете форми.
