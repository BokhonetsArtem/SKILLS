---
name: frontend-ssr
description: >-
  Решает, нужен ли странице SSR/RSC/island, аудирует render-mode и точечно
  мигрирует или чинит server-client границу по фактическому стеку зоны. Use
  when SSR, RSC, hydration, server-client boundary, «проверь SSR / гидрацию»,
  «нужен ли SSR этой странице», «переведи на SSR/RSC», «почини hydration
  mismatch». Не для `'use client'` / `useRouter` / `Link` / `redirect` /
  `loading.tsx` / `generateMetadata` как синтаксиса маршрута — это
  frontend-routing. Не для HOW loader/RSC fetch, dehydrate, query keys — это
  frontend-api. Не для куда класть server-модуль vs client island / public
  API — это frontend-architecture. Не для XSS / token в бандле / `cookies()`
  как auth — это frontend-security. Не для default values / edit-гидрации
  формы — это frontend-forms. Не для skeleton / fallback как визуала — это
  frontend-layout. Не для системного a11y/WCAG — это frontend-a11y. Не ставит
  SSR-фреймворк и не мигрирует CSR/Pages/App Router без явного запроса.
disable-model-invocation: false
argument-hint: "[аудит | решение | миграция | исправить]"
---

# Frontend SSR

Решай **render-mode** страницы и точечную **server-client границу** по **фактическому стеку зоны**: нужен ли SSR/RSC/island, безопасна ли миграция, где гидрация и данные расходятся. Сначала классифицируй режим (`audit` / `decision` / `migrate` / `fix` / `NOT_APPLICABLE`), затем обнаружь render-mode зоны. Не ставь SSR-фреймворк и не переводи CSR / Pages Router / App Router «потому что файл открыт».

Инструменты зависят от режима (таблица шага 1). В `audit` и `decision` — только чтение, как [`frontend-security`](../frontend-security/SKILL.md): файлы не меняй. В `migrate` и `fix` — точечные правки в заявленном scope, без массового рефакторинга и без установки фреймворка. Runtime-логи, cookie/header values, токены и PII в чат не выводи: редактируй как `[REDACTED]`.

Правила discovery, статусов, границ и гидрации **не копируй в этот файл**. Читай только нужный файл из `references/` (пути относительно этой папки скила).

**Не этот скил.** `'use client'` / `useRouter` / `Link` / `redirect` / `loading.tsx` / `generateMetadata` как синтаксис маршрута — [`frontend-routing`](../frontend-routing/SKILL.md) + [next-routing-rules.md](../frontend-routing/references/next-routing-rules.md) §2; этот скил не копирует Next/RR рецепты. Как писать loader/RSC fetch, dehydrate, query keys, не дублировать GET — [`frontend-api`](../frontend-api/SKILL.md) + [queries-cache-and-concurrency.md](../frontend-api/references/queries-cache-and-concurrency.md) §6; здесь — нужна ли server-data граница, не HOW кэша. Куда класть server-модуль vs client island / public API — [`frontend-architecture`](../frontend-architecture/SKILL.md); `migrate` не изобретает папочную схему. XSS / token в бандле / `cookies()` как auth / cookie flags — [`frontend-security`](../frontend-security/SKILL.md) + [`aif-security-checklist`](../../aif-security-checklist/SKILL.md); здесь — serializable props и запрет server-only import в client, не source-to-sink. Default values / edit-гидрация **формы** — [`frontend-forms`](../frontend-forms/SKILL.md) + [lifecycle-and-submit.md](../frontend-forms/references/lifecycle-and-submit.md) §1; «hydration» в forms = form values, не SSR/RSC mismatch. Skeleton / fallback как визуал — [`frontend-layout`](../frontend-layout/SKILL.md); `Suspense`/streaming как render-контракт — здесь. Системный a11y/WCAG — [`frontend-a11y`](../frontend-a11y/SKILL.md). Нет клиентского UI — `references/` не читай.

Маршруты (формулировка запроса → скил):

