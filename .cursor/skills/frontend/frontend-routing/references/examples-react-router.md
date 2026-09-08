# Примеры: React Router data / framework

Читай **только этот файл**, если mode зоны — React Router **data** или **framework**. Declarative — [examples-react-router-declarative.md](examples-react-router-declarative.md). v5 — этот файл не открывай, только docs установленной major. `examples-next-*.md` не открывай.

Сниппеты — обобщённые рецепты. Импорт (`react-router-dom` v6 vs `react-router` v7) и поля `element` / `Component` бери у **соседей этой зоны** и установленной major, не из «latest docs». Не копируй `@/`, имена слайсов и product path. Нейтральные учебные пути вроде `src/routes.tsx` допустимы как иллюстрация. Runtime-логи navigation, params, return URL и PII не добавляй.

Mode уже выбран. Не переводи зону на declarative «заодно» и не подмешивай `next/navigation`. Loaders/actions, `useBlocker`, `ScrollRestoration` — только если mode/версия их экспортируют **и** в зоне уже есть прецедент (или явный запрос).

Правила дерева, Outlet и hosting — в [react-router-rules.md](react-router-rules.md). History, URL-state, guards, 404/loading и a11y — в [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md). Сюда их не дублируй целиком.

Как читать пары: сверху **надо** (GOOD), ниже **не надо** (BAD), затем почему. Копируй только data/framework API. `BrowserRouter`+`<Routes>` из другого файла сюда не тащи.

## 1. Декларация маршрута

Один существующий tree. Новый экран — sibling/child в нём. Path helper зоны важнее разбросанных литералов. Relative и absolute children в одном родителе не смешивай.

```tsx
// надо — один tree; relative children; path как у helper зоны
// файл вроде src/routes.tsx (имя — как у соседей)
const PATHS = {
  catalog: '/catalog',
  catalogNew: '/catalog/new',
  catalogItem: (itemId: string) => `/catalog/${itemId}`,
  signIn: '/sign-in',
} as const;

const catalogRoute = {
  path: 'catalog',
  element: <CatalogLayout />,
  children: [
    { index: true, element: <CatalogListPage /> },
    { path: 'new', element: <CatalogNewPage /> },
    { path: ':itemId', element: <CatalogItemPage /> },
  ],
};
```

Declarative JSX-дерево — не этот файл. Framework-зона — route module / `routes.ts` **этого** каркаса, не копируй data-объекты «потому что привычнее», если каркас уже на modules.

```tsx
// не надо — второй роутер; hardcoded drift; смесь relative/absolute; catalog/catalog/new
function CatalogFeature() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/catalog/:itemId" element={<CatalogItemPage />} />
      </Routes>
    </BrowserRouter>
  );
}

const brokenParent = {
  path: 'catalog',
  children: [
    { path: ':itemId', element: <CatalogItemPage /> },
    { path: '/settings', element: <SettingsPage /> },
    { path: 'catalog/new', element: <CatalogNewPage /> },
  ],
};

<Link to="/catalog/42">Item</Link>
<Link to="/catlog/42">Typo</Link>
```

**Почему.** Вложенный `BrowserRouter` сбрасывает history внешнего дерева: deep link и «назад» живут в другой вселенной. Литералы `/catalog/42` и опечатка `/catlog` разъедутся с `path` в tree. Absolute child `/settings` не продолжает `/catalog`. Relative `catalog/new` под родителем `catalog` даёт `/catalog/catalog/new`.

**Как исправить.** Один router на приложение. Children — все relative **или** все absolute, как уже в этом родителе. Ссылки — через `PATHS` / `generatePath` зоны. Второй `BrowserRouter` / второй `createBrowserRouter` не заводи.

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

**Как исправить.** Children объяви в tree. В родителе — только chrome + `<Outlet />`. Сетку колонок оставь layout-скилу.

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

**Почему.** `<button onClick={navigate}>` ломает новую вкладку, copy URL и prefetch (если он есть в mode). `window.location` сбрасывает client state. Push на `/sign-in` оставляет защищённый URL в history: «назад» вернёт на закрытый экран.

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

Return path — same-origin relative (`/` да, `//` нет). Произвольная строка из query — не `to` навигатору.

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

**Почему.** `?next=https://evil.example` (и `//evil.example`, `javascript:`) уводит пользователя с origin. Это не «гибкий return URL», а open redirect → [`frontend-security`](../../frontend-security/SKILL.md). Хелпер ниже — минимум, не allowlist.

