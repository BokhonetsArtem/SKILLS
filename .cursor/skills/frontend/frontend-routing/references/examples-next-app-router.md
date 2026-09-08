# Примеры: Next.js App Router

Читай **только этот файл**, если трек зоны — Next App Router (`app/` + `page.tsx` / `layout.tsx`). `examples-next-pages-router.md` и `examples-react-router*.md` не открывай. `next/router` (Pages) сюда не подмешивай.

Сниппеты — обобщённые рецепты. Корень `app/` vs `src/app/` — как у соседей, второй каталог не заводи. `params` / `searchParams` как объект (Next 13–14) или `Promise` (Next 15+), `middleware.ts` vs `proxy.ts` — **только установленная major**, не latest из памяти. Не копируй `@/`, имена слайсов и product path. Runtime-логи navigation, params, return URL и PII не добавляй.

Parallel/intercepting routes, `middleware`/`proxy` — не default. Loaders нет: данные страницы — Server Component / принятый fetch-слой ([`frontend-api`](../../frontend-api/SKILL.md)).

Правила convention и Server/Client — в [next-routing-rules.md](next-routing-rules.md) (раздел App Router). History, URL-state, guards и a11y — в [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md).

Как читать пары: сверху **надо** (GOOD), ниже **не надо** (BAD), затем почему.

## 1. Декларация маршрута

Папки = сегменты URL. Файл `page.tsx` делает URL публичным. Группа `(shell)` в URL не входит. Не клади тот же экран в `pages/`, чтобы «было проще», и не ставь React Router рядом.

```tsx
// надо — файловый tree App Router (пути учебные)
// app/catalog/layout.tsx
// app/catalog/page.tsx              → /catalog
// app/catalog/new/page.tsx          → /catalog/new
// app/catalog/[itemId]/page.tsx     → /catalog/:itemId
// app/sign-in/page.tsx              → /sign-in

// app/catalog/page.tsx
import Link from 'next/link';

export default function CatalogPage() {
  return (
    <Link href="/catalog/new">New item</Link>
  );
}
```

Имена динамических сегментов — как у соседей (`[itemId]` vs `[id]`), не переименовывай дерево ради вкуса. Path helper зоны, если он есть, важнее литералов, разбросанных по client-кнопкам.

```tsx
// не надо — Pages-файл «заодно»; второй роутер; drift сегмента
// pages/catalog/[id].tsx          ← другой tree
// app/catalog/[itemId]/page.tsx
<Link href="/catalog/42">Item</Link>
<Link href="/catlog/42">Typo</Link>

'use client';
import { BrowserRouter } from 'react-router-dom';
```

**Почему.** Два файловых дерева и React Router внутри Next дают три канона URL. Опечатка `/catlog` не ловится компилятором сегментов. `[id]` и `[itemId]` на одном уровне — конфликт динамических имён.

**Как исправить.** Новый экран — только в существующем `app/` (или `src/app/`). Ссылки — тот же сегмент, что папка. Второй router-пакет не ставь.

## 2. Nested layouts

`layout.tsx` принимает `children` и остаётся смонтированным при смене `page`. Root layout содержит `html`/`body`. Не подменяй child ручным pathname и не путай layout с `template.tsx` без прецедента.

```tsx
// надо — app/catalog/layout.tsx
export default function CatalogLayout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <CatalogNav />
      {children}
    </>
  );
}
```

```tsx
// не надо — CSS-колонка вместо layout; ручной выбор page
'use client';
import { usePathname } from 'next/navigation';

export default function CatalogLayout({ children }: { children: React.ReactNode }) {
  const pathname = usePathname();
  return (
    <div className="grid-two-columns">
      {pathname.endsWith('/new') ? <CatalogNewPage /> : children}
    </div>
  );
}
```

**Почему.** Ручной `endsWith` обходит файловый tree: `[itemId]`, intercept и loading сегмента перестают совпадать. Сетка — [`frontend-layout`](../../frontend-layout/SKILL.md), не замена `layout.tsx`. Лишний `'use client'` на layout тащит вниз client boundary без нужды.

