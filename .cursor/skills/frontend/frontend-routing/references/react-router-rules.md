# React Router

Читай **только** если трек зоны — React Router **v6/v7**. Mode (framework / data / declarative) уже выбран в [discovery-and-stack.md](discovery-and-stack.md). v5 — этот файл не применяй, только docs установленной major. Next examples и `next/navigation` не применяй.

Синтаксис сверяй с **установленной major** (`react-router-dom` v6 ≠ `react-router` v7 ≠ v5). Сниппеты ниже — иллюстрация контракта, не канон импортов чужого репо. Полные GOOD/BAD пары — своего mode: data/framework — `examples-react-router.md`; declarative — `examples-react-router-declarative.md`. v5 — examples не читай, только docs установленной major.

Loaders/actions, framework route modules, `useBlocker` — не default: только при локальном прецеденте или явном запросе **и** если mode/версия это экспортируют.

Runtime-логи navigation/params не добавляй.

## 1. Route tree — один, уже существующий

Новый экран подключай в **текущий** tree зоны. Второй `createBrowserRouter` / вложенный `BrowserRouter` «для фичи» не заводи.

| Mode | Где живёт дерево | Точка монтирования UI |
| --- | --- | --- |
| framework | `routes.ts` (или принятый route config) + route modules | layout-модуль + `<Outlet />` / эквивалент версии |
| data | объекты `RouteObject` / `createRoutesFromElements` рядом с `createBrowserRouter` | `element` / `Component` родителя + `<Outlet />` |
| declarative | `<Routes>` / `<Route>` под `BrowserRouter` | то же: родитель с `<Outlet />` |

Имена полей сверяй с версией: v6 data часто `element: <Page />`; v7 docs чаще `Component: Page`. Повтори **локальный** стиль, не переписывай дерево под статью.

v5 (`Switch`, `exact`, `component`, `useHistory`, `<Redirect>`) — следуй docs установленной v5. `examples-react-router.md` и `examples-react-router-declarative.md` **не** читай: там контракт v6/v7. Не подставляй v6 `element` / `useNavigate` / `<Routes>` в v5-зону без запроса на миграцию.

```tsx
// иллюстрация data-контракта, не импорты конкретного репо
{
  path: 'catalog',
  element: <CatalogLayout />,
  children: [
    { index: true, element: <CatalogListPage /> },
    { path: ':itemId', element: <CatalogItemPage /> },
  ],
}
```

Declarative JSX (`<Routes>` / `<Route>`) — в `examples-react-router-declarative.md`, не копируй этот объект в `BrowserRouter`.

Path helper зоны (`generatePath`, локальный каталог путей) важнее разбросанных литералов. Нет helper — литерал должен совпадать с `path` в tree.

Если зона уже задаёт `basename` на `BrowserRouter` / `createBrowserRouter` — повтори его **в этом конфиге**, не в каждом `to`. Литерал `/catalog` остаётся path дерева: API сам префиксует. Не пиши `/app/catalog` в `Link` при `basename="/app"` (двойной префикс). Сырой `<a href="/catalog">` префикс обойдёт.

## 2. Relative vs absolute children

В одном родителе не смешивай relative и absolute children без прецедента в **этом** tree.

- Relative child (`path: ':itemId'`) строится на parent path → `/catalog/:itemId`.
- Absolute child (`path: '/catalog/:itemId'`) обязан начинаться с полного path родителя; иначе маршрут недостижим.
- Index route — `index: true` / `<Route index>`, не `path=""`, если соседи уже на `index`.
- Splat (`*`, `path: '*'`) — как у соседей; relative splat в v6/v7 зависит от flags/версии (`future.v7_relativeSplatPath` и аналоги). Сверь docs **установленной** версии, не копируй latest flag в старый проект.

`Link to` без ведущего `/` резолвится относительно route hierarchy (`relative="route"` по умолчанию). `to=".."` поднимается на **родительский route**, не обязательно на один URL-сегмент. Нужен подъём на один сегмент URL — `relative="path"`, и только если версия это поддерживает. Не чини «сломанный назад» абсолютным `/` вперемешку с relative siblings.

Layout- и index-маршруты без собственного path не участвуют в расчёте `..`. Не клади `path: 'catalog'` и child `path: 'catalog/new'` как relative sibling — получится `/catalog/catalog/new`.

## 3. Outlet и nested layout

Родитель, у которого есть children, обязан рендерить `<Outlet />` (или принятый wrapper зоны). Иначе child никогда не появится.

```tsx
function CatalogLayout() {
  return (
    <>
      <CatalogNav />
      <Outlet />
    </>
  );
}
```

Не рендерь child вручную по `useParams` рядом с незаполненным `Outlet`. Не подменяй `Outlet` CSS-колонкой без маршрута — визуал колонок: [`frontend-layout`](../../frontend-layout/SKILL.md).