**Как исправить.** После decode path должен начинаться с `/` и не начинаться с `//`, без `\`. Fallback — известный внутренний маршрут. Это не полный разбор (двойное кодирование и т.п.) — [`frontend-security`](../../frontend-security/SKILL.md). `window.location.href` / полный URL из query не используй.

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
// не надо — client-only auth как «защита»; пустой экран; leave-guard формы здесь
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

Нет маршрута — splat / принятый 404-элемент. Ошибка сегмента — `errorElement` / boundary **этой** ветки (data/framework). Loading не размонтирует chrome и не рисует пустой DOM. Loader, который ходит в HTTP, не дублируй `useEffect`+fetch на той же странице.

```tsx
// надо — splat 404; pending не blank; данные один раз в принятом слое
const routes = [
  {
    path: 'catalog',
    element: <CatalogLayout />,
    errorElement: <CatalogErrorPage />,
    children: [
      { index: true, element: <CatalogListPage /> },
      { path: ':itemId', element: <CatalogItemPage /> },
      { path: '*', element: <NotFoundPage /> },
    ],
  },
];

function CatalogShell() {
  const navigation = useNavigation();
  return (
    <>
      <CatalogNav />
      {navigation.state === 'loading' ? <p role="status">Loading</p> : null}
      <Outlet />
    </>
  );
}
```

HTTP внутри loader — [`frontend-api`](../../frontend-api/SKILL.md).

```tsx
// не надо — blank gate; тихий null; второй fetch поверх loader
function CatalogItemPage() {
  const { itemId } = useParams();
  const loaderItem = useLoaderData() as Item;
  const [item, setItem] = useState<Item | null>(null);
  const navigation = useNavigation();

  useEffect(() => {
    fetch(`/api/items/${itemId}`).then((r) => r.json()).then(setItem);
  }, [itemId]);

  if (navigation.state === 'loading') return null;
  if (!itemId) return null;
  return <ItemView item={item ?? loaderItem} />;
}
```

**Почему.** `return null` на loading — blank gate: AT и пользователь теряют chrome и фокус. Тихий `null` на известном path маскирует баг param. Второй `fetch` расходится с loader (двойной запрос, гонки, другой error path).

**Как исправить.** Route-level 404/error. Pending оставляет layout + status. Данные — либо loader/query зоны, либо одно из двух, не оба.

## 8. Scroll и focus после client navigation

После смены **экрана** восстанови scroll по контракту зоны и перенеси focus на главное содержимое. Title — у экрана, не одно имя приложения. Не логируй pathname.

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

Data/framework: если в зоне уже есть `<ScrollRestoration />` — не пиши второй `window.scrollTo` на каждый search-param. Смена только `q=` на том же списке — не scroll-to-top и не `autoFocus` на первый input, если соседи так не делают.

```tsx
// не надо — фокус на body; scroll на каждый filter; сырой id в title вместо контракта зоны
useEffect(() => {
  document.title = window.location.pathname;
  window.scrollTo(0, 0);
}, [searchParams.toString()]);
```

**Почему.** Filter-navigation скроллит длинный список вверх без нужды. Title из сырого pathname хуже `generatePath`/helper и может уехать от экрана. Фокус, оставшийся на размонтированной ссылке, теряется для клавиатуры.

**Как исправить.** ScrollRestoration или scroll-to-top **на смену pathname**, как у соседей. Focus — `h1` / `main` с `tabIndex={-1}`. Title — принятый helper (`Meta`, helmet, эффект **экрана**).

## 9. Тестовый harness

Используй уже принятый способ зоны. Новый runner не добавляй. Проверяй URL, param и 404, не вёрстку. Не мокай весь пакет, если можно обернуть tree.

```tsx
// надо — тот же tree (или stub этого tree)
const router = createMemoryRouter(routes, {
  initialEntries: ['/catalog/42?q=oak'],
});

render(<RouterProvider router={router} />);
expect(screen.getByRole('heading', { name: /item/i })).toBeInTheDocument();
```

```tsx
// не надо — глобальный mock navigate; чужой runner; declarative MemoryRouter «потому что проще»
jest.mock('react-router-dom', () => ({
  ...jest.requireActual('react-router-dom'),
  useNavigate: () => mockNavigate,
  useParams: () => ({ itemId: '42' }),
}));
```

**Почему.** Глобальный mock расходится с реальным relative `to`, splat и `replace`. Харкод `useParams` не ловит drift дерева. Новый runner и логи `location` — запрет скила / шум с URL.

**Как исправить.** `createMemoryRouter` / `createRoutesStub` — что **уже** есть. Клик по `<Link>` и assert `router.state.location` (или UI, завязанный на param). Hosting fallback (`try_files`) тестируй только если такой конфиг уже в репо; не выдумывай nginx «для полноты». Не подставляй `MemoryRouter` в data-зону «потому что в статье проще».