**Как исправить.** Вложенные маршруты — вложенные папки + `layout.tsx` + `children`. Visual grid оставь layout-скилу.

## 3. History: push vs replace

Внутренняя ссылка — `next/link` (push + prefetch по умолчанию версии). Login success и канонизация — `router.replace` на клиенте или `redirect()` на сервере. Не заменяй `Link` на `<a>` внутри приложения.

```tsx
// надо
import Link from 'next/link';
import { useRouter } from 'next/navigation';

<Link href={`/catalog/${item.id}`}>Open</Link>

function onSignInSuccess(returnPath: string) {
  router.replace(returnPath);
}
```

```tsx
// не надо — <a> без client navigation; push на login; window.location
<a href="/catalog">Catalog</a>
<button onClick={() => router.push('/sign-in')}>Sign in</button>
window.location.href = `/catalog/${id}`;
```

**Почему.** `<a href="/catalog">` даёт полную перезагрузку и теряет prefetch App Router. Push на sign-in оставляет закрытый URL в history. `window.location` сбрасывает RSC-переход.

**Как исправить.** `Link` для нового места. `replace` / серверный `redirect` для login и канонизации. Импорт роутера — `next/navigation`, не `next/router`.

## 4. Params и search params

На сервере — проп `params` / `searchParams` страницы (sync или `await` по major). На клиенте — `useParams` / `useSearchParams` из `next/navigation`. Фильтр, который должен быть deep link, пиши в query, не только в `useState`.

```tsx
// надо — server page. Next 13–14: params — объект. Next 15+: Promise, его await.
type ItemParams = { itemId: string };

export default async function CatalogItemPage({
  params,
}: {
  params: ItemParams; // 15+: Promise<ItemParams>; const { itemId } = await params;
}) {
  const { itemId } = params;
  return <CatalogItemBody itemId={itemId} />;
}

// клиентский фильтр списка (отдельный Client Component).
// На статическом сегменте оберни слот в <Suspense>, если родитель ещё не обернул.
'use client';
import { Suspense } from 'react';
import { usePathname, useRouter, useSearchParams } from 'next/navigation';

function CatalogFilters() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const q = searchParams.get('q') ?? '';

  function onQueryChange(next: string) {
    const nextParams = new URLSearchParams(searchParams.toString());
    if (next) nextParams.set('q', next);
    else nextParams.delete('q');
    const qs = nextParams.toString();
    const hash = window.location.hash;
    router.replace(`${pathname}${qs ? `?${qs}` : ''}${hash}`);
  }

  return <input value={q} onChange={(e) => onQueryChange(e.target.value)} />;
}

function CatalogFiltersSlot() {
  return (
    <Suspense fallback={<p role="status">Loading</p>}>
      <CatalogFilters />
    </Suspense>
  );
}

// Server page монтирует CatalogFiltersSlot (или свой <Suspense> вокруг CatalogFilters), не хук напрямую.
```

Не копируй `await params` в Next 13/14, если проп уже объект, и не читай sync `params.itemId` в 15+, если major требует await. Сверяй docs установленной версии.

```tsx
// не надо — useState вместо URL; затирание query; Pages-хук
const [q, setQ] = useState('');
router.push(`?q=${next}`);
import { useRouter } from 'next/router';
const { query } = useRouter();
```

**Почему.** Локальный `useState` не переживает refresh. `` push(`?q=`) `` сбрасывает другие ключи и путь. `next/router` — API Pages: `query` и `pathname` живут в другом контракте.

**Как исправить.** Проп страницы на сервере, `useSearchParams` на клиенте. Копируй текущие params, меняй один ключ. `replace`, если это тот же экран-фильтр. `useSearchParams` без предка-`<Suspense>` на статике даёт CSR-bailout / в части major ошибку сборки. Hash `next/navigation` не отдаёт: дописывай `window.location.hash`, если соседи hash сохраняют.

