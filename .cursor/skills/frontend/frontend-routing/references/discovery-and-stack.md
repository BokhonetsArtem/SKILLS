# Стек и обнаружение маршрутизации

Читай, когда трек неясен, в репо несколько роутеров, нужно отличить mode React Router или App Router от Pages Router. Рецепты реализации — только rules/examples **выбранного** трека, не все сразу. Links, URL-state, guards и a11y навигации — [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md).

Не ставь `react-router`, `react-router-dom`, `next` или другой router, пока пользователь явно не попросил. Не мигрируй declarative → data, Pages → App, Remix → React Router v7 «заодно».

## 1. Это не маршрутизация этого скила

Слово «роут» в UI не равно client routing. Если нет route tree, API установленного router, route-level URL-state или framework routing convention — остановись и отдай другому скилу.

| Что видно | Куда |
| --- | --- |
| Куда класть page-модуль / public API, без декларации маршрута | [`frontend-architecture`](../../frontend-architecture/SKILL.md) |
| HTTP 3xx, `Location`, path/query params **endpoint** | [`frontend-api`](../../frontend-api/SKILL.md) |
| Поиск, фильтр, пагинация UI без записи в URL маршрута | [`frontend-layout`](../../frontend-layout/SKILL.md) |
| Простой `<a href>` на статику, якорь, download, внешнюю ссылку без смены route behavior | не этот скил |
| Dirty form / confirm leave / submit | [`frontend-forms`](../../frontend-forms/SKILL.md) |
| CSS nested layout, skeleton, сетка экрана | [`frontend-layout`](../../frontend-layout/SKILL.md) |
| Аудит open redirect / client-only authorization | [`frontend-security`](../../frontend-security/SKILL.md) |

Имя файла `*Page*` или папки `app/` / `pages/` само по себе ничего не доказывает. FSD-слой `src/pages` без Next `_app` / GSSP и без `createBrowserRouter` / `<Routes>` — не файловый роутер. Смотри router API и convention фреймворка.

## 2. Сначала репозиторий, затем major version

Смотри **зону изменения**, не популярность библиотеки и не «latest из памяти». Достаточно сигналов, не полный аудит репо.

Порядок:

1. Манифест и lockfile: `react-router`, `react-router-dom`, `@react-router/dev`, `next`, другой router зоны (`@tanstack/react-router`, `wouter`, Remix route modules и т.п.).
2. **Major version** установленного пакета. Синтаксис v5 / v6 / v7 и Next 13–14 / 15 / 16 — разные контракты. Не объявляй latest универсальным.
3. Каркас: `app/**/page.tsx` / `layout.tsx` vs `pages/_app` / `pages/**`; `middleware.ts` vs `proxy.ts` — только тот файл, который есть у установленной версии.
4. Импорты **в зоне изменения**: `createBrowserRouter`, `RouterProvider`, `BrowserRouter`, `<Routes>`, `Outlet`, `next/navigation`, `next/router`, `next/link`.
5. Ближайший маршрут той же зоны — локальный прецедент важнее «главного» стека репо.

Пакет и import path сверяй с lockfile:

| Что стоит | Типичный import | Не делай |
| --- | --- | --- |
| `react-router-dom` v6 | `from 'react-router-dom'` | не переписывай на `react-router` «как в v7 docs» |
| `react-router` v7 (+ опционально `@react-router/dev`) | `from 'react-router'` / `react-router/dom` — как у соседей и docs **этой** major | не оставляй v6-only API, которого уже нет, и не тащи v7 framework plugin в data/declarative зону |
| `react-router-dom` v5 | `Switch`, `Redirect`, `useHistory`, `component={}` | не подставляй v6 `<Routes>` / `element` / `useNavigate`; `examples-react-router*.md` не читай — только docs установленной v5 |
| `next` | `next/link`, `next/navigation` или `next/router` по треку зоны | не ставь React Router рядом с файловым роутингом Next |

В lockfile одна библиотека, в **этой** зоне импорты другой — трек **зоны**, не «главный» пакет репо.

## 3. React SPA: какой это роутер

Папка `src/` и React без `next` ещё не значит React Router.

