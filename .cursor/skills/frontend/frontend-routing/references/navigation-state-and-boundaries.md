# Навигация, URL-state и границы

Читай после выбора трека, когда нужны links, history, params/search/hash, nested layouts, 404/error/loading, lazy/prefetch, blockers/guards, scroll/focus/title. Синтаксис API — в [react-router-rules.md](react-router-rules.md) или [next-routing-rules.md](next-routing-rules.md) своего трека, не объявляй latest универсальным.

Визуал loading/error/skeleton, CSS nested layout и сетка экрана — [`frontend-layout`](../../frontend-layout/SKILL.md). Этот файл — поведение маршрута, history и a11y **после client navigation**.

Навигацию, params, return URL и PII **не логируй**.

## 1. Link, navigate, redirect, replace

| Намерение | Чем выражать | History |
| --- | --- | --- |
| Пользователь идёт в новое место (пункт меню, карточка, «открыть») | `Link` / `NavLink` / `next/link` — первичный способ | push |
| Программный переход после жеста, которого нет в ссылке (wizard step, закрыть модалку в маршрут) | `navigate` / `router.push` трека | push, если это новое место |
| Коррекция URL, login success, post-submit, канонизация trailing slash, «уже здесь» | `navigate(..., { replace: true })` / `router.replace` / `redirect` трека | **replace** — не копи entry |
| Внешний origin, download, якорь, статический файл | обычный `<a>` | не router SPA |

`window.location.assign` / `href` / полная перезагрузка — не замена router API внутри приложения. Исключение: зона уже так выходит на другой origin или сбрасывает весь client state сознательно.

Не подменяй `<Link>` кнопкой с `onClick={navigate}` для обычного перехода: ломаются открытие в новой вкладке, prefetch и copy URL. Обратно: не ставь `<a href="/catalog">` там, где соседи используют router `Link` — потеряется client navigation.

