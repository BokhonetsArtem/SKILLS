# Discovery and render-mode

Читай на шаге 2. Дальше грузи только разделы того **трека**, который доказан манифестом, конфигом или точкой входа. Не навязывай Next, RSC, Remix, Nuxt, islands или Vite-SSR, которых нет в lockfile.

CSR / Vite / SPA **текущего** репозитория или соседнего пакета — сигнал **этой** зоны, не канон скила и не продуктовый прецедент для других корней. Не переноси «у нас всегда CSR» на пакет, где уже есть loader/RSC.

Этот файл не меняет код и не ставит пакеты. Cookie/header values, токены и PII в чат не копируй: `[REDACTED]`.

## 1. Что считается треком

Трек — уже существующий способ отдать первый HTML и провести границу server/client. Имя папки (`app/`, `pages/`, `src/`) само по себе трек не доказывает.

| Трек | Чем доказывается | Что из этого следует |
| --- | --- | --- |
| CSR-only | `index.html` + бандлер; `createRoot` / клиентский `hydrateRoot` на пустой `#root`; нет SSR-фреймворка и нет loader/RSC в зоне | Не предлагай поставить фреймворк. Вопрос про render-mode → `KEEP` + в рекомендации «SSR не нужен / фреймворк не ставить». Не `MIGRATE`. `NOT_APPLICABLE` — только нет UI / чужая зона |
| Classic SSR | `getServerSideProps` / `getStaticProps` / `getInitialProps`, Remix/React Router `loader`, Nuxt server page, серверный render + клиентский hydrate **уже есть** | Границу сужай внутри этого трека. Подтип «когда HTML» — §1.1. Pages → App / CSR → Next — не миграция этого скила |
| RSC | Фреймворк с RSC уже в зоне (`next` в манифесте/lockfile **или** явный `react-server-dom-*`) **и** server-first дерево (`app/` + server page/layout без `'use client'` на корне). Отдельный пакет с именем RSC не нужен: `next` тянет RSC транзитивно | Client — только island с браузерным API. Не вешай `'use client'` на всю страницу. Имя папки `app/` само по себе трек не доказывает |
| Islands | Astro `client:*`, Qwik, partial hydrate, изолированный `hydrateRoot` на виджете | Не раздувай island до страницы. Корректный island — кандидат на `KEEP` |
| Hybrid | Server parent (данные/разметка) + узкий client island | Если граница и first render уже корректны — `KEEP`, не «улучшай» |
| Нет прецедента | В зоне нет SSR/RSC/loader/islands и ставить фреймворк нельзя | `KEEP` + «SSR не нужен / фреймворк не ставить», не выдуманный `MIGRATE`. Не путай с `NOT_APPLICABLE` |

Несколько UI-корней — каждый своим треком. Не унифицируй монорепо на RSC «чтобы было одинаково».

## 1.1 Когда появляется HTML (подтип, не трек)

Трек (§1) — как устроена граница. Подтип — **когда** готов первый HTML. Не добавляй SSG/ISR как шестой трек: статическая `app/` page остаётся RSC; `getStaticProps` — classic SSR. «Уже отдаёт HTML на билде» ≠ «нужен runtime SSR».

| Подтип | Чем доказывается | Что из этого следует |
| --- | --- | --- |
| `static` | `getStaticProps`, `generateStaticParams`, `dynamic = 'force-static'`, нет request-only API на странице | Вопрос «нужен ли SSR» при корректной границе → `KEEP`: HTML уже на билде. Не предлагай GSSP / request SSR «для SEO» |
| `isr` | В зоне уже есть ISR / segment `revalidate` на этом типе страниц | Факт периодической пересборки — здесь. TTL и рецепт `revalidate` не изобретай и не меняй без явного запроса |
| `request-dynamic` | `getServerSideProps`, `cookies()` / `headers()` как server API, `connection()`, `unstable_noStore` / эквивалент зоны | Зафиксируй «route dynamic». Не снимай `cookies()` «чтобы стало static». Auth по cookie — [`frontend-security`](../../frontend-security/SKILL.md) |
| `н/п` | CSR-only / нет server HTML | В evidence пиши `н/п`, не выдумывай `static`/`dynamic` |

Client-cache HOW (`dehydrate`, query keys, SWR `revalidateOnFocus`) — [`frontend-api`](../../frontend-api/SKILL.md). Segment `revalidate` / `noStore` / `connection()` в api **не** отдавай: это не query-cache. Рецепт этих API не копируй.

## 2. Где искать доказательства

Смотри репозиторий, не привычку модели. Достаточно сигналов для классификации трека; полный аудит страниц — не этот файл.

