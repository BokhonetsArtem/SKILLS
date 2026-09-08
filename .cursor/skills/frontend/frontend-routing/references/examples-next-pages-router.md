# Примеры: Next.js Pages Router

Читай **только этот файл**, если трек зоны — Next Pages Router (`pages/`, `_app`, `_document`) **без** App Router в этой правке. `examples-next-app-router.md` и `examples-react-router*.md` не открывай. Не переноси экран в `app/` «заодно».

Сниппеты — обобщённые рецепты. Навигация — `next/router` + `next/link`. **`next/navigation`, `loading.tsx`, `generateMetadata`, `notFound()` сюда не копируй.** `getServerSideProps` / `getStaticProps` — только если этот тип страницы ими уже пользуется. Не копируй `@/`, имена слайсов и product path. Runtime-логи navigation, params, return URL и PII не добавляй.

`router.query` на первом client render может быть пустым — учитывай гидрацию, как соседние Pages. `middleware` на старых Pages проверяй по major: файл не доказательство, что matcher «закрыл» page.

Правила файлов и data API — в [next-routing-rules.md](next-routing-rules.md) (раздел Pages Router). History, URL-state, guards и a11y — в [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md).

Как читать пары: сверху **надо** (GOOD), ниже **не надо** (BAD), затем почему.

## 1. Декларация маршрута

Файл в `pages/` = URL. Dynamic — `[itemId].tsx`. Catch-all — только если зона уже так режет хвост. Не создавай параллельный `app/` и не ставь React Router.

```tsx
// надо — учебные пути Pages
// pages/catalog/index.tsx           → /catalog
// pages/catalog/new.tsx             → /catalog/new
// pages/catalog/[itemId].tsx        → /catalog/:itemId
// pages/sign-in.tsx                 → /sign-in

import Link from 'next/link';

export default function CatalogPage() {
  return <Link href="/catalog/new">New item</Link>;
}
```

Path helper зоны, если он есть, важнее литералов в десяти файлах. Имя param в файле и в `router.query` должно совпадать (`itemId`).

```tsx
// не надо — App-файл рядом; второй роутер; drift имени param
// app/catalog/[itemId]/page.tsx
// pages/catalog/[id].tsx
<Link href="/catlog/42">Item</Link>

import { BrowserRouter } from 'react-router-dom';
```

**Почему.** App + Pages на один URL — два канона и гонка. `[id]` vs `query.itemId` даёт `undefined` после гидрации. Опечатка в `href` не ломает сборку, ломает deep link.

**Как исправить.** Новый экран — только в существующем `pages/`. Имя `[param]` = ключ `query`. Второй router не ставь.

## 2. Nested layouts

Глобальный chrome — Custom `_app`. Повторяющийся shell раздела — общий компонент layout, который page рендерит вокруг себя (как соседи). Второго мини-роутера нет. Не жди `app/layout.tsx` и `children` сегмента.

```tsx
// надо — pages/_app.tsx + оболочка раздела
function CatalogShell({ children }: { children: React.ReactNode }) {
  return (
    <>
      <CatalogNav />
      {children}
    </>
  );
}

export default function CatalogItemPage() {
  return (
    <CatalogShell>
      <CatalogItemBody />
    </CatalogShell>
  );
}
```

```tsx
// не надо — layout.tsx App Router; ручной pathname; вложенный router
// pages/catalog/layout.tsx   ← не convention Pages
function CatalogPage() {
  const { pathname } = useRouter();
  return pathname.includes('/new') ? <CatalogNew /> : <CatalogList />;
}
```

**Почему.** `layout.tsx` в `pages/catalog/` Next Pages не вложит как App Router: это просто мёртвый файл, если его никто не импортирует. Ручной `includes` дублирует file tree и ломает `[itemId]`.

**Как исправить.** `_app` для глобального chrome. Общий `CatalogShell` импортируй в pages раздела. Вложенность URL — вложенность файлов `pages/catalog/...`, не CSS-grid «как route» ([`frontend-layout`](../../frontend-layout/SKILL.md)).

## 3. History: push vs replace

`Link` — push. Login и канонизация — `router.replace`. Программный переход — `router.push({ pathname, query })`, чтобы не потерять query-контракт зоны.

```tsx
// надо
import Link from 'next/link';
import { useRouter } from 'next/router';

<Link href={`/catalog/${item.id}`}>Open</Link>

function onSignInSuccess(returnPath: string) {
  void router.replace(returnPath);
}

function onFilter(q: string) {
  void router.replace({ pathname: '/catalog', query: { ...router.query, q }, hash: window.location.hash });
}
```

```tsx
// не надо — <a>; Pages+App хуки в одном экране; window.location; push на login
import { useRouter as useAppRouter } from 'next/navigation';
<a href="/catalog">Catalog</a>
window.location.href = `/catalog/${id}`;
void router.push('/sign-in');
```

