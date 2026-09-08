# Примеры: React Router declarative

Читай **только этот файл**, если mode зоны — React Router **declarative** (`BrowserRouter` + `<Routes>` / `<Route>` без data router). Data/framework — [examples-react-router.md](examples-react-router.md). v5 — этот файл не открывай, только docs установленной major. `examples-next-*.md` не открывай.

Сниппеты — обобщённые рецепты. Импорт (`react-router-dom` v6 vs `react-router` v7) бери у **соседей этой зоны** и установленной major. Не копируй `@/`, имена слайсов и product path. Нейтральные учебные пути вроде `src/routes.tsx` допустимы как иллюстрация. Runtime-логи navigation, params, return URL и PII не добавляй.

Не переводи зону на data/framework «заодно»: без `loader`/`action`, `errorElement`, `useNavigation`, `createBrowserRouter`, `createMemoryRouter`. `useBlocker` — только если версия его экспортирует **и** это не form leave-guard ([`frontend-forms`](../../frontend-forms/SKILL.md)).

Правила дерева, Outlet и hosting — в [react-router-rules.md](react-router-rules.md). History, URL-state, guards, 404/loading и a11y — в [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md). Сюда их не дублируй целиком.

Как читать пары: сверху **надо** (GOOD), ниже **не надо** (BAD), затем почему. Копируй JSX-дерево, не объекты `RouteObject`.

## 1. Декларация маршрута

Один существующий tree. Новый экран — sibling/child в нём. Path helper зоны важнее разбросанных литералов. Relative и absolute children в одном родителе не смешивай. `basename` — проп `BrowserRouter` зоны, не префикс в каждом `to`.

```tsx
// надо — один JSX-tree; relative children; path как у helper зоны
const PATHS = {
  catalog: '/catalog',
  catalogNew: '/catalog/new',
  catalogItem: (itemId: string) => `/catalog/${itemId}`,
  signIn: '/sign-in',
} as const;

function AppRoutes() {
  return (
    <Routes>
      <Route path="catalog" element={<CatalogLayout />}>
        <Route index element={<CatalogListPage />} />
        <Route path="new" element={<CatalogNewPage />} />
        <Route path=":itemId" element={<CatalogItemPage />} />
      </Route>
    </Routes>
  );
}
```

Не заводи второй `BrowserRouter` внутри фичи и не копируй `RouteObject` / `createBrowserRouter` из data-examples.

```tsx
// не надо — второй роутер; data-объект; hardcoded drift; смесь relative/absolute
function CatalogFeature() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/catalog/:itemId" element={<CatalogItemPage />} />
      </Routes>
    </BrowserRouter>
  );
}

const catalogRoute = {
  path: 'catalog',
  element: <CatalogLayout />,
  children: [
    { path: ':itemId', element: <CatalogItemPage /> },
    { path: '/settings', element: <SettingsPage /> },
  ],
};

<Link to="/catlog/42">Typo</Link>
```

**Почему.** Вложенный `BrowserRouter` сбрасывает history внешнего дерева. Объект `RouteObject` без `RouterProvider` — чужой mode. Absolute child `/settings` не продолжает `/catalog`. Опечатка `/catlog` разъедется с tree.

**Как исправить.** Один router на приложение. Children — все relative **или** все absolute, как уже в этом родителе. Ссылки — через `PATHS` / `generatePath` зоны. Data-router не добавляй.

## 2. Nested layouts

Родитель с children рендерит `<Outlet />`. Chrome остаётся смонтированным. Не подменяй child ручным `pathname` и не выдавай CSS-колонку за маршрут.

```tsx
// надо
function CatalogLayout() {
  return (
    <>
      <CatalogNav />
      <Outlet />
    </>
  );
}
```

```tsx
// не надо — ручной child; пустой Outlet; CSS «как будто route»
function CatalogLayout() {
  const { pathname } = useLocation();
  return (
    <div className="grid-two-columns">
      <CatalogNav />
      {pathname.includes('/new') ? <CatalogNewPage /> : <CatalogListPage />}
    </div>
  );
}
```

**Почему.** Ручной `includes` расходится с tree: param, index и splat не работают, layout размонтирует «лишнее». Сетка без `<Outlet />` — визуал, не nested route ([`frontend-layout`](../../frontend-layout/SKILL.md)).

**Как исправить.** Children объяви в `<Routes>`. В родителе — только chrome + `<Outlet />`. Сетку колонок оставь layout-скилу.

## 3. History: push vs replace

Пункт меню и карточка — `<Link>` (push). Коррекция URL, login success, канонизация — `replace`. `window.location` — не внутренняя навигация.

```tsx
// надо
<Link to={PATHS.catalogItem(item.id)}>Open</Link>

function onSignInSuccess(returnPath: string) {
  navigate(returnPath, { replace: true });
}

function onCloseDrawerToItem(itemId: string) {
  navigate(PATHS.catalogItem(itemId));
}
```