- «проверь SSR / гидрацию / server-client boundary» без «переведи/почини» → этот скил, `audit`
- «нужен ли SSR этой странице / какой render-mode» → этот скил, `decision`
- «переведи / примени / мигрируй на SSR/RSC» → этот скил, `migrate` (только страницы со статусом `MIGRATE`)
- «почини hydration mismatch / server-only leak / двойной fetch» → этот скил, `fix`
- «переведи всё на SSR» / массовый перенос соседних экранов → отказ, не `migrate`
- «'use client' / useRouter / Link / redirect / loading / generateMetadata» → [`frontend-routing`](../frontend-routing/SKILL.md)
- «как писать loader / dehydrate / query keys / не дублировать GET» → [`frontend-api`](../frontend-api/SKILL.md)
- «куда положить server-модуль / client island / public API» → [`frontend-architecture`](../frontend-architecture/SKILL.md)
- «XSS / token в бандле / cookies() как auth» → [`frontend-security`](../frontend-security/SKILL.md)
- «default values / edit hydration формы» → [`frontend-forms`](../frontend-forms/SKILL.md)
- «skeleton / fallback как визуал» → [`frontend-layout`](../frontend-layout/SKILL.md)
- «проверь доступность / WCAG» → [`frontend-a11y`](../frontend-a11y/SKILL.md)

Routing остаётся владельцем route tree и URL-state; API — HTTP/DTO/cache HOW; architecture — placement/public API; security — source-to-sink audit; layout/forms/a11y сохраняют свои зоны. Соседи не решают render-mode «заодно».

**Матрица владения:**

| Тема | Кто ведёт | Кто не ведёт |
| --- | --- | --- |
| Нужен ли странице SSR/RSC/island; audit-статус; точечная миграция границы | этот скил | соседи не решают render-mode «заодно» |
| `'use client'` / `useRouter` / `Link` / `redirect` / `loading.tsx` / `generateMetadata` как синтаксис маршрута | [`frontend-routing`](../frontend-routing/SKILL.md) + [next-routing-rules.md](../frontend-routing/references/next-routing-rules.md) §2 | этот скил не копирует Next/RR рецепты |
| Как писать loader/RSC fetch, dehydrate, query keys, не дублировать GET | [`frontend-api`](../frontend-api/SKILL.md) + [queries-cache-and-concurrency.md](../frontend-api/references/queries-cache-and-concurrency.md) §6 | этот скил решает, нужна ли server-data граница, не HOW кэша |
| Куда класть server-модуль vs client island / public API | [`frontend-architecture`](../frontend-architecture/SKILL.md) | `migrate` не изобретает новую папочную схему; если файлы должны переехать — сначала placement |
| XSS / token в бандле / `cookies()` как auth / cookie flags | [`frontend-security`](../frontend-security/SKILL.md) + [`aif-security-checklist`](../../aif-security-checklist/SKILL.md) | этот скил — serializable props и запрет server-only import в client, не source-to-sink |
| Default values / edit-гидрация **формы** | [`frontend-forms`](../frontend-forms/SKILL.md) + [lifecycle-and-submit.md](../frontend-forms/references/lifecycle-and-submit.md) §1 | слово «hydration» в forms = form values, не SSR/RSC mismatch |
| Skeleton / fallback как визуал | [`frontend-layout`](../frontend-layout/SKILL.md) | `Suspense`/streaming boundary как render-контракт — здесь |
| Системный a11y/WCAG | [`frontend-a11y`](../frontend-a11y/SKILL.md) | не этот скил |

**Режим** (не грузи лишнее):

