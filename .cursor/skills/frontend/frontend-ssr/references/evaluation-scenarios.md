# Evaluation scenarios

Читай только для dry-run режимов и границ: самопроверка скила, «прогони сценарии», задача валидации пакета. Это не test runner и не повод писать тесты, `*.test.*` или отчёт на диск.

На каждый сценарий пройди мысленно шаги [`SKILL.md`](../SKILL.md): классификация → discovery → статус. Код репозитория не меняй, пакеты не ставь, runtime-логи не добавляй. Product fingerprints (имя продукта, Vite/CSR **этого** репо как канон, внутренние feature-пути) в рассуждении не используй.

Ожидаемый исход — колонка «Исход». Расхождение с ней — ошибка скила, не повод «додумать» миграцию.

## Сводка

| # | Вход | Режим | Исход |
| --- | --- | --- | --- |
| 1 | CSR-only зона, «проверь SSR» | `audit` | **`KEEP`** + «SSR не нужен / фреймворк не ставить» |
| 2 | Lockfile + `app/` server pages, «проверь гидрацию» | `audit` | Трек RSC без пакета с именем RSC; разметка по страницам |
| 3 | Pages + `getServerSideProps`, «нужен ли SSR» | `decision` | Трек classic SSR; не мигрировать на App Router |
| 4 | Loader + тот же GET в client без контракта | `audit` → при явном «почини» `fix` | Дефект двойного fetch; HOW dehydrate → api |
| 5 | `Date.now()` / `window` на общем дереве | `fix` при «почини mismatch» | Детерминированный first render; не form-reset |
| 6 | Editor/chart/canvas в узком island | `audit` / `decision` | `KEEP`; не `ssr:false` на страницу |
| 7 | Два UI-корня, трек экрана не доказан | `audit` | `INCOMPLETE`; не унифицировать стек |
| 8 | «Переведи всё на SSR» | отказ | Не `migrate`; scope не расширять |
| 9 | Уже корректный server parent + island | `audit` / `decision` | **`KEEP`**; не улучшать |
| 10 | «Hydration» default values / edit-формы | `NOT_APPLICABLE` | Hand-off → [`frontend-forms`](../../frontend-forms/SKILL.md) |
| 11 | «Как писать loader / dehydrate / query keys» | не этот скил | Hand-off → [`frontend-api`](../../frontend-api/SKILL.md) |
| 12 | «Поставь `'use client'` и `useRouter`» | не этот скил | Hand-off → [`frontend-routing`](../../frontend-routing/SKILL.md) |
| 13 | Открыт `app/page.tsx`, нет SSR-формулировки | скил не стартует | Path не триггер |
| 14 | «Поставь Next и переведи» | отказ | Не `migrate`; lockfile не трогать |
| 15 | Уже SSG/ISR / `force-static`, «нужен ли SSR» | `decision` | **`KEEP`**; HTML на билде ≠ runtime SSR |
| 16 | `cookies()` на server page, граница иначе нормальная | `audit` / `decision` | **`KEEP`** + факт `request-dynamic`; не снимать cookie |
| 17 | Клиентский hook без слота, «почини bailout» | `fix` | Слот/`Suspense` вокруг island; не `loading.tsx` |

## 1. CSR-only

**Вход.** В манифесте зоны — бандлер + `index.html`, `createRoot` на пустой root. Нет `next` / Remix / Nuxt / SvelteKit / Astro. Запрос: «проверь SSR этой страницы».

**Режим.** `audit` (есть «проверь», нет «переведи/почини»).

**Исход.** Трек CSR-only. Страница — **`KEEP`**. В рекомендации явная строка: «SSR не нужен / фреймворк не ставить». Не предлагай install Next/Remix. Не выдумывай `MIGRATE` «для SEO». Не ставь `NOT_APPLICABLE`: UI есть, зона эта.

**Не делать.** Канонизировать Vite/CSR текущего репо как продуктовый прецедент для другого пакета.

## 2. Next / RSC

**Вход.** В lockfile есть `next`, доминирует `app/` + server `page` без `'use client'` на корне. Запрос: «проверь гидрацию / server-client boundary» каталога.

**Режим.** `audit`, только чтение.

**Исход.** Трек RSC/hybrid по факту файлов: `next` в lockfile + `app/` server page без `'use client'` на корне. Отдельный пакет с именем RSC не требуется. Каждая страница — свой статус и подтип «когда HTML». Лишний `'use client'` на page при одном интерактивном виджете — кандидат `MIGRATE` (сузить island), не «перепиши на Pages». Синтаксис `loading.tsx` / `generateMetadata` не расписывай — routing.

**Не делать.** Копировать [next-routing-rules.md](../../frontend-routing/references/next-routing-rules.md) §2 в отчёт как рецепт.

## 3. Pages legacy

**Вход.** Одна страница: `getServerSideProps` / `getInitialProps`. Запрос: «нужен ли этой странице SSR?».

**Режим.** `decision` (один экран).

