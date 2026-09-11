---
name: frontend-routing
description: Реализует или изменяет клиентскую маршрутизацию по установленному router-стеку зоны — route tree, navigation, redirect/replace, params/search/hash, URL-state, nested layouts, 404/error/loading, guards. Use when route tree, router API, route-level URL state, framework routing convention, navigation, redirect, params, search, nested layouts, route guards, scroll/focus after navigation. Не для HTTP redirect/endpoint params, обычного поиска или фильтра без URL-state, простого <a> без изменения route behavior. Placement/public API — frontend-architecture; данные маршрута — frontend-api; leave guards формы — frontend-forms; экран/layout — frontend-layout; open redirect/client-only authorization — frontend-security. Не для системного a11y-аудита / WCAG — это frontend-a11y. Не ставит второй роутер и не добавляет runtime-логи.
disable-model-invocation: false
argument-hint: "[маршрут | redirect | params]"
---

# Frontend routing

Реализуй **клиентскую маршрутизацию** по **уже установленному router-стеку этой зоны**: route tree, navigation, redirect/replace, params/search/hash, route-level URL-state, nested layouts, 404/error/loading, lazy/prefetch, guards, scroll/focus после client navigation. Сначала собери positive evidence маршрутизации, затем выбери трек (React Router / другой установленный SPA-router / Next App Router / Next Pages Router), затем читай только релевантные references. Не выбирай роутер «по умолчанию» и не ставь второй «для удобства».

Не ограничивай себе инструменты. Для этой работы нужны доступные инструменты чтения, поиска, правки файлов и запуска уже существующих в репо проверок. Узкий read-only набор здесь не подходит. Runtime-логи navigation, params, return URL и PII не добавляй.

Правила треков, URL-state, guards и примеры **не копируй в этот файл**. Читай только нужный файл из `references/` (пути относительно этой папки скила). Примеры — **только выбранного трека**, не все `examples-*` сразу.

**Не этот скил**, пока нет evidence маршрутизации (route tree, API установленного router, route-level URL-state или framework routing convention). HTTP 3xx / `Location` / params endpoint — [`frontend-api`](../frontend-api/SKILL.md). Обычный поиск или фильтр без записи в URL маршрута — [`frontend-layout`](../frontend-layout/SKILL.md). Простой `<a>` без изменения route behavior — не повод начинать этот скил. Куда класть page-модуль и public API — [`frontend-architecture`](../frontend-architecture/SKILL.md). Leave-guard грязной формы — [`frontend-forms`](../frontend-forms/SKILL.md); этот скил не владеет dirty/submit. Вёрстка экрана, skeleton, CSS nested layout — [`frontend-layout`](../frontend-layout/SKILL.md). Open redirect и выдача client guard за authorization — [`frontend-security`](../frontend-security/SKILL.md), не повод начинать аудит из обычной навигации. Системный a11y-аудит / WCAG — [`frontend-a11y`](../frontend-a11y/SKILL.md); scroll / focus / title после client navigation при работе над маршрутом остаются здесь.

Маршруты (формулировка запроса → скил):

- «куда положить page/слайс / public API» → [`frontend-architecture`](../frontend-architecture/SKILL.md)
- «route tree / navigation / redirect / params / search / URL-state» → этот скил
- «loader/action ходит в HTTP / query / DTO / cache» → [`frontend-api`](../frontend-api/SKILL.md)
- «dirty form / leave guard / submit» → [`frontend-forms`](../frontend-forms/SKILL.md)
- «вёрстка экрана / skeleton / CSS layout» → [`frontend-layout`](../frontend-layout/SKILL.md)
- «проверь доступность / WCAG / системный a11y-аудит» → [`frontend-a11y`](../frontend-a11y/SKILL.md)
- «open redirect / client-only auth как защита» → [`frontend-security`](../frontend-security/SKILL.md)