| Задача | Что читать |
| --- | --- |
| CSR/SSR/RSC/islands/hybrid, подтип static/isr/dynamic, stop-условия | [references/discovery-and-render-mode.md](references/discovery-and-render-mode.md) |
| Доказательства `MIGRATE`/`KEEP`/`INCOMPLETE`/`NOT_APPLICABLE`, формат audit, отличие `decision`, отказ «переведи всё» | [references/decision-and-migration.md](references/decision-and-migration.md) |
| Минимальный client island, serializable props, server-only imports, cookies/headers как render-input | [references/server-client-boundaries.md](references/server-client-boundaries.md) |
| Deterministic first render, mismatch, Suspense/streaming, запрет двойного fetch | [references/hydration-and-data.md](references/hydration-and-data.md) |
| Пары Плохо / Хорошо (нейтральный стек, без product path) | [references/examples-good-and-bad.md](references/examples-good-and-bad.md) |
| Dry-run режимов и границ без test runner | [references/evaluation-scenarios.md](references/evaluation-scenarios.md) |

Next/RR синтаксис, loader HOW и form-values hydration **не копируй** из соседних references: отсылай к routing / api / forms.

## Шаг 1. Классифицируй запрос и scope

Опора: сообщение пользователя, `$ARGUMENTS` (если скил вызван с аргументом), открытые клиентские файлы. В обычном чате `$ARGUMENTS` может быть пустым — это не стоп. Сначала классифицируй **режим** по намерению (сообщение и `$ARGUMENTS`), затем path как scope. Path, открытый `page.tsx` / `app/` / `pages/` сами по себе не включают `migrate` или `fix` и **не** стартуют скил. «Hydration» default values / edit-формы без mismatch — сразу `NOT_APPLICABLE` (forms), даже если есть «проверь» или «почини». «Переведи всё» классифицируй до точечного `migrate`. Неясный SSR-запрос без migrate/fix-сигнала — `audit`.

| Сигнал | Режим | Инструменты |
| --- | --- | --- |
| «default values» / «edit hydration» / «гидрация формы» без SSR mismatch | `NOT_APPLICABLE` | hand-off [`frontend-forms`](../frontend-forms/SKILL.md); `references/` не читать |
| «переведи всё на SSR» / «поставь Next и переведи» / массовый перенос соседних экранов | отказ, не `migrate` | файлы и lockfile не менять; если рядом «проверь» — оставайся в `audit` |
| «проверь» / «аудит» / SSR-«гидрация» / «server-client boundary» без «переведи/почини»; `$ARGUMENTS` = `аудит` или path без write-сигнала | `audit` | только чтение, как [`frontend-security/SKILL.md`](../frontend-security/SKILL.md); без записи файлов |
| «нужен ли SSR» / «какой render-mode» этой **одной** странице; `$ARGUMENTS` = `решение` | `decision` | только чтение; одна страница/экран; не обход всего scope и не замена `migrate` |
| «переведи» / «примени» / «мигрируй» на SSR/RSC **этой страницы** (не «всё»); `$ARGUMENTS` = `миграция` | `migrate` | точечная запись только страниц со статусом `MIGRATE`; без массового переноса |
| «почини» hydration mismatch / server-only leak / двойной fetch; не form values; `$ARGUMENTS` = `исправить` | `fix` | точечные правки confirmed defects в заявленном scope |
| нет клиентского scope / не UI / чужая зона | `NOT_APPLICABLE` | `references/` не читать |

Явный path, открытый экран, каталог или `changed` / `diff` / PR — заявленный scope. Без понятного клиентского scope **спроси** страницу, экран, каталог или diff; не сканируй монорепозиторий наугад и не подставляй «все `app/`» по умолчанию.

«Переведи всё на SSR» / «поставь Next и переведи» / миграция соседних экранов «заодно» — **отказ**: не переходи в `migrate`, не ставь фреймворк, не расширяй scope. Пустой scope, только backend/infra или нет UI — верни `NOT_APPLICABLE`, `references/` дальше не читай.

Явный path ограничивает **проверяемые страницы**, но не обрывает доказательство: в `audit`/`decision` разрешён read-only просмотр импорта/родителя/island за пределы path ровно настолько, чтобы подтвердить трек, границу или first render. Эти файлы укажи как evidence dependencies и не считай расширением scope. В `migrate`/`fix` правь только заявленный scope.

## Шаг 2. Обнаружь render-mode зоны