`errorElement` / `ErrorBoundary` на сегменте ловит ошибку **этого** поддерева. Глобальный boundary корня не заменяет сегментный, если зона уже режет ошибки по веткам.

## 4. Loaders, actions, pending — только с прецедентом

| Mode | Data API | Правило |
| --- | --- | --- |
| framework / data, и соседи уже грузят в `loader`/`clientLoader`/`action` | используй тот же слой | HTTP внутри loader — [`frontend-api`](../../frontend-api/SKILL.md); здесь только вплетение в route |
| data router без loader в зоне | не добавляй loader «для правильности» | данные — существующий query-слой |
| declarative | `loader`/`useLoaderData` нет | не переводи зону на data ради fetch |

`useNavigation().state` (`idle` / `loading` / `submitting`) — pending **маршрута** data/framework, не CSS spinner. Blank gate на время `loading` запрещён: [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md).

`redirect` / `replace` из loader/action — server/data API этих modes. Из client-компонента для жеста пользователя — `navigate` / `<Navigate>`. Не путай.

## 5. Навигация и URL API своего mode

Общее для v6/v7 data и declarative (если экспорт есть): `Link`, `NavLink`, `Navigate`, `useNavigate`, `useLocation`, `useParams`, `useSearchParams`, `Outlet`. Это **не** список v5: там `useHistory`, `Switch`, `<Redirect>`, `component={}` — docs установленной v5.

Только data/framework (проверь таблицу версии): `useLoaderData`, `useFetcher`, `useNavigation`, `useBlocker`, `usePrompt`, `ScrollRestoration`, `redirect` из `react-router` / `react-router-dom`.

- Push vs replace — [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md) §1.
- Search params: штатный `useSearchParams` (кортеж `[params, setParams]`). Не собирай query вручную через `navigate(\`?${s}\`)`, если соседи уже на `setParams`. Сохраняй несвязанные ключи, когда меняешь один фильтр.
- `<Navigate to="..." replace />` для guard/канонизации, не для пункта меню.

Guard: `<Navigate to={signInPath} replace state={{ from: location }} />` (или принятый return-URL контракт зоны). `from` — relative path, не `window.location.href`. Разбор return URL — §1 navigation-state (same-origin).

`useBlocker` — leave-guard формы: [`frontend-forms`](../../frontend-forms/SKILL.md). В declarative его может не быть: не полифиль data API.

## 6. Hosting fallback

Browser / data router без SSR: сервер и CDN обязаны отдавать **тот же SPA entry** для deep link (`/catalog/42`), иначе refresh даёт HTTP 404 до React.

- Повтори уже существующий rewrite (`try_files`, `historyApiFallback`, hosting `rewrites`).
- Не выдумывай nginx/vercel конфиг, если его нет в репо и пользователь не просил.
- Hash-router fallback не требует path rewrite — но не переключай на hash вместо настройки хостинга без запроса.

Framework mode / SSR: entry и headers — как у принятого деплоя зоны, не копируй SPA-rewrite в SSR-хостинг вслепую.

404 **внутри** приложения (splat, `errorElement` со статусом) ≠ 404 хостинга. Сначала должен загрузиться router.

## 7. Тестовый harness

Используй уже принятый в зоне способ. Новый test runner не добавляй.

| Что уже есть | Что делать |
| --- | --- |
| `createMemoryRouter` + `RouterProvider` | тот же tree (или урезанный stub **этого** tree), `initialEntries` с нужным path/search |
| `MemoryRouter` + `<Routes>` | declarative harness; не подставляй `createMemoryRouter` «потому что современнее» |
| `createRoutesStub` (если версия экспортирует) | stub loader/action соседей, не мокай весь пакет вслепую |
| Storybook decorator с `BrowserRouter` / `MemoryRouter` | повтори decorator зоны |

Тест проверяет route behavior (куда ушли, какой param, 404), не вёрстку. Не мокай `useNavigate` глобально, если можно обернуть harness. Не логируй `location` в тестах «для отладки», если зона этого не делает.

## 8. Чего не делать на этом треке

- Не ставить второй роутер и не оборачивать feature в свой `BrowserRouter` внутри уже существующего.
- Не переводить declarative ↔ data ↔ framework без явного запроса.
- Не добавлять `loader`/`action`, пока их нет в зоне.
- Не смешивать relative и absolute children в одном родителе без прецедента.
- Не заменять `Outlet` ручным выбором child.
- Не использовать `window.location` для внутренней навигации.
- Не выдавать client guard за authorization.
- Не копировать v7 import `from 'react-router'` в проект, где стоит только `react-router-dom` v6 (и наоборот).