Разведение: architecture — размещение page-модуля и ownership; этот скил — декларация маршрута, URL-контракт и client navigation; API — загрузка данных маршрута; forms — leave-guard формы через уже принятый blocker API; layout — визуал; security — source-to-sink по URL/guard. Если задача сразу про **новый слайс и маршрут** — сначала размещение (architecture), затем маршрут (здесь), затем HTTP ([`frontend-api`](../frontend-api/SKILL.md)), если loader/page data нужны, затем оболочка (layout), если она нужна. Шаги не смешивай.

**Режим** (не грузи лишнее):

| Задача | Что читать |
| --- | --- |
| Стек неясен / смешанный / какой трек / какой mode React Router | [references/discovery-and-stack.md](references/discovery-and-stack.md) |
| Links, navigate, redirect/replace, params/search/hash, URL-state, nested layouts, 404/error/loading, lazy, guards, scroll/focus/title | [references/navigation-state-and-boundaries.md](references/navigation-state-and-boundaries.md) |
| Рецепт React Router data / framework (только этот mode) | [references/react-router-rules.md](references/react-router-rules.md) + [references/examples-react-router.md](references/examples-react-router.md) |
| Рецепт React Router declarative (только этот mode) | [references/react-router-rules.md](references/react-router-rules.md) + [references/examples-react-router-declarative.md](references/examples-react-router-declarative.md) |
| Рецепт Next App Router (только этот трек) | [references/next-routing-rules.md](references/next-routing-rules.md) — App Router + [references/examples-next-app-router.md](references/examples-next-app-router.md) |
| Рецепт Next Pages Router (legacy, только этот трек) | [references/next-routing-rules.md](references/next-routing-rules.md) — Pages Router + [references/examples-next-pages-router.md](references/examples-next-pages-router.md) |
| Другой установленный SPA-router / React Router v5 / нет examples-файла | [references/discovery-and-stack.md](references/discovery-and-stack.md) + [references/navigation-state-and-boundaries.md](references/navigation-state-and-boundaries.md) + локальный route tree и official docs **установленной версии** |
| Проверка качества самого скила | [references/evaluation-scenarios.md](references/evaluation-scenarios.md) |

## Шаг 1. Классифицируй задачу

Опора: сообщение пользователя, `$ARGUMENTS` (если скил вызван с аргументом), открытые файлы. В обычном чате `$ARGUMENTS` может быть пустым — это не стоп.

Сначала проверь **positive evidence** маршрутизации в зоне. Без одного из сигналов ниже — остановись, `references/` не читай.

| Evidence | Это этот скил |
| --- | --- |
| Route tree: `createBrowserRouter`, `RouteObject`, `<Routes>`, `app/**/page.tsx`, Next `pages/_app` / `getServerSideProps` | Да |
| API установленного router: `Link`/`NavLink` роутера, `useNavigate`/`useRouter`/`redirect`/`notFound`, `Outlet` | Да |
| Route-level URL-state: search/hash/params, которыми владеет маршрут, а не локальный `useState` фильтра | Да |
| Framework routing convention: Next `app/**/page.tsx` / `layout.tsx`, Next `pages/_app` + `next/router`, `middleware.ts` / `proxy.ts` установленной версии | Да |
| HTTP redirect, `Location`, query/path params **endpoint** без client route | [`frontend-api`](../frontend-api/SKILL.md) — остановись |
| Поиск, фильтр, пагинация UI без записи в URL маршрута | [`frontend-layout`](../frontend-layout/SKILL.md) — остановись |
| `<a href>` на статику, якорь, внешнюю ссылку или download без смены route behavior | не этот скил |
| Куда положить page-модуль / public API, без декларации маршрута | [`frontend-architecture`](../frontend-architecture/SKILL.md) — сначала размещение |
| Dirty form, confirm leave, `useBlocker` как часть form lifecycle | [`frontend-forms`](../frontend-forms/SKILL.md) — не этот скил |
| CSS nested layout, skeleton, сетка экрана | [`frontend-layout`](../frontend-layout/SKILL.md) — не этот скил |
| Аудит open redirect / client-only authorization | [`frontend-security`](../frontend-security/SKILL.md) — не этот скил |
| «проверь доступность» / WCAG / системный a11y без работы над маршрутом | [`frontend-a11y`](../frontend-a11y/SKILL.md) — не этот скил |