```tsx
// не надо — перезагрузка; кнопка вместо Link; push на login
<button onClick={() => navigate(PATHS.catalog)}>Catalog</button>
window.location.href = PATHS.catalogItem(id);
navigate(PATHS.signIn);
```

**Почему.** `<button onClick={navigate}>` ломает новую вкладку и copy URL. `window.location` сбрасывает client state. Push на `/sign-in` оставляет защищённый URL в history: «назад» вернёт на закрытый экран.

**Как исправить.** Обычный переход — `<Link>` / `<NavLink>`. Login/канонизация — `navigate(..., { replace: true })` или `<Navigate replace />`. Внутренние URL — только router API своей версии.

## 4. Params и search params

Path param читается из router API, не из props, которые могут разойтись с URL. Фильтр списка, который должен переживать refresh, — search params. Несвязанные ключи сохраняй.

```tsx
// надо
function CatalogItemPage() {
  const { itemId } = useParams();
  if (!itemId) return <UnknownItemState />;
  return <CatalogItemBody itemId={itemId} />;
}

function CatalogFilters() {
  const [params, setParams] = useSearchParams();
  const q = params.get('q') ?? '';

  function onQueryChange(next: string) {
    const nextParams = new URLSearchParams(params);
    if (next) nextParams.set('q', next);
    else nextParams.delete('q');
    setParams(nextParams, { replace: true });
  }

  return <input value={q} onChange={(e) => onQueryChange(e.target.value)} />;
}
```

`<Link to={PATHS.catalogItem(id)}>` с текущего списка — сохрани search/hash, **если** соседи так делают (`<Link to={{ pathname, search, hash }}>` или relative `to` без сброса). Не тащи фильтры в path, если зона держит их в query.

```tsx
// не надо — фильтр только в useState; ручной navigate-query; потеря чужих ключей
const [q, setQ] = useState('');
navigate(`?q=${q}`);
setParams({ q: next });
```

**Почему.** `useState` сбрасывается на refresh и не шарится как deep link. `` navigate(`?q=`) `` затирает hash и другие ключи и обходит `useSearchParams`. Объект `{ q }` в `setParams` **заменяет** весь search.

**Как исправить.** Штатный `useParams` / `useSearchParams`. Копируй текущие `URLSearchParams`, меняй один ключ. Локальный поиск без URL — не этот скил ([`frontend-layout`](../../frontend-layout/SKILL.md)).

## 5. Redirect и return URL

Return path — same-origin relative (`/` да, `//` нет). Произвольная строка из query — не `to` навигатору. Хелпер — минимум, не allowlist.

```tsx
// надо
function sameOriginPath(raw: string | null, fallback: string): string {
  if (!raw) return fallback;
  let path = raw;
  try {
    path = decodeURIComponent(raw);
  } catch {
    return fallback;
  }
  if (!path.startsWith('/') || path.startsWith('//') || path.includes('\\')) return fallback;
  if (/^[a-zA-Z][a-zA-Z+.-]*:/.test(path)) return fallback;
  return path;
}

function SignInPage() {
  const [params] = useSearchParams();
  const location = useLocation();
  const fromState = (location.state as { from?: { pathname?: string } } | null)?.from?.pathname;
  const returnPath = sameOriginPath(params.get('from') ?? fromState ?? null, PATHS.catalog);

  function onSuccess() {
    navigate(returnPath, { replace: true });
  }
}
```

```tsx
// не надо — open redirect; абсолютный href из query
const next = searchParams.get('next');
navigate(next ?? '/');
window.location.assign(next!);
```

**Почему.** `?next=https://evil.example` (и `//evil.example`, `javascript:`) уводит пользователя с origin. Это не «гибкий return URL», а open redirect → [`frontend-security`](../../frontend-security/SKILL.md).

**Как исправить.** После decode path должен начинаться с `/` и не начинаться с `//`, без `\`. Fallback — известный внутренний маршрут. Полный разбор — [`frontend-security`](../../frontend-security/SKILL.md). `window.location.href` / полный URL из query не используй.

## 6. Guards

Client guard — UX: `replace` на sign-in **с** same-origin return path. Это не authorization. Секреты, которые уже в бандле, скрытая кнопка не защищает.

```tsx
// надо — UX-ограничение; данные по-прежнему за session/API
function RequireSession({ children }: { children: ReactNode }) {
  const session = useSession();
  const location = useLocation();

  if (!session) {
    return (
      <Navigate
        to={PATHS.signIn}
        replace
        state={{ from: location }}
      />
    );
  }

  return children;
}
```

Список на этом экране всё равно грузится через принятый API-слой с credentials ([`frontend-api`](../../frontend-api/SKILL.md)). Guard не заменяет 401/403 сервера.

```tsx
// не надо — client-only auth как «защита»; пустой экран; data-only blocker
function SecretPage({ secretList }: { secretList: Item[] }) {
  const token = localStorage.getItem('token');
  if (!token) return null;
  return <pre>{JSON.stringify(secretList)}</pre>;
}