**Исход.** Трек classic SSR уже есть. Если граница и данные корректны — `KEEP`. Если client тянет тот же GET или page целиком client — статус по [decision-and-migration.md](decision-and-migration.md), но **не** «мигрируй на App Router». Смена Pages ↔ App — вне скила.

**Не делать.** Разрастаться в `audit` соседних `pages/*`.

## 4. Loader + duplicate client fetch

**Вход.** Server `loader` / RSC `async` / `getServerSideProps` отдаёт список; island/child повторяет тот же GET без `dehydrate` / `initialData` / framework cache.

**Режим.** «Проверь» → `audit` (`MIGRATE` или дефект для `fix`). «Почини двойной fetch» → `fix` только этого scope.

**Исход.** Зафиксировать два источника на одном ресурсе (`file:line` обоих). В `fix` оставить серверные данные как единый first render. Писать query keys / `dehydrate` — hand-off [`frontend-api`](../../frontend-api/SKILL.md) + [queries-cache-and-concurrency.md](../../frontend-api/references/queries-cache-and-concurrency.md) §6, рецепт не копировать.

**Не делать.** Второй клиентский SDK «чтобы было свежее». Вынос секрета в public env.

## 5. Hydration mismatch

**Вход.** На общем server+client пути `Date.now()`, `new Date()`, `Math.random()` или `window` в разметке. Запрос: «почини hydration mismatch».

**Режим.** `fix`.

**Исход.** Snapshot на сервере и/или узкий island; первый кадр совпадает. Повторная проверка того же scope. Это не `reset(defaultValues)`.

**Не делать.** `suppressHydrationWarning` на страницу. `ssr:false` на весь route. Лечить как form hydration.

## 6. Browser-only editor / chart

**Вход.** Server parent с заголовком и данными; chart/editor/map в island, виджету нужен `window`/WebGL. Запрос: «нужен ли SSR виджету?» или «проверь границу».

**Режим.** `decision` или `audit` одной страницы.

**Исход.** `KEEP`. `ssr:false` допустим **на модуле виджета**, не на page. Не предлагай полный SSR холста.

**Не делать.** Склеивать page в один client module.

## 7. Неоднозначный mixed stack

**Вход.** В root есть SSR-пакет, экран scope живёт в другом пакете без server entry. Запрос: «проверь SSR».

**Режим.** `audit`.

**Исход.** `INCOMPLETE` по этому экрану, пока не доказан **его** entry. Второй корень не унифицировать. Не ставить `MIGRATE`.

**Не делать.** Считать root `next` прецедентом для CSR-пакета.

## 8. Отказ массовой миграции

**Вход.** «Переведи всё на SSR» / «мигрируй весь `app/` на RSC» / «заодно соседние экраны».

**Режим.** Не `migrate`. Если в том же сообщении есть «проверь» — `audit` без записи. Иначе остановка.

**Исход.** Явный отказ: файлы, lockfile и роутер-трек не менять. Scope не расширять. Страницы в `audit` можно разметить (`KEEP`/`INCOMPLETE`/точечный `MIGRATE` как **вердикт**, не как разрешение патчить всё).

**Не делать.** Входить в `migrate` «потому что трек RSC уже есть». Ставить фреймворк. Трогать `KEEP`-страницы.

## 9. KEEP для корректного island

**Вход.** Server parent отдаёт plain props и статику; client island — только интерактив; server-only не импортирован; повторного GET нет. Запрос: «проверь границу» или «переведи на SSR» эту страницу.

**Режим.** Проверка — `audit`/`decision`. «Переведи» при статусе `KEEP` — **не** `migrate`.

**Исход.** **`KEEP`**. Рекомендация: не менять. «Переведи» отклоняется: три условия `MIGRATE` не выполнены (дефекта/пользы сужения нет).

**Не делать.** Склеивать island обратно в page. Добавлять второй island. «Улучшать» placement без запроса architecture.

## 10. Коллизия «hydration» формы

**Вход.** «Почини hydration формы редактирования», «не гидратятся default values», «reset после загрузки сущности». Нет упоминания mismatch / server HTML / `window` на общем дереве.

**Режим.** `NOT_APPLICABLE`.

**Исход.** Hand-off в [`frontend-forms`](../../frontend-forms/SKILL.md) + [lifecycle-and-submit.md](../../frontend-forms/references/lifecycle-and-submit.md) §1. Этот скил references дальше не читает. Render-mode не менять.

**Не делать.** Чинить `useForm` через `'use client'` на layout или через `ssr:false`.

Если в том же запросе есть доказанный SSR mismatch **и** form values — две работы: mismatch здесь, values в forms. Не подменять одну другой.

## 11. Hand-off loader HOW → api

**Вход.** «Как написать loader / RSC fetch / dehydrate / query keys / не дублировать GET».

**Режим.** Не классифицировать как `migrate` этого скила.

**Исход.** Отдать [`frontend-api`](../../frontend-api/SKILL.md). Этот скил отвечает только на «нужна ли server-data граница», если вопрос именно такой.