Открытый `ts`/`tsx` сам по себе не триггер. Имя файла `*Page*` или папки `app/` / `pages/` без router API и без framework convention — ещё не evidence. FSD-слой `src/pages` без Next `_app` / GSSP и без `createBrowserRouter` / `<Routes>` — не файловый роутер.

Несколько сигналов сразу — веди по самому широкому: сначала граница и владелец (architecture), потом маршрут (здесь), потом данные (API), потом визуал (layout).

## Шаг 2. Обнаружь router-стек

Смотри репозиторий, не привычку модели. Не классифицируй трек по популярности React Router и не навязывай его React-проекту с другим установленным роутером.

Что проверить (достаточно сигналов, не полный аудит репо):

- манифест и lockfile: `react-router`, `react-router-dom`, `next`, другой router зоны; major version пакета, не «latest из памяти»;
- каркас: `app/**/page.tsx` / `layout.tsx` vs `pages/_app` / `pages/**`; `middleware.ts` vs `proxy.ts` — только тот файл, который есть у установленной версии;
- импорты в **зоне изменения**: `createBrowserRouter`, `RouterProvider`, `BrowserRouter`, `<Routes>`, `Outlet`, `next/navigation`, `next/router`, `next/link`;
- ближайший маршрут той же зоны — локальный прецедент важнее «главного» стека репо;
- data API (`loader`/`action`/`clientLoader`) — только если зона ими уже пользуется.

| Что видно в зоне | Трек | Что читать |
| --- | --- | --- |
| `next` + `app/**/page.tsx` / `layout.tsx` (App Router доминирует в зоне) | Next App Router | [next-routing-rules.md](references/next-routing-rules.md) — App Router; examples — [examples-next-app-router.md](references/examples-next-app-router.md). Discovery — только если сигналы противоречат |
| `next` + `pages/` (`_app`, `getServerSideProps` / `next/router`) без App Router в зоне | Next Pages Router (legacy) | [next-routing-rules.md](references/next-routing-rules.md) — Pages Router; examples — [examples-next-pages-router.md](references/examples-next-pages-router.md) |
| `react-router` / `react-router-dom`: framework mode (Vite plugin / route modules как каркас) | React Router framework | [react-router-rules.md](references/react-router-rules.md) + [examples-react-router.md](references/examples-react-router.md); mode — [discovery-and-stack.md](references/discovery-and-stack.md) |
| `createBrowserRouter` / `RouterProvider` / data `loader`/`action` | React Router data | то же; не подменяй на `<BrowserRouter>`+`<Routes>` |
| `BrowserRouter` + `<Routes>` / `<Route>` без data router | React Router declarative | [react-router-rules.md](references/react-router-rules.md) + [examples-react-router-declarative.md](references/examples-react-router-declarative.md); не добавляй loaders и не читай data-examples |
| `react-router-dom` v5 (`Switch`, `useHistory`, `component`, `<Redirect>`) | React Router v5 | [discovery-and-stack.md](references/discovery-and-stack.md) + [navigation-state-and-boundaries.md](references/navigation-state-and-boundaries.md) + docs установленной v5; `examples-react-router*.md` не читай |
| Другой установленный SPA-router (`@tanstack/react-router`, `wouter`, Remix route modules и т.п.) | существующий трек зоны | [discovery-and-stack.md](references/discovery-and-stack.md) + [navigation-state-and-boundaries.md](references/navigation-state-and-boundaries.md) + локальный tree и docs установленной версии; React Router / Next examples не читай |
| В репо несколько роутеров, в этой зоне один прецедент | трек зоны | examples **только** этого трека; чужой не подмешивай |
| Сигналов мало, App+Pages равноправны в зоне или два SPA-router | спроси | не ставь пакет и не мигрируй Pages→App / declarative→data «заодно» |