Redirect target и return URL — только same-origin relative path по аналогу зоны. Произвольный абсолютный / protocol-relative URL из query не принимай: это open redirect → [`frontend-security`](../../frontend-security/SKILL.md). Минимум: после `decodeURIComponent` path начинается с `/`, не с `//`, без `\`, без `http:`/`javascript:`. Это не allowlist; полный разбор — frontend-security.

## 2. Params, search, hash, URL-state

Владелец route-level URL-state — маршрут, не локальный `useState` фильтра, пока значение не пишется в URL.

| Часть URL | Кто читает/пишет | Не путать с |
| --- | --- | --- |
| Path params (`:itemId`, `[itemId]`) | router API зоны (`useParams`, props `params`) | id, который живёт только в props родителя и не совпадает с URL |
| Search / query | `useSearchParams` / `searchParams` трека; запись через router, не ручной `history.pushState`, если соседи уже на router API | фильтр в `useState`, который сбрасывается при refresh |
| Hash | `useLocation().hash` / `location.hash` — как у соседей | in-page якорь без смены маршрута (не этот скил, если route behavior не меняется) |

Сохраняй search и hash при соседней навигации, если ближайшие маршруты той же зоны их сохраняют (пагинация списка → карточка → назад). Не сбрасывай query «на всякий случай». Собирая URL строкой (`pathname + search`), дописывай hash из `window.location.hash` / принятого API, если соседи hash держат. Не тащи все фильтры в path params, если зона держит их в search.

Локальный поиск без записи в URL — [`frontend-layout`](../../frontend-layout/SKILL.md), не начинай этот скил. Когда фильтр **становится** URL-state, пиши его тем же search-контрактом, что у соседнего списка.

Path helper / каталог путей зоны важнее захардкоженной строки: drift ломает deep links и тесты. Нет helper — повтори литерал **как в route tree**, не выдумывай второй канон. `basename` / `basePath` — в конфиге роутера или `next.config`, не префикс в каждом `to`/`href`.

HTTP path/query params endpoint без client route — [`frontend-api`](../../frontend-api/SKILL.md).

## 3. Nested layouts

Вложенный layout — convention трека (`Outlet`, `layout.tsx`, эквивалент установленного router), не CSS-обёртка «как будто маршрут».

- Общий chrome (nav, shell) остаётся смонтированным при смене child route.
- Child рендерится в штатную точку (`Outlet` / `children` layout), не вторым деревом рядом.
- Visual grid / padding shell — [`frontend-layout`](../../frontend-layout/SKILL.md); файл маршрута не превращай в styled-layout без route-роли.

Не подменяй nested route условным `{pathname.startsWith(...) && <Child />}` внутри родителя, если у трека уже есть child routes.

## 4. 404, error, loading — route-level

Пустой экран и вечный blank gate — не 404 и не loading.

| Состояние | Где живёт | Что не делать |
| --- | --- | --- |
| Нет маршрута | splat / `notFound` / `not-found` convention трека | «тихий» `return null` на известном path |
| Ошибка маршрута / loader | `errorElement` / `error.tsx` / Error Boundary **этого сегмента** | ронять всё дерево на ошибке child |
| Ожидание перехода или данных маршрута | `loading.tsx` / `useNavigation().state` / принятый pending зоны | размонтировать layout и показывать пустой экран на каждый click |

Loading обязан оставлять ориентир (оставшийся chrome, status для AT — §7). Как **выглядит** skeleton — layout. Здесь — чтобы маршрут не терял shell и не блокировал фокус в никуда.

Данные маршрута, которые ходят в HTTP, не дублируй вторым fetch-слоем здесь: [`frontend-api`](../../frontend-api/SKILL.md). Этот скил только вплетает уже принятый loader/page data в route API.

## 5. Lazy и prefetch

Code-split маршрута — только тем API, которым зона уже режет routes (`lazy()` data router, `React.lazy` + declarative, файловый split Next, `next/link` prefetch). Не добавляй второй bundler-plugin «для скорости».

Prefetch:

- Next `Link` префетчит по умолчанию в App Router — не выключай без причины и не включай глобально `prefetch={true}` на каждую ссылку в зоне, где соседи ограничивают.
- React Router: prefetch API есть не во всех modes (см. таблицу режима в docs установленной версии). Не подставляй framework-only prefetch в declarative.

Не prefetch недоверенный URL из query.

## 6. Blockers и guards

Это **разные** вещи.

| Механизм | Смысл | Владелец |
| --- | --- | --- |
| Leave-blocker | «форма грязная — подтверди уход» | [`frontend-forms`](../../frontend-forms/SKILL.md); API `useBlocker` / `usePrompt` / `beforeunload` **уже принятый** зоной |
| Route guard | UX: не пускать в экран без сессии/роли, показать login/forbidden | этот скил + session-модуль зоны; **не** authorization |
| Server / endpoint auth | доступ к данным и мутациям | [`frontend-api`](../../frontend-api/SKILL.md) / сервер; аудит client-only — [`frontend-security`](../../frontend-security/SKILL.md) |

`useBlocker` / `usePrompt` доступны не во всех modes и не во всех major React Router. Не добавляй data-only blocker в declarative-зону, где его нет в экспорте установленной версии. Не пиши второй `useBlocker`, если в зоне уже есть leave-guard примитив.

Guard на клиенте:

- читает уже существующий session/feature flag;
- делает `replace` на login/forbidden **с** same-origin return path;
- не считает скрытую кнопку и `if (!session) return null` защитой данных.

**Client guard ≠ server authorization.** Секретный список, который уже пришёл в бандл, guard не защищает. Не выдавай `middleware` / `proxy` / `Navigate` за проверку прав на API.

## 7. Route-transition UX и доступность

После **client** navigation (не полной перезагрузки) восстанови контекст чтения. Визуал — layout; поведение ниже — этот скил. Системный a11y/WCAG-аудит — [`frontend-a11y`](../../frontend-a11y/SKILL.md); scroll / focus / title после client navigation при работе над маршрутом остаются здесь.

**Scroll.** Восстанавливай по контракту зоны: `ScrollRestoration` (data/framework React Router), default Next App, ручной scroll-to-top на смене pathname, если так делают соседи. Deep link с hash — прокрути к цели, если зона это делает. Не скролль на каждый search-param, если меняется только фильтр того же экрана. Не оставляй пользователя внизу длинной страницы после перехода на новый экран.

**Focus.** После смены маршрута фокус не должен оставаться на размонтированной ссылке или улетать в `document.body` без ориентира. Перенеси его на главное содержимое сегмента: `main` / заголовок `h1` с `tabIndex={-1}` / принятый skip-target зоны. Не делай `autoFocus` на первый input списка при каждом filter-navigation, если это не новый экран. Не открывай модалку фокусом, если маршрут не про модалку.

**Document title.** Каждый осмысленный экран задаёт `document.title` (или `Meta` / `generateMetadata` / helmet-аналог зоны). Title отражает ресурс (`Catalog · Item`, `Sign in`), не одно имя приложения на всех client routes. Сверяй API с треком: framework React Router `Meta`, Next `metadata` / `generateMetadata`, SPA без них — тот helper, что уже есть. Не ставь сырой id из URL в title, если соседи подставляют человекочитаемое имя после загрузки — но не блокируй title пустым экраном (§4).

**Status для AT.** Смена страницы в SPA не всегда объявляется сама. Если в зоне есть live region / route announcer — переиспользуй. Нет — не изобретай второй toast-logger; минимум: фокус на `h1` и обновлённый title. Pending-переход не должен оставлять AT в тишине надолго: route-level loading (§4) лучше пустого DOM.

Не добавляй `console.*` и telemetry с pathname/search/PII «для отладки навигации».

## 8. Границы соседних скилов

| Задача | Куда |
| --- | --- |
| Размещение page-файла, public API слайса | [`frontend-architecture`](../../frontend-architecture/SKILL.md) |
| Loader/page fetch, DTO, cache, retry | [`frontend-api`](../../frontend-api/SKILL.md) |
| Dirty form leave-guard, submit | [`frontend-forms`](../../frontend-forms/SKILL.md) |
| Skeleton, CSS shell, сетка | [`frontend-layout`](../../frontend-layout/SKILL.md) |
| Open redirect, client-only auth как «защита» | [`frontend-security`](../../frontend-security/SKILL.md) |
| «проверь доступность» / WCAG / системный a11y | [`frontend-a11y`](../../frontend-a11y/SKILL.md) |

Reuse **не** значит копировать дыры: hardcoded path drift, смешение relative/absolute children, blank loading gate, open redirect, client-only auth, `window.location` вместо router, server `redirect` из client-only модуля (и наоборот).