**Почему.** Смесь `next/navigation` и `next/router` в одном экране — два источника pathname/query. `<a>` и `window.location` сбрасывают client transition Pages. Push на sign-in оставляет закрытую страницу в history.

**Как исправить.** Только `next/router` + `next/link` на этом треке. `replace` для login/фильтра того же URL. `query` мержь, не собирай `?q=` строкой вслепую.

## 4. Params и search params

`router.query.itemId` может быть `string | string[] | undefined`. На первом клиентском рендере ключа ещё нет — не читай его как обязательный id без guard. Search — те же `query`, не отдельный `useState`, если фильтр должен жить в URL.

```tsx
// надо
function readItemId(query: ParsedUrlQuery): string | undefined {
  const raw = query.itemId;
  return typeof raw === 'string' ? raw : undefined;
}

export default function CatalogItemPage() {
  const router = useRouter();
  const itemId = readItemId(router.query);

  if (!router.isReady || !itemId) {
    return <p role="status">Loading</p>;
  }

  return <CatalogItemBody itemId={itemId} />;
}

function onQueryChange(next: string) {
  const query = { ...router.query };
  if (next) query.q = next;
  else delete query.q;
  void router.replace({
    pathname: router.pathname,
    query,
    hash: window.location.hash,
  });
}
```

```tsx
// не надо — query как string без гидрации; App searchParams; фильтр только в state
const itemId = router.query.itemId.toUpperCase();
import { useSearchParams } from 'next/navigation';
const [q, setQ] = useState('');
```

**Почему.** До `isReady` `query` пустой: `.toUpperCase()` падает. `string[]` на catch-all не имеет `.toUpperCase` как у строки. `useSearchParams` — App Router. `useState` не переживает refresh.

**Как исправить.** Жди `router.isReady`, сузь тип param. Фильтры пиши в `query`. Hash передавай в UrlObject, если соседи его сохраняют. App-хуки не импортируй.

## 5. Redirect и return URL

Серверный UX-redirect — `redirect` из `getServerSideProps` / `getStaticProps` (destination — внутренний path). Клиент — `router.replace`. Сырой query `next` не является destination.

```tsx
// надо
function sameOriginPath(raw: string | undefined, fallback: string): string {
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

export const getServerSideProps: GetServerSideProps = async (ctx) => {
  const session = await readSession(ctx.req);
  if (!session) {
    const from = sameOriginPath(ctx.resolvedUrl, '/catalog');
    return {
      redirect: {
        destination: `/sign-in?from=${encodeURIComponent(from)}`,
        permanent: false,
      },
    };
  }
  return { props: {} };
};

function onSignInSuccess() {
  const returnPath = sameOriginPath(
    typeof router.query.from === 'string' ? router.query.from : undefined,
    '/catalog',
  );
  void router.replace(returnPath);
}
```

```tsx
// не надо — open redirect; App redirect(); абсолютный destination
import { redirect } from 'next/navigation';
redirect(String(ctx.query.next));
return { redirect: { destination: String(ctx.query.next), permanent: false } };
```

**Почему.** `destination: 'https://evil.example'` и `//evil.example` — open redirect → [`frontend-security`](../../frontend-security/SKILL.md). `redirect()` из `next/navigation` на Pages-треке — чужой API.

**Как исправить.** Same-origin relative path. GSSP `redirect` или `router.replace`. После decode path начинается с `/` и не с `//`, без `\`. Это минимум, не allowlist → [`frontend-security`](../../frontend-security/SKILL.md). Не принимай абсолютный URL из query.

## 6. Guards

GSSP/GST redirect — серверный UX до HTML. Client `replace` — после гидрации, если сессии ещё нет в пропах. Это не authorization: API всё равно проверяет права. Не выдавай скрытый рендер за защиту.

```tsx
// надо — серверный UX-redirect + client fallback на тот же контракт
export default function AccountPage({ hasSession }: { hasSession: boolean }) {
  const router = useRouter();

  useEffect(() => {
    if (!hasSession) {
      void router.replace({
        pathname: '/sign-in',
        query: { from: router.asPath },
      });
    }
  }, [hasSession, router]);

  if (!hasSession) {
    return <p role="status">Redirecting</p>;
  }

  return <AccountBody />;
}
```

`from: router.asPath` всё равно пропусти через `sameOriginPath` на странице sign-in. Список секретных сущностей не клади в `pageProps`, если GSSP уже знает, что сессии нет.

```tsx
// не надо — client-only auth; return null с данными в props; App layout guard
export default function SecretPage({ secretList }: { secretList: Item[] }) {
  const token = typeof window !== 'undefined' && localStorage.getItem('token');
  if (!token) return null;
  return <pre>{JSON.stringify(secretList)}</pre>;
}
```

**Почему.** `getServerSideProps` уже отдал `secretList` в HTML/JSON page props — `return null` ничего не защищает. `localStorage` не виден на сервере и не равен session cookie API. Аудит — [`frontend-security`](../../frontend-security/SKILL.md).

**Как исправить.** Без сессии — GSSP `redirect`, в props нет секрета. Client `replace` только как UX. Данные — [`frontend-api`](../../frontend-api/SKILL.md).

## 7. Loading, error, 404

`pages/404.tsx` и `pages/_error.tsx` — как у соседей. Нет `not-found.tsx` / `loading.tsx` App. Пока `!router.isReady` — status, не пустой DOM. Не дублируй GSSP клиентским `fetch` того же GET.

```tsx
// надо
export const getServerSideProps: GetServerSideProps = async (ctx) => {
  const itemId = typeof ctx.params?.itemId === 'string' ? ctx.params.itemId : undefined;
  if (!itemId) return { notFound: true };
  const item = await getItem(itemId, ctx.req); // HTTP — frontend-api
  if (!item) return { notFound: true };
  return { props: { item } };
};