## 5. Redirect и return URL

Сервер: `redirect(path)` / `permanentRedirect(path)` из `next/navigation` в Server Component, server action или Route Handler. Path — внутренний. Query `from` / `next` — только same-origin relative.

```tsx
// надо — server
import { redirect } from 'next/navigation';

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

export default async function SignInPage({
  searchParams,
}: {
  searchParams: { from?: string }; // 15+: Promise<{ from?: string }> и await
}) {
  const returnPath = sameOriginPath(searchParams.from ?? null, '/catalog');
  return <SignInForm returnPath={returnPath} />;
}

async function completeSignIn(returnPath: string) {
  redirect(returnPath);
}
```

```tsx
// не надо — open redirect; server redirect из onClick
'use client';
import { redirect } from 'next/navigation';

function onSuccess() {
  redirect(searchParams.get('next') ?? '/');
}
window.location.assign(searchParams.get('next')!);
```

**Почему.** `redirect()` в client-модуле — чужой runtime: установленная версия может бросить или проигнорировать. Сырой `next=https://evil.example` — open redirect → [`frontend-security`](../../frontend-security/SKILL.md).

**Как исправить.** Server `redirect(внутренний path)`. На клиенте после жеста — `useRouter().replace(sameOriginPath(...))`. После decode path начинается с `/` и не с `//`, без `\`. Это минимум, не allowlist → [`frontend-security`](../../frontend-security/SKILL.md). Абсолютный URL из query не принимай.

## 6. Guards

Проверка сессии в Server Component / layout + `redirect` — UX до отрисовки. Client `replace` — пока гидрация не завершена. Ни то, ни другое не authorization. `middleware`/`proxy` — не замена API.

```tsx
// надо — server layout сегмента
import { redirect } from 'next/navigation';

export default async function AccountLayout({ children }: { children: React.ReactNode }) {
  const session = await readSession(); // как у зоны, не новый auth-стек
  if (!session) {
    redirect('/sign-in?from=/account');
  }
  return children;
}
```

`from` собери из известного внутреннего path сегмента, не из непроверенного заголовка Referer. Данные списка всё равно с endpoint с credentials.

```tsx
// не надо — client-only auth как защита; пустой UI; middleware = authorization
'use client';
export default function SecretPage({ secretList }: { secretList: Item[] }) {
  const session = useSession();
  if (!session) return null;
  return <pre>{JSON.stringify(secretList)}</pre>;
}

// middleware.ts / proxy.ts
export function middleware() {
  return NextResponse.next(); // «и так сойдёт — matcher закрыл /account»
}
```

**Почему.** RSC payload и client bundle уже могли получить `secretList`. `return null` только прячет UI. Matcher на Edge/Node — optimistic check, его обходят прямой URL к API. Аудит дырявого client/proxy — [`frontend-security`](../../frontend-security/SKILL.md).

**Как исправить.** `redirect` в server layout + реальные права на API. Client guard — запасной UX, не единственная дверь. Не создавай `middleware.ts` и `proxy.ts` сразу; не переименовывай файл «под следующую major».

## 7. Loading, error, 404

`notFound()` + `not-found.tsx`. Ошибка сегмента — `error.tsx` (client). Pending — `loading.tsx` **сегмента**, parent layout жив. Не дублируй server fetch клиентским `useEffect` на тот же GET без принятой гидрации.

```tsx
// надо — app/catalog/[itemId]/page.tsx
import { notFound } from 'next/navigation';

export default async function CatalogItemPage({
  params,
}: {
  params: { itemId: string }; // 15+: Promise<{ itemId: string }> и await
}) {
  const { itemId } = params;
  const item = await getItem(itemId); // HTTP — frontend-api
  if (!item) notFound();
  return <CatalogItemBody item={item} />;
}