| Доказательство в зоне | Трек |
| --- | --- |
| Vite plugin `@react-router/dev`, `routes.ts` / route modules, `Route` typegen (`./+types/...`) | React Router **framework** |
| `createBrowserRouter` / `createHashRouter` / `createMemoryRouter` + `RouterProvider`; часто `loader` / `action` / `errorElement` | React Router **data** |
| `BrowserRouter` / `HashRouter` + `<Routes>` / `<Route>` без data router | React Router **declarative** |
| `react-router-dom` v5: `Switch`, `useHistory`, `component={}`, `<Redirect>` | React Router **v5**; examples v6/v7 **не** читай, только docs установленной major |
| Remix `app/routes/` как каркас (ещё не v7) | существующий Remix-трек зоны; API — docs установленной версии Remix, не Next и не «просто RR declarative» |
| `@tanstack/react-router`, `wouter`, `expo-router` и другой установленный SPA-router | существующий трек зоны; React Router / Next examples **не** читай |
| Нет router-пакета, нет файлового каркаса Next | спроси; не ставь React Router «по умолчанию» |

Modes React Router **аддитивны**, но это не лицензия апгрейдить зону. Data = declarative + loaders/actions вне React-дерева. Framework = data + Vite plugin / route modules. Не добавляй `loader` в declarative-зону и не подключай `@react-router/dev` к data-зоне без явного запроса.

`createBrowserRouter` **без** `loader`/`action` в дереве — всё равно data mode (`RouterProvider`). Не подменяй его на `<BrowserRouter>`+`<Routes>` и наоборот.

Hash-router (`HashRouter` / `createHashRouter`) — только если зона уже на hash. Не переводи browser ↔ hash «для хостинга», пока пользователь не просил; hosting fallback — [react-router-rules.md](react-router-rules.md).

## 4. Next.js: App или Pages

Имя папки `app/` само по себе не FSD и не App Router. Смотри файлы convention и импорты. Архитектурный слой vs каркас фреймворка — [`frontend-architecture`](../../frontend-architecture/references/principles-and-discovery.md).

| Доказательство в зоне | Трек |
| --- | --- |
| `app/**/page.tsx` / `layout.tsx`, импорты `next/navigation`, `redirect` / `notFound` из `next/navigation` | Next **App Router** |
| `pages/_app`, `pages/**/*.tsx`, `getServerSideProps` / `getStaticProps` / `getInitialProps`, импорты `next/router` | Next **Pages Router** (legacy) |
| В репо есть и `app/`, и `pages/`, но правка только в одном дереве | трек **этого** дерева |
| `app/` и `pages/` равноправно затронуты одной задачей | спроси; не мигрируй Pages → App и не смешивай `next/navigation` с `next/router` в одном экране |

`middleware.ts` (Next ≤15) и `proxy.ts` (Next 16+) — request interception установленной major, не отдельный роутер и не authorization. Не создавай второй файл с другим именем «на будущее». Не переименовывай `middleware` → `proxy` без запроса, если текущая major ещё на `middleware`.

## 5. Смешанный и неясный стек

| Ситуация | Решение |
| --- | --- |
| В репо несколько роутеров, в зоне один устойчивый прецедент | Трек зоны. Чужой examples-файл не читай |
| Next App в одной зоне, React Router SPA в другой (монорепо / две оболочки) | Трек **зоны изменения**; пакеты соседней оболочки не тащи |
| В одном экране уже смешаны два router API | Не усугубляй. Новую навигацию — в уже выбранный API этого экрана |
| Библиотека распознана, но для неё нет `examples-*.md` | Локальный route tree + official docs **установленной версии** + [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md). Стек не переключай |
| Сигналов мало, App+Pages равноправны или два SPA-router в одной зоне | Спроси. Не ставь пакет и не выбирай «как принято» |
| Пользователь явно просит другой роутер | Можно ставить только после явного запроса; не мигрируй соседние маршруты заодно |

Паттерн бери у **ближайшего маршрута той же зоны**, не у первого попавшегося поиска и не из другого репозитория. Если в зоне уже есть path helper, `Link`-обёртка или route config — переиспользуй. Имена и импорты — как **здесь**. Нейтральные учебные пути вроде `src/routes.tsx` допустимы в скиле; чужие product path и alias не переноси.

## 6. Что читать дальше

| Нужно | Файл |
| --- | --- |
| Link vs navigate vs redirect/replace, params/search/hash, nested layouts, 404/error/loading, lazy, guards, scroll/focus/title | [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md) |
| Рецепт React Router data / framework | [react-router-rules.md](react-router-rules.md) + `examples-react-router.md`, если файл есть |
| Рецепт React Router declarative | [react-router-rules.md](react-router-rules.md) + `examples-react-router-declarative.md`, если файл есть |
| React Router v5 / другой SPA-router / нет examples | этот файл + navigation-state + локальный tree + docs установленной версии; `examples-react-router*.md` и Next examples не читай |
| Рецепт Next App или Pages | [next-routing-rules.md](next-routing-rules.md) + examples своего трека, если файл есть |