export default function CatalogItemPage({ item }: { item: Item }) {
  return <CatalogItemBody item={item} />;
}
```

```tsx
// не надо — blank; App notFound(); второй fetch; client push /404
import { notFound } from 'next/navigation';

export default function CatalogItemPage() {
  const { query, isReady, push } = useRouter();
  const [item, setItem] = useState<Item | null>(null);

  useEffect(() => {
    if (!isReady) return;
    fetch(`/api/items/${query.itemId}`).then((r) => r.json()).then(setItem);
  }, [isReady, query.itemId]);

  if (!isReady) return null;
  if (!item) {
    notFound();
    void push('/404');
  }
  return <ItemView item={item} />;
}
```

**Почему.** `return null` до `isReady` — blank gate. `notFound()` из App API на Pages не работает как в `app/`. Второй `fetch` игнорирует GSSP и cookies запроса. `push('/404')` портит history вместо `notFound: true`.

**Как исправить.** `notFound: true` в data API страницы. Pending — `role="status"` в оболочке. Один источник — GSSP/GSP **или** клиентский query, не оба без принятой гидрации.

## 8. Scroll и focus после client navigation

Pages не подставляет App `loading.tsx`. Если соседи скроллят на смену `pathname` — повтори; смена только `query` на том же pathname — обычно нет. Title — `next/head` (или уже принятый helper), не сырой `asPath`. Focus — `h1` нового экрана.

```tsx
// надо
import Head from 'next/head';

export default function CatalogItemPage({ item }: { item: Item }) {
  const headingRef = useRef<HTMLHeadingElement>(null);
  const router = useRouter();

  useEffect(() => {
    headingRef.current?.focus();
  }, [router.pathname, item.id]);

  return (
    <>
      <Head>
        <title>{item.title}</title>
      </Head>
      <h1 ref={headingRef} tabIndex={-1}>
        {item.title}
      </h1>
    </>
  );
}
```

```tsx
// не надо — generateMetadata; scroll на каждый query; title = asPath
export async function generateMetadata() {
  return { title: 'Item' };
}

useEffect(() => {
  document.title = router.asPath;
  window.scrollTo(0, 0);
}, [router.asPath]);
```

**Почему.** `generateMetadata` — App Router, Pages его не вызовет. `asPath` в title светит query и служебные ключи. Scroll на каждую смену `q=` сбрасывает список.

**Как исправить.** `next/head` как у соседей. Scroll — зависимость `pathname`, не весь `asPath`, если меняется только фильтр. Focus на заголовок страницы.

## 9. Тестовый harness

Повтори принятый mock `next/router` / `next/link` зоны. Новый runner не добавляй. Не оборачивай Pages в `MemoryRouter` и не мокай `next/navigation`.

```tsx
// надо — контракт как у соседнего Pages-теста
const replace = jest.fn();
const push = jest.fn();

jest.mock('next/router', () => ({
  useRouter: () => ({
    pathname: '/catalog/[itemId]',
    query: { itemId: '42', q: 'oak' },
    asPath: '/catalog/42?q=oak',
    isReady: true,
    push,
    replace,
    prefetch: jest.fn(),
  }),
}));
```

Имена `jest` / `vi` и helper `createRouterMock` — только если они **уже** в репо.

```tsx
// не надо — App navigation mock; React Router harness
jest.mock('next/navigation', () => ({
  useRouter: () => ({ replace }),
}));
render(
  <MemoryRouter>
    <CatalogItemPage />
  </MemoryRouter>,
);
```

**Почему.** Тест зелёный на чужом API и красный в реальном Pages. `MemoryRouter` не эмулирует `query`/`isReady`. Логи `asPath` тащат URL в вывод.

**Как исправить.** Mock `next/router` как у соседей. Assert `replace` destination, `notFound` из GSSP, param `itemId`. Не добавляй второй test runner.