Смотри репозиторий, не привычку модели. Не навязывай Next, RSC, Remix или islands, которых нет в lockfile/манифесте. CSR/Vite/SPA **этого** репозитория не канонизируй как продуктовый прецедент для других зон.

Что проверить для **discovery** (достаточно сигналов, не полный аудит на этом этапе):

- манифест и конфиги: `package.json` / workspace; `next.config.*`, Vite, Remix, Nuxt, SvelteKit, Astro — как **сигнал трека**, не как триггер скила;
- корни UI: `src/`, `app/`, `pages/`, `apps/*`, `packages/*` с клиентской или server-entry точкой;
- уже существующий трек: CSR-only, SSR, RSC, loaders, islands, hybrid server parent + client island;
- подтип «когда HTML»: `static` / `isr` / `request-dynamic` (не отдельный трек; SSG-страница остаётся RSC или classic SSR);
- data-граница: есть ли server fetch / dehydrate / повторный client GET (HOW кэша не расписывай — hand-off в api);
- исключения: generated, vendor, `node_modules`, тесты, фикстуры, docs, markdown-примеры.

Не делай: установка Next/Remix/Nuxt/другого SSR-фреймворка, правка lockfile, смена Pages ↔ App Router, массовый `'use client'` или массовый `ssr:false`.

Смешанный стек — проверяй каждый найденный UI-корень своим API; не унифицируй на RSC. Нет SSR/RSC/loader/islands-прецедента в зоне — не выдумывай трек: статус `KEEP` и в рекомендации «SSR не нужен / фреймворк не ставить», не `MIGRATE`. `NOT_APPLICABLE` — нет UI / чужая зона, не синоним CSR-only. RSC не требует пакета с именем RSC: `next` + server `app/` page без `'use client'` на корне достаточно.

Детали треков и stop-условий — [discovery-and-render-mode.md](references/discovery-and-render-mode.md).

## Шаг 3. Выбери references и статусы

По типу задачи читай только нужное (progressive disclosure — не грузи все references сразу):

| Категория | Reference |
| --- | --- |
| какой трек, подтип «когда HTML», CSR vs SSR vs RSC vs islands, когда остановиться | [discovery-and-render-mode.md](references/discovery-and-render-mode.md) |
| доказательства статуса, формат audit, `decision` ≠ `audit`, точечный migrate, отказ mass-migrate | [decision-and-migration.md](references/decision-and-migration.md) |
| узкий island, serializable props, server-only import, cookies/headers как render-input | [server-client-boundaries.md](references/server-client-boundaries.md) |
| first render, mismatch, streaming, один источник данных | [hydration-and-data.md](references/hydration-and-data.md) |
| как надо / как не надо | [examples-good-and-bad.md](references/examples-good-and-bad.md) |
| dry-run / самопроверка режимов (не каждый audit) | [evaluation-scenarios.md](references/evaluation-scenarios.md) |

Узкий запрос (одна страница, один mismatch, один island) — читай один тематический файл плюс статусы, не все. Рецепты роутера, кэша и form defaults **не копируй**.

**Evidence / status matrix** (на каждую страницу в scope). Перед статусом читай [decision-and-migration.md](references/decision-and-migration.md).

| Статус | Когда ставить | Что не делать |
| --- | --- | --- |
| `MIGRATE` | В зоне уже есть SSR/RSC/loader/islands-трек **и** есть конкретная польза или дефект **и** граница сужается безопасно (server parent + узкий client island) | Не ставить из-за открытого `page.tsx`; не мигрировать без явного «переведи» |
| `KEEP` | Чистый CSR по замыслу зоны; преимущественно browser-only UI (editor/chart); уже корректный server parent + client island; нет SSR-прецедента и ставить фреймворк нельзя; страница уже `static`/`isr` и граница корректна | Не «улучшать» корректный island; не предлагать `ssr:false` на всю страницу. CSR-only → в рекомендации «SSR не нужен / фреймворк не ставить» |
| `INCOMPLETE` | Недостаток evidence: смешанный стек без доказанного трека, нельзя подтвердить first render / границу, файлы недоступны | Не выдумывать `MIGRATE`; не повышать гипотезу до дефекта |
| `NOT_APPLICABLE` | Нет страницы/UI, не render-surface, запрос про другую зону (form values, XSS, route syntax, placement) | Не читать `references/` дальше по этому узлу; отдай соседу. Не ставь на CSR-only вместо `KEEP` |