Не устанавливай `react-router`, `react-router-dom`, `next` или другой router, пока пользователь явно не попросил. Не добавляй второй роутер рядом с уже стоящим.

Если библиотека распознана, но отдельного examples-файла для неё нет, это **не** повод переводить зону на React Router или Next. Сохрани её модель route tree / navigation / params, возьми API из установленной версии и ближайшего локального аналога, а универсальные требования к history, URL-state, guards, 404/error/loading и scroll/focus — из тематических references этого скила.

API сверяй с **установленной major version**. Не объявляй синтаксис latest универсальным: `middleware` vs `proxy`, data router vs declarative, `next/navigation` vs `next/router` — только то, что есть в зоне.

## Шаг 3. Выбери references и аналог

По типу задачи читай только нужное (progressive disclosure — не грузи все references сразу):

| Задача | Тематический reference сверх уже выбранных файлов режима |
| --- | --- |
| Link vs `navigate` vs redirect/replace, params/search/hash, вложенные layout, 404/error/loading, lazy/prefetch, blockers, guards, scroll/focus/title | [navigation-state-and-boundaries.md](references/navigation-state-and-boundaries.md) |
| Спорный mode React Router, чужой router, App vs Pages, смешанный стек | [discovery-and-stack.md](references/discovery-and-stack.md) |

Паттерн бери у **ближайшего маршрута той же зоны**, не у первого попавшегося поиска и не из другого репозитория. Если в зоне уже есть path helper, `Link`-обёртка или route config — переиспользуй, не изобретай параллельный каталог путей. Имена и импорты — как **здесь**.

Нейтральные учебные пути вроде `src/routes.tsx` допустимы в examples скила; чужие product path, alias и module id в этот репозиторий не переноси. Нет близкого аналога — читай rules/examples выбранного трека. Нет examples-файла — тематические references + локальный аналог + official docs установленной версии. Всё ещё пусто или спорно — спроси, не выдумывай канон и не переключай стек.

Loaders/actions, route middleware/proxy, parallel/intercepting routes — не default: только при локальном прецеденте или явном запросе.

## Шаг 4. Реализуй маршрутизацию

По выбранному треку и аналогу:

- новый экран подключай в **существующий** route tree / framework convention зоны; параллельный роутер и дублирующий tree не заводи;
- navigation меняет history предсказуемо: push для нового места, replace для login/redirect/коррекции URL; `window.location` / полная перезагрузка — не замена router API;
- params, search и hash сохраняй, если соседние маршруты так делают; локальный фильтр не выдавай за URL-state, пока он не пишется в URL маршрута;
- relative и absolute children не смешивай в одном tree без прецедента; path helper зоны важнее захардкоженной строки, которая разъедется с деревом; `basename` (RR) / `basePath` (Next) задай в конфиге роутера/`next.config` как у зоны — не дублируй префикс в каждом `to`/`href`;
- nested layout — `Outlet` / `layout.tsx` (или эквивалент трека), не CSS-обёртка «как будто маршрут»;
- 404/error/loading — route-level convention трека, не пустой экран и не вечный blank gate;
- guard на клиенте — UX-ограничение навигации, **не** server authorization и не замена session/endpoint checks ([`frontend-api`](../frontend-api/SKILL.md) / [`frontend-security`](../frontend-security/SKILL.md));
- return URL / redirect target — только same-origin relative path по аналогу зоны; произвольный абсолютный URL из query не принимай (open redirect → [`frontend-security`](../frontend-security/SKILL.md));
- после client navigation восстанови scroll по контракту зоны и перенеси focus/document title; визуал loading/error — [`frontend-layout`](../frontend-layout/SKILL.md);
- данные маршрута не дублируй вторым fetch-слоем здесь: loader/page data, которые ходят в HTTP, реализует [`frontend-api`](../frontend-api/SKILL.md), этот скил только вплетает их в route API, если такой прецедент уже есть.