| Источник | Что снимать | Чего не делать |
| --- | --- | --- |
| `package.json` / workspace / lockfile | `next`, `react-dom/server`, `@remix-run/*`, `nuxt`, `@sveltejs/kit`, `astro`, `vite` + ssr-плагин — как **наличие** трека. Явный `react-server-dom-*` — запасной сигнал RSC, не обязательный. Пакета с именем RSC не ищи | Не ставить пакет из этой таблицы |
| Конфиг | `next.config.*`, `vite.config.*`, `nuxt.config.*`, `remix.config.*`, `astro.config.*` — только как сигнал | Не считать конфиг триггером скила и не переписывать его «под SSR» |
| Корни UI | `src/`, `app/`, `pages/`, `apps/*`, `packages/*` с client- или server-entry | Не сканировать весь монорепо; соседний app — другой корень |
| Когда HTML | `getStaticProps` / `generateStaticParams` / `force-static` vs `getServerSideProps` / `cookies()` / `headers()` / `connection()` / `noStore` | Не менять подтип. Рецепт segment cache не писать; client-cache HOW — api |
| Data-граница | Server fetch / `loader` / RSC `async` / dehydrate vs повторный client GET | HOW ключей и `dehydrate` — [`frontend-api`](../../frontend-api/SKILL.md) + [queries-cache-and-concurrency.md](../../frontend-api/references/queries-cache-and-concurrency.md) §6 |
| Роутер-файлы | `page.tsx`, `loading.tsx`, `generateMetadata`, `routes.ts` — как **расположение** поверхности | Синтаксис маршрута — [`frontend-routing`](../../frontend-routing/SKILL.md); не копируй Next/RR рецепты |

Явный path ограничивает проверяемые страницы. Read-only взгляд в импорт/родителя/island за path — только чтобы подтвердить трек или first render. Эти файлы — evidence dependencies, не расширение scope.

## 3. Stop-условия

Останови discovery (этот файл дальше не применяй или верни статус без выдуманного трека), если:

| Условие | Статус / действие |
| --- | --- |
| Нет клиентского UI / не render-surface | `NOT_APPLICABLE` |
| Scope не задан | Спроси страницу, экран, каталог или diff; не подставляй «все `app/`» |
| Запрос про другую зону (form values, XSS, route syntax, placement, визуал skeleton, системный a11y) | `NOT_APPLICABLE` + hand-off |
| В зоне нет SSR/RSC/loader/islands-прецедента | `KEEP` + «SSR не нужен / фреймворк не ставить»; не предлагай install |
| Смешанный стек, трек корня не доказан | `INCOMPLETE` по этому корню; не унифицируй |
| Файлы/lockfile недоступны | `INCOMPLETE` с блокером |
| Корень — generated / vendor / тесты / docs | Пропусти как прод-трек |

Не делай из stop-условия задачу `migrate`.

## 4. Смешанный стек

В одном репозитории могут жить CSR-пакет, Pages-приложение и RSC-приложение.

1. Раздели UI-корни по entry/манифесту.
2. Классифицируй каждый корень своим API из §1.
3. Не переноси остров из корня A как прецедент для корня B.
4. Не предлагай свести всё к одному фреймворку.

Неоднозначность («есть `next` в root, а экран в CSR-пакете без SSR») — `INCOMPLETE`, пока не доказан **этот** entry, не `MIGRATE`.

## 5. Исключения (не считать прод-треком)

Не снимай трек и не поднимай статусы **только** из:

- `node_modules/`, vendor-бандлы, скопированный third-party dist;
- generated: `.next/`, `dist/`, `build/`, `coverage/`, `.svelte-kit/`, `generated/`;
- тесты: `__tests__`, `*.test.*`, `*.spec.*`, `cypress/`, `e2e/`, `playwright/`;
- фикстуры, моки, Storybook `*.stories.*`;
- docs и учебные сниппеты в `*.md`.

Код рядом с тестом, но в прод-модуле — смотри прод-модуль.

## 6. Чего не делать на discovery

- Ставить Next / Remix / Nuxt / islands-runtime / любой SSR-фреймворк, которого нет в lockfile.
- Канонизировать Vite+CSR текущего репо как «правильный дефолт продукта».
- Считать открытый `page.tsx` / `app/` / `pages/` доказательством RSC или стартом `migrate`.
- Менять Pages ↔ App Router, CSR ↔ SSR «на всякий случай».
- Копировать сюда файловую convention Next/RR — это [`frontend-routing`](../../frontend-routing/SKILL.md) + [next-routing-rules.md](../../frontend-routing/references/next-routing-rules.md) §2.
- Решать placement server-модуля vs island — [`frontend-architecture`](../../frontend-architecture/SKILL.md).
- Аудировать `cookies()` / `headers()` как auth — [`frontend-security`](../../frontend-security/SKILL.md). Здесь они сигнал render-input **и** подтипа: вызов на server page делает route `request-dynamic`. Не снимай их, чтобы «включить SSG».

Детали статусов после того, как трек известен — [decision-and-migration.md](decision-and-migration.md).