`MIGRATE` без всех трёх условий — ошибка классификации. Неподтверждённый сигнал — не выдуманный дефект и не `MIGRATE`.

## Шаг 4. Аудит, решение, миграция или fix

### Режим `audit`

Пройди страницы заявленного scope. Файлы, конфиги агента и зависимости не меняй. Итог — только в чат, persistent SSR-отчёт не создавай.

На каждую страницу:

- статус: `MIGRATE` | `KEEP` | `INCOMPLETE` | `NOT_APPLICABLE`;
- evidence: трек зоны, когда HTML (`static` / `isr` / `request-dynamic` / `н/п`), польза или дефект, граница, `file:line`, confidence;
- блокеры и evidence dependencies;
- hand-off (placement / loader HOW / route syntax / XSS / form hydration / визуал / a11y);
- безопасная рекомендация без патча.

`decision` этот обход не заменяет: `audit` — сетка scope, `decision` — один экран.

### Режим `decision`

Узкий вердикт по **одной** странице/экрану: нужен ли здесь SSR/RSC/island. Тот же статус и те же доказательства, что в `audit`. Не обходи соседние экраны, не правь файлы, не подменяй `migrate`.

### Режим `migrate`

Разрешён только при явном «переведи/примени» и только для страниц, которые уже имеют статус `MIGRATE` (из этого прогона `audit`/`decision` или из явно подтверждённого вердикта в запросе). Страницы со статусом `KEEP` / `INCOMPLETE` / `NOT_APPLICABLE` не трогай.

- Держи server parent, вынеси в client только island с браузерным API.
- Props через границу — serializable; server-only модуль в client не импортируй.
- Файлы должны переехать — сначала [`frontend-architecture`](../frontend-architecture/SKILL.md) на placement, затем граница здесь.
- Не меняй роутер-трек (Pages ↔ App, CSR → SSR-фреймворк), не ставь пакеты, не мигрируй соседей.
- HOW loader/dehydrate — [`frontend-api`](../frontend-api/SKILL.md), не изобретай второй data-слой.

Как надо / как не надо — [examples-good-and-bad.md](references/examples-good-and-bad.md). Детали безопасной миграции — [decision-and-migration.md](references/decision-and-migration.md) и [server-client-boundaries.md](references/server-client-boundaries.md).

### Режим `fix`

Закрой **подтверждённые** hydration / server-client дефекты в заявленном scope, затем **повторно проверь этот же scope**.

- Deterministic first render: не оставляй `Date.now()` / `window` / случайность на общем server+client пути.
- CSR-bailout без слота: оберни island в `<Suspense>` (или слот трека). Не создавай `loading.tsx` и не копируй router hook.
- Не чини «hydration» form default values — это [`frontend-forms`](../frontend-forms/SKILL.md).
- Двойной fetch без принятого hydration-контракта не оставляй; контракт кэша — [`frontend-api`](../frontend-api/SKILL.md).
- XSS / token в бандле не закрывай render-границей: hand-off в [`frontend-security`](../frontend-security/SKILL.md).
- Не выходи за scope, не рефакторь соседние экраны, не ставь фреймворк.

Детали — [hydration-and-data.md](references/hydration-and-data.md).

## Шаг 5. Чеклист перед сдачей

Если пункт не выполнен — не сдавай.

**Общее**

- Режим выбран по таблице шага 1; tool policy режима не нарушен.
- Scope явный (страница / экран / каталог / diff); монорепо наугад не сканировали.
- Трек взят из репо; SSR-фреймворк и lockfile не менялись.
- Соседи получили hand-off, а не скопированные рецепты.
- PII, cookie/header values и токены — `[REDACTED]`. Runtime-логи не добавлялись.