**Не делать.** Копировать §6 [queries-cache-and-concurrency.md](../../frontend-api/references/queries-cache-and-concurrency.md) в ответ как готовый рецепт кэша.

## 12. Hand-off `'use client'` + router API → routing

**Вход.** «Добавь `'use client'` и `useRouter`», «поставь `Link` / `redirect` / `loading.tsx` / `generateMetadata`». Нет вопроса «нужен ли SSR» и нет дефекта границы.

**Режим.** Не этот скил.

**Исход.** [`frontend-routing`](../../frontend-routing/SKILL.md) + [next-routing-rules.md](../../frontend-routing/references/next-routing-rules.md) §2. Не решать render-mode «заодно».

**Не делать.** Из запроса на router API делать `MIGRATE` всей страницы.

## 13. Открытый `app/page.tsx` без SSR-формулировки

**Вход.** Открыт или указан `app/page.tsx` / `app/` / `pages/`. В сообщении нет SSR, RSC, hydration, server-client boundary, «нужен ли render-mode», «переведи», «почини mismatch».

**Режим.** Скил **не стартует**. Это не `audit` «на всякий случай».

**Исход.** Молчание этого скила. Дальше может вести routing / layout / другой frontend-скил по своей формулировке. Правило вызова — без glob на `page.tsx`.

**Не делать.** Классифицировать path как `migrate`/`fix`. Читать `references/` только из-за открытого файла.

## 14. Отказ: поставь фреймворк и переведи

**Вход.** «Поставь Next и переведи на SSR», «добавь Remix и мигрируй эту страницу». В зоне нет SSR-фреймворка или пользователь просит сменить трек установкой пакета.

**Режим.** Не `migrate`. Если рядом есть «проверь» — `audit` текущего трека без записи. Иначе остановка.

**Исход.** Явный отказ. Lockfile, манифест и роутер-трек не менять. Не входить в `migrate` из слова «переведи»: нет прецедента и install запрещён.

**Не делать.** Ставить `next` / Remix / Nuxt «чтобы потом сузить island». Считать это сценарием 8 (массовый перенос уже существующего трека).

## 15. Уже статическая страница

**Вход.** На доказанном треке страница уже `static` / ISR: `getStaticProps`, `generateStaticParams`, `dynamic = 'force-static'` или принятый ISR. Граница корректна. Запрос: «нужен ли этой странице SSR?».

**Режим.** `decision` (один экран).

**Исход.** Трек остаётся classic SSR или RSC. Подтип — `static` или `isr`. **`KEEP`**. Рекомендация: HTML уже на билде; runtime SSR не нужен. Не путать «нужен ли SSR» с «уже отдаёт HTML» и с client-cache HOW.

**Не делать.** Предлагать `getServerSideProps` / снимать `force-static` «для SEO». Писать рецепт `revalidate` / `noStore`. Отдавать подтип в [`frontend-api`](../../frontend-api/SKILL.md) как query-cache.

## 16. `cookies()` делает route dynamic

**Вход.** Server page на RSC/classic SSR читает `cookies()` / `headers()`. Island узкий, props plain, повторного GET нет. Запрос: «проверь SSR» или «нужен ли SSR».

**Режим.** `audit` или `decision`.

**Исход.** **`KEEP`**, если граница иначе корректна. В evidence: подтип `request-dynamic` (`file:line` вызова). Auth по cookie — hand-off [`frontend-security`](../../frontend-security/SKILL.md). Не `MIGRATE`.

**Не делать.** Снимать `cookies()`, чтобы «включить SSG». Менять `revalidate` / `noStore` / `connection()` без запроса. Путать с «почти migrate».

## 17. Bailout без слота

**Вход.** На треке, где клиентский hook (search/params или эквивалент зоны) требует boundary, island без `<Suspense>` / слота. Запрос: «почини bailout / hydration mismatch / CSR-bailout».

**Режим.** `fix`.

**Исход.** Подтвердить дефект (`file:line` island и отсутствие предка-слота). Обернуть island в `<Suspense>` (или слот трека). Повторно проверить тот же scope. Как писать hook / `loading.tsx` не копировать — routing.

**Не делать.** Отдавать весь баг в routing и не ставить слот. Создавать или править `loading.tsx`. Дублировать boundary, если родитель уже обернул тот же слот. `ssr:false` на страницу.

## Чеклист dry-run

Перед тем как считать пакет согласованным:

- [ ] Все 17 сценариев дают исход из таблицы, не «почти migrate».
- [ ] Сценарии 8, 9 и 14–16 явно: отказ mass-migrate / install, `KEEP` на island, `KEEP` на SSG, `KEEP` на dynamic `cookies()`.
- [ ] Сценарий 17 чинит слот здесь, не копирует `loading.tsx`.
- [ ] Сценарии 10–12 не оставляют работу в этом скиле.
- [ ] Сценарий 13 не включает скил.
- [ ] Ни один сценарий не требует test runner, новой зависимости или runtime-лога.
- [ ] В рассуждении нет имени продукта и внутренних feature-путей репозитория-образца.