// app/catalog/loading.tsx — не пустой экран всего дерева
export default function CatalogLoading() {
  return <p role="status">Loading</p>;
}
```

```tsx
// не надо — blank gate; client push на /404; второй fetch
'use client';
export default function CatalogItemPage({ itemId }: { itemId: string }) {
  const router = useRouter();
  const [item, setItem] = useState<Item | null>(null);

  useEffect(() => {
    fetch(`/api/items/${itemId}`)
      .then((r) => r.json())
      .then(setItem);
  }, [itemId]);

  if (!item) return null; // blank gate
  router.push('/404'); // вместо notFound()
  return <ItemView item={item} />;
}
```

**Почему.** Пустой `null` — blank gate: chrome и фокус пропадают. `router.push('/404')` маскирует `notFound()` и портит history. Второй client GET дублирует Server Component и расходится кэшем ([`frontend-api`](../../frontend-api/SKILL.md)).

**Как исправить.** `notFound()` / `error.tsx` / `loading.tsx` сегмента. Один источник данных — server page или уже принятая гидрация, не оба с нуля.

## 8. Scroll и focus после client navigation

`Link` по умолчанию скроллит к верху (сверь major). Смена только search на том же page — не обязана скроллить. Title — `metadata` / `generateMetadata`. После client transition перенеси focus на `h1` сегмента, если зона так делает.

```tsx
// надо — app/catalog/[itemId]/page.tsx (server)
export async function generateMetadata({
  params,
}: {
  params: { itemId: string }; // 15+: Promise<{ itemId: string }> и await
}) {
  const item = await getItem(params.itemId);
  return { title: item ? item.title : 'Item' };
}

// надо — Client Component заголовка, не тот же server-файл
'use client';
function CatalogHeading({ children }: { children: React.ReactNode }) {
  const pathname = usePathname();
  const headingRef = useRef<HTMLHeadingElement>(null);

  useEffect(() => {
    headingRef.current?.focus();
  }, [pathname]);

  return (
    <h1 ref={headingRef} tabIndex={-1}>
      {children}
    </h1>
  );
}
```

```tsx
// не надо — scroll на каждый q=; сырой URL в title; prefetch выключен у всех ссылок
<Link href="/catalog" prefetch={false}>
  Catalog
</Link>
useEffect(() => {
  document.title = window.location.href;
  window.scrollTo(0, 0);
}, [searchParams.toString()]);
```

**Почему.** Глобальный `prefetch={false}` убивает default App Router без причины зоны. Scroll на смену фильтра сбрасывает позицию списка. Title из полного href светит query и origin.

**Как исправить.** Prefetch — как у соседних `Link`. Scroll — default `Link` / `scroll={false}` точечно. Title — `generateMetadata`. Focus — заголовок сегмента, не `document.body`.

## 9. Тестовый harness

Повтори уже принятый в зоне способ (render Server Component, `next/navigation` mock **как у соседей**, Playwright route). Новый runner не добавляй. Проверяй сегмент, redirect и `notFound`, не сетку.

```tsx
// надо — тот же контракт, что у соседнего app-теста зоны
const replace = vi.fn();
vi.mock('next/navigation', () => ({
  useRouter: () => ({ push: vi.fn(), replace, prefetch: vi.fn(), back: vi.fn() }),
  usePathname: () => '/catalog',
  useSearchParams: () => new URLSearchParams('q=oak'),
}));
```

Имена `vi` / `jest` — как в репо. Не копируй этот mock, если зона рендерит страницу через свой test utility.

```tsx
// не надо — мок всего 'next'; React Router harness
jest.mock('next', () => ({}));
render(
  <MemoryRouter>
    <CatalogPage />
  </MemoryRouter>,
);
```

**Почему.** `MemoryRouter` проверяет чужой API. Мок пакета `next` целиком ломает `Link`/`Image`. Логи search/pathname — PII/URL в выводе.

**Как исправить.** Существующий test utility зоны. Assert `replace` с same-origin path, `notFound` вызван, сегмент `[itemId]` дошёл до page. Не добавляй Jest в Vitest-репо и наоборот.