useBlocker(() => !window.confirm('Leave?'));
```

**Почему.** `return null` оставляет URL и данные в памяти/бандле: это не доступ. `secretList` в пропах уже утекли. `useBlocker` для грязной формы принадлежит [`frontend-forms`](../../frontend-forms/SKILL.md); в declarative его может не быть — не полифиль data API.

**Как исправить.** `<Navigate replace>` + return path. Данные — только с API, который проверяет сессию. Form leave-guard не пиши в этом скиле.

## 7. Loading, error, 404

Нет маршрута — splat / принятый 404-элемент. Ошибка сегмента — Error Boundary **как у соседей**, не `errorElement` data router. Loading не размонтирует chrome. Pending — query `isPending` страницы, не `useNavigation`.

```tsx
// надо — splat 404; pending не blank; без data API
<Route path="catalog" element={<CatalogLayout />}>
  <Route index element={<CatalogListPage />} />
  <Route path=":itemId" element={<CatalogItemPage />} />
  <Route path="*" element={<NotFoundPage />} />
</Route>

function CatalogListPage() {
  const { isPending } = useCatalogQuery();
  return (
    <>
      {isPending ? <p role="status">Loading</p> : null}
      <CatalogList />
    </>
  );
}
```

```tsx
// не надо — errorElement / useNavigation / loader; blank gate
const routes = [
  {
    path: 'catalog',
    element: <CatalogLayout />,
    errorElement: <CatalogErrorPage />,
    loader: catalogLoader,
  },
];

function CatalogItemPage() {
  const navigation = useNavigation();
  const loaderItem = useLoaderData() as Item;
  if (navigation.state === 'loading') return null;
  return <ItemView item={loaderItem} />;
}
```

**Почему.** `errorElement` / `useNavigation` / `useLoaderData` в declarative нет или молча не работают: это миграция на data. `return null` на loading — blank gate.

**Как исправить.** Splat 404 в JSX-tree. Pending оставляет layout + status из query-слоя зоны. HTTP — [`frontend-api`](../../frontend-api/SKILL.md), не loader «заодно».

## 8. Scroll и focus после client navigation

После смены **экрана** восстанови scroll по контракту зоны и перенеси focus на главное содержимое. Title — у экрана, не одно имя приложения. Не логируй pathname. `<ScrollRestoration />` — data/framework, сюда не копируй без прецедента.

```tsx
// надо
function CatalogItemPage() {
  const headingRef = useRef<HTMLHeadingElement>(null);
  const { itemId } = useParams();

  useEffect(() => {
    headingRef.current?.focus();
  }, [itemId]);

  return (
    <main>
      <h1 ref={headingRef} tabIndex={-1}>
        Item
      </h1>
    </main>
  );
}
```

Смена только `q=` на том же списке — не scroll-to-top и не `autoFocus` на первый input, если соседи так не делают.

```tsx
// не надо — фокус на body; scroll на каждый filter; сырой id в title
useEffect(() => {
  document.title = window.location.pathname;
  window.scrollTo(0, 0);
}, [searchParams.toString()]);
```

**Почему.** Filter-navigation скроллит длинный список вверх без нужды. Title из сырого pathname хуже `generatePath`/helper. Фокус, оставшийся на размонтированной ссылке, теряется для клавиатуры.

**Как исправить.** Scroll-to-top **на смену pathname**, как у соседей. Focus — `h1` / `main` с `tabIndex={-1}`. Title — принятый helper экрана.

## 9. Тестовый harness

Используй уже принятый способ зоны. Новый runner не добавляй. Проверяй URL, param и 404, не вёрстку. Не мокай весь пакет, если можно обернуть tree. `createMemoryRouter` — data API, сюда не подставляй.

```tsx
// надо — MemoryRouter, если зона на нём
render(
  <MemoryRouter initialEntries={['/catalog/42']}>
    <Routes>
      <Route path="catalog" element={<CatalogLayout />}>
        <Route path=":itemId" element={<CatalogItemPage />} />
      </Route>
    </Routes>
  </MemoryRouter>,
);
```

```tsx
// не надо — глобальный mock navigate; createMemoryRouter «потому что современнее»
const router = createMemoryRouter(routes, { initialEntries: ['/catalog/42'] });
render(<RouterProvider router={router} />);

jest.mock('react-router-dom', () => ({
  ...jest.requireActual('react-router-dom'),
  useNavigate: () => mockNavigate,
  useParams: () => ({ itemId: '42' }),
}));
```

**Почему.** Глобальный mock расходится с реальным relative `to`, splat и `replace`. `createMemoryRouter` переводит тест на data-режим, которого нет в зоне. Харкод `useParams` не ловит drift дерева.

**Как исправить.** `MemoryRouter` + `<Routes>` как у соседей. Клик по `<Link>` и assert URL/UI, завязанный на param. Hosting fallback тестируй только если такой конфиг уже в репо.
