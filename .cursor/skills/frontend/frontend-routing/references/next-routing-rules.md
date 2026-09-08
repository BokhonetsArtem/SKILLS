# Next.js routing

Читай **только** если трек зоны — Next.js. App Router и Pages Router — разные треки: читай свой раздел, второй не применяй «на всякий случай». React Router examples не открывай.

Major version сверяй с lockfile. `params` / `searchParams` как Promise, `middleware.ts` vs `proxy.ts`, sync `cookies()` — **не** универсальны.

Сниппеты — иллюстрация convention, не канон `@/` чужого репо. GOOD/BAD пары — `examples-next-app-router.md` / `examples-next-pages-router.md`, если файлы есть.

Parallel/intercepting routes, `proxy`/`middleware` — не default: только прецедент или явный запрос. Runtime-логи navigation/params не добавляй.

---

## App Router

Читай, только если в зоне доминирует `app/` + `page.tsx` / `layout.tsx`.

### 1. Файловая convention

Папки = URL-сегменты. Файлы = UI и обработка сегмента. Корень `app/` или `src/app/` — как у соседей, второй каталог не заводи.

| Файл | Роль |
| --- | --- |
| `page.tsx` | публичный URL сегмента; без `page` папка не роут |
| `layout.tsx` | вложенный layout, `children` = page или child layout; root layout обязателен и содержит `html`/`body` |
| `template.tsx` | как layout, но **ремонитируется** при навигации — только если зона уже так сбрасывает state |
| `loading.tsx` | route-level pending сегмента, не blank всего дерева |
| `error.tsx` | error boundary сегмента; client component |
| `not-found.tsx` | UI для `notFound()` и неизвестного сегмента |
| `default.tsx` | fallback parallel slot |
| `route.ts` | HTTP handler сегмента — транспорт: [`frontend-api`](../../frontend-api/SKILL.md), не дублируй page data |

Не клади page «рядом» в `pages/`, чтобы «было проще». Не называй это FSD-слоем `app` только из-за имени папки — [`frontend-architecture`](../../frontend-architecture/references/principles-and-discovery.md).

```tsx
// иллюстрация, пути нейтральные
// app/catalog/layout.tsx  →  app/catalog/page.tsx
//                          →  app/catalog/[itemId]/page.tsx
```

Размещение содержимого page-модуля (слайс vs колокация) — architecture. Здесь — только что файл **существует** в tree Next.

### 2. Server / Client boundaries

По умолчанию файлы `app/` — Server Components. `'use client'` — только там, где нужны хуки браузера или обработчики.

| Нужно | Где |
| --- | --- |
| `useRouter`, `usePathname`, `useSearchParams` из `next/navigation` | Client Component |
| `<Link>` из `next/link` | можно в Server Component |
| `redirect`, `permanentRedirect`, `notFound` из `next/navigation` | Server Component, server action, Route Handler — **не** из произвольного client onClick |
| `cookies()`, `headers()` | сервер; async-это или нет — docs **установленной** major |
| Чтение `params` / `searchParams` в `page`/`layout` | проп страницы; sync object (Next 13–14) или `Promise` (Next 15+). Не копируй `await params` в 13/14 и не читай sync в 15+, если major уже требует await |

Не импортируй server-only модуль (секреты, `cookies`, ORM) в `'use client'` файл. Не вызывай `router.push` из Server Component. Не подставляй `next/router` (Pages) в App-экран.

`useSearchParams` в Client Component на статическом сегменте оборачивай в `<Suspense>` (или читай проп `searchParams` на server page). Иначе CSR-bailout; в части major — ошибка сборки `missing-suspense-with-csr-bailout`. Если соседний layout уже обернул слот — второй boundary не дублируй.

Клиентский `router.replace` после жеста ≠ серверный `redirect()` в RSC: первый пишет history на клиенте, второй отвечает до отрисовки. Для login/канонизации URL на сервере предпочти `redirect` в server-контексте, как у соседей.

Данные страницы: fetch в Server Component / принятый loader зоны. Параллельный client query на тот же GET — только если гидрация уже принята. Иначе [`frontend-api`](../../frontend-api/SKILL.md) по прецеденту, не второй клиент «на всякий случай».

### 3. Dynamic, catch-all, group, parallel

| Папка | URL | Когда |
| --- | --- | --- |
| `[itemId]` | один сегмент | карточка, сущность |
| `[...segments]` | обязательный catch-all | только если зона уже так режет хвост |
| `[[...segments]]` | optional catch-all | то же; не путай с обязательным |
| `(shell)` | **нет** в URL | группировка layout без сегмента |
| `@slot` | parallel slot | только прецедент или явный запрос; нужен `default.tsx` соседнего слота, если так требует установленная major |
| `(.)` / `(..)` intercept | перекрывает URL | не default; не добавляй «модальный intercept» без запроса |

Имена динамических сегментов — как у соседей (`[id]` vs `[itemId]`), не переименовывай дерево ради вкуса. Не смешивай `[id]` и `[itemId]` на одном уровне.

`generateStaticParams` / `dynamicParams` — только если зона уже SSG/ISR этот тип страниц.

Parallel routes не используй как CSS split. Visual: [`frontend-layout`](../../frontend-layout/SKILL.md).

### 4. `redirect` / `notFound`