**Только `audit` / `decision`**

- Файлы не менялись.
- У каждой проверенной страницы есть статус из матрицы и evidence с `file:line`.
- `decision` закрыл одну страницу и не разросся в полный `audit`.
- `INCOMPLETE` содержит блокеры, не выдуманный `MIGRATE`.
- Отчёт только в чат.

**Только `migrate`**

- Правились только страницы со статусом `MIGRATE` и только по явному «переведи/примени».
- Массовый «переведи всё» отвергнут.
- Placement уехал в architecture, если файлы должны были сменить папку.
- Граница узкая; serializable props; server-only не попал в client.

**Только `fix`**

- Правились только confirmed defects в заявленном scope.
- Bailout без слота закрыли обёрткой island, не файлом `loading.tsx`.
- Form-values hydration не лечили как SSR mismatch.
- Scope повторно проверен; массового рефакторинга не было.
- Запущены уже существующие lint / typecheck по затронутой области (без установки пакетов).

Перед сдачей — коротко, не эссе:

- **Режим:** `audit` / `decision` / `migrate` / `fix` / `NOT_APPLICABLE`.
- **Доказательства трека:** 2–3 факта из репо (CSR-only / SSR / RSC / islands / hybrid; `static` / `isr` / `request-dynamic` / `н/п`).
- **Scope:** путь или diff.
- **Статусы:** сколько `MIGRATE` / `KEEP` / `INCOMPLETE` / `NOT_APPLICABLE`.
- **Hand-off:** routing / api / architecture / security / forms / layout / a11y — если были.

## Запрещено

- ставить Next, Remix, Nuxt, islands-runtime или любой SSR-фреймворк, которого нет в lockfile;
- мигрировать CSR / Pages Router / App Router без явного «переведи/примени»;
- массовый перенос соседних экранов и «переведи всё на SSR»;
- считать открытый `page.tsx` / `app/` / `pages/` стартом `migrate` или `fix`;
- копировать Next/RR рецепты, loader/cache HOW, form-values hydration или source-to-sink XSS;
- изобретать папочную схему в `migrate` вместо architecture;
- запись файлов в `audit` / `decision`;
- persistent SSR-отчёт, waiver, runtime-логи, debug first-render HTML, cookie/token values;
- чинить доступность или XSS через смену render-mode.

## Если что-то пошло не так

- Запрос только про `'use client'` / router API / `redirect` / `loading` / metadata → [`frontend-routing`](../frontend-routing/SKILL.md), этот скил дальше не веди.
- Запрос только про loader/dehydrate/query keys → [`frontend-api`](../frontend-api/SKILL.md), этот скил дальше не веди.
- Нужен перенос файлов / public API → сначала [`frontend-architecture`](../frontend-architecture/SKILL.md).
- XSS / token в бандле / `cookies()` как auth → [`frontend-security`](../frontend-security/SKILL.md) или [`aif-security-checklist`](../../aif-security-checklist/SKILL.md).
- Default values / edit hydration формы → [`frontend-forms`](../frontend-forms/SKILL.md), не SSR mismatch.
- Skeleton / fallback как визуал → [`frontend-layout`](../frontend-layout/SKILL.md).
- Системный a11y/WCAG → [`frontend-a11y`](../frontend-a11y/SKILL.md).
- «Переведи всё на SSR» / «поставь Next и переведи» → отказ; оставайся в `audit` или остановись.
- Scope неясен → спроси страницу, экран, каталог или diff; не сканируй весь монорепо.
- Стек смешанный → проверяй каждый UI-корень своим треком; не унифицируй.
- Нет SSR-прецедента в зоне → `KEEP` + «SSR не нужен / фреймворк не ставить», не предлагай поставить фреймворк.
- Evidence не хватает → `INCOMPLETE` с блокерами, не выдумывай `MIGRATE`.
- Запрос «проверь» / path без «переведи/почини» → оставайся в `audit`.