Не копируй локальные anti-patterns даже из этого репо: второй роутер, hardcoded path drift, смешение relative/absolute children, blank loading gate, open redirect, client-only auth как защита, server `redirect`/`notFound` из client-only модуля (и наоборот). Как надо / как не надо — в examples выбранного трека (или в тематических references, если examples-файла нет).

После маршрута, если нужна вёрстка экрана — передай её в [`frontend-layout`](../frontend-layout/SKILL.md). Куда лежат page-файлы — уже решено architecture; здесь файлы не переезжают, кроме декларации маршрута в принятом tree.

## Шаг 5. Чеклист перед сдачей

Если пункт не выполнен — почини, не сдавай. Пункты про конкретный инструмент — только если он **есть** в репозитории.

- **Evidence.** Был хотя бы один positive signal маршрутизации. HTTP redirect, фильтр без URL-state и простой `<a>` сюда не тащили.
- **Трек.** Выбран уже доминирующий в зоне (React Router mode / v5 / другой SPA-router / Next App / Next Pages). Второй роутер не установлен. Прочитан только examples своего mode/трека (v5 и чужой SPA — `examples-react-router*.md` / Next examples не читали). API сверен с установленной major version.
- **Границы.** Placement/public API не решались здесь. HTTP/DTO/cache не писались здесь. Form leave-guard не перехватывался здесь. Экран не верстался здесь. Security-аудит не запускался без явного запроса.
- **History и URL.** Push/replace осмысленны; params/search/hash не потеряны без причины; path не захардкожен в обход helper/tree зоны.
- **Состояния маршрута.** 404/error/loading — route-level, не blank gate. Nested layout через convention трека.
- **Guards.** Client guard ≠ authorization. Return URL — same-origin. Open redirect не добавлен.
- **a11y навигации.** После client navigation — scroll restoration по аналогу, focus и document title.
- **Данные.** Navigation, params, return URL и PII не логируй.
- **Условные возможности.** Loaders/actions, middleware/proxy, parallel routes — только при прецеденте или явном запросе. SPA hosting fallback и тестовый harness — по аналогу зоны, новый runner не добавляй.
- **Проверки репозитория.** Запусти существующие targeted tests, lint и typecheck по затронутой области. При изменении поведения обнови тесты в принятом стеке; новый test runner и runtime-логи не добавляй.

Перед сдачей — коротко, не эссе:

- **Трек:** React Router (framework / data / declarative / v5) / другой SPA-router / Next App / Next Pages (и что спросили у пользователя).
- **Доказательства:** 2–3 факта из репо (зависимость+версия, импорт зоны, ближайший route tree).
- **Fallback:** если examples-файла не было — какие тематические references, аналог и official docs версии использованы.
- **Что не копировали:** если в зоне были anti-patterns второго роутера / path drift / blank gate / open redirect / client-only auth — что сделали иначе.

## Если что-то пошло не так

- Стек неясен, App+Pages в зоне равноправны или стоят два SPA-router → спроси; не подставляй React Router или App Router наугад и не мигрируй стек.
- Нет examples-файла → тематический reference + локальный аналог + official docs установленной версии; стек не переключай.
- Нет аналога → ищи соседний маршрут **этого** репо; чужие импорты, alias и product path не копируй.
- Пользователь не просил новый роутер → не добавляй второй и не ставь `react-router` / `next` «для удобства».
- Нужно только размещение page-модуля → это [`frontend-architecture`](../frontend-architecture/SKILL.md), не этот скил.
- Нужны HTTP, DTO, cache или endpoint params без client route → это [`frontend-api`](../frontend-api/SKILL.md), не этот скил.
- Нужен leave-guard формы → это [`frontend-forms`](../frontend-forms/SKILL.md), не этот скил.
- Нужна только вёрстка экрана / skeleton / CSS → это [`frontend-layout`](../frontend-layout/SKILL.md), не этот скил.
- Нужен аудит open redirect / client-only authorization → это [`frontend-security`](../frontend-security/SKILL.md), не этот скил.
- Scope ушёл в соседнюю область → вернись к таблице маршрутов; client guard не выдавай за server authorization.