- Нет сущности → `notFound()` в Server Component (или принятый `not-found.tsx` поток), не `return null` и не client `router.push('/404')`, если соседи на `notFound()`.
- Нужен другой канонический URL → `redirect(path)` / `permanentRedirect(path)` в server-контексте. Path — внутренний, не пользовательский абсолютный URL.
- Client guard (сессия ещё не гидратировалась): `useRouter` + `replace` на same-origin path. Это UX, не authorization — [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md).

Не вызывай `redirect()` из `'use client'` модуля, если установленная версия это запрещает / игнорирует. Не путай HTTP `redirect` в Route Handler с client navigation.

### 5. Навигация App Router

Первично: `next/link`. Программно: `useRouter()` из **`next/navigation`** (`push` / `replace` / `prefetch` / `back`). Pathname: `usePathname`. Search: `useSearchParams` (клиент) или проп `searchParams` (server page).

- `router.push` для нового места, `replace` для login/канонизации.
- Не собирай `<a href="/catalog">` вместо `Link` для внутренних маршрутов: потеряешь prefetch и client transition.
- Prefetch — поведение установленной версии `Link`; не копируй `prefetch={false}` на все ссылки без причины зоны.
- Если в `next.config` задан `basePath` — повтори его там, не в каждом `href`. `Link href="/catalog"` остаётся path сегмента; Next сам префиксует. Не пиши `/app/catalog` при `basePath: '/app'`.
- Клиентский `replace` строкой `pathname + search`: если соседи сохраняют hash, допиши `window.location.hash` (`next/navigation` hash не отдаёт). Не претендуй на сохранение hash, если сниппет его отбрасывает.
- После client navigation: scroll/focus/title — [navigation-state-and-boundaries.md](navigation-state-and-boundaries.md) §7. `generateMetadata` / `metadata` задают title на сервере; клиентский переход должен сменить title вместе с сегментом.

Loading: `loading.tsx` сегмента + сохранение parent layout. Не вешай на root единственный blank spinner.

### 6. `middleware.ts` / `proxy.ts` — по major

| Major (ориентир) | Файл и export | Runtime |
| --- | --- | --- |
| Next 13–15 | `middleware.ts` / `middleware.js`, `export function middleware` | Edge (типичный default этой линии) — сверь docs установленной версии |
| Next 16+ | `proxy.ts` / `proxy.js`, `export function proxy`; `middleware` **deprecated** | Node.js-only у `proxy` в линии 16; не выставляй Edge runtime, если версия это запрещает |

Не создавай оба файла. Не переименовывай `middleware` → `proxy` в репо на 15-й major «заранее». Не оставляй старый `middleware.ts` рядом с `proxy.ts` на 16+: старый файл могут игнорировать.

Matcher и rewrite/redirect на границе сети — last resort, как у соседей. Это не Express-middleware и **не** authorization (optimistic check максимум). Реальные права — layout/server/data + API. Аудит «дырявого» client/proxy check — [`frontend-security`](../../frontend-security/SKILL.md).

Не клади тяжёлую сессию, ORM и PII-логи в `middleware`/`proxy`.

---

## Pages Router (legacy)

Читай, только если зона — `pages/` (`_app`, `_document`, file-based pages) **без** App Router в этой правке. Не переноси экран в `app/` без запроса.

### 7. Файлы и data API

| Файл | URL |
| --- | --- |
| `pages/index.tsx` | `/` |
| `pages/catalog/index.tsx` | `/catalog` |
| `pages/catalog/[itemId].tsx` | `/catalog/:itemId` |
| `pages/catalog/[...segments].tsx` | catch-all |
| `pages/404.tsx` / `pages/_error.tsx` | 404 / error — как у соседей |
| `pages/api/**` | Route Handlers Pages — [`frontend-api`](../../frontend-api/SKILL.md) |

`getServerSideProps` / `getStaticProps` / `getInitialProps` — только если этот тип страницы ими уже пользуется. Не добавляй GSSP «для SEO» на статическую страницу без запроса. Не подставляй App `generateMetadata` / `loading.tsx` в `pages/`.

Импорты навигации: `next/router` (`useRouter` с `pathname`, `query`, `asPath`, `push`, `replace`). `next/link` — да. **`next/navigation` в Pages-зоне не используй**, пока соседи этого экрана на нём не живут (смешанный репо — трек **этого** файла).

`router.query` может быть пустым на первом client render — не читай обязательный id без учёта гидрации, как делают соседние Pages. `router.push({ pathname, query, hash })` сохраняй query- и hash-контракт зоны (`hash` из `window.location.hash` / `asPath`, если соседи hash держат).

Custom `_app` держит глобальный chrome. Не оборачивай page во второй mini-router.

`middleware` на старых Pages: проверь, что установленная Next major вообще применяет его к Pages в этом деплое. Не считай rename в `proxy` доказательством, что Pages-поведение внезапно «починилось».

Guard: client `useRouter().replace` + same-origin return URL. `getServerSideProps` redirect — серверный UX-redirect, всё ещё не замена API authorization.

### 8. Чего не делать на Next-треках

- Не смешивать `next/navigation` и `next/router` в одном экране.
- Не создавать параллельный React Router внутри Next.
- Не вызывать server `redirect` / `notFound` из client-only модуля (App) и не ждать App `loading.tsx` в Pages.
- Не объявлять `await params` или `proxy.ts` универсальными — только major зоны.
- Не выдавать `middleware`/`proxy`/скрытый UI за authorization.
- Не принимать return URL из query как абсолютный внешний адрес.
- Не дублировать page fetch клиентским query без принятой гидрации.
