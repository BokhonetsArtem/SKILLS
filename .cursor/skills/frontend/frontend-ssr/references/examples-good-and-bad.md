# Примеры: Плохо / Хорошо

Читай, когда нужен образец границы, не рецепт проекта. Синтаксис — нейтральный TSX; директиву client/island подставь ту, что уже есть в треке (`'use client'`, `client:load`, split + `onMount`). Импорты, алиасы, пути и библиотеки отсюда в репозиторий не копируй. Product path, имя пакета текущего репо и Vite/CSR «как у нас» каноном не считай.

Каждая пара объясняет **сломанную границу или first render**, не «поставь директиву». Runtime-логи в примеры не добавляй. Не создавай per-framework `examples-*`.

HOW `useRouter` / `Link` / `loading.tsx` — [`frontend-routing`](../../frontend-routing/SKILL.md). HOW `dehydrate` / query keys — [`frontend-api`](../../frontend-api/SKILL.md). Form `defaultValues` — [`frontend-forms`](../../frontend-forms/SKILL.md).

## 1. `'use client'` на всю страницу против узкого island

Плохо: статичная статья и список уезжают в клиентский бандл. Браузерный API нужен только фильтрам.

```tsx
"use client";

export function CatalogPage({ items }: { items: { id: string; title: string }[] }) {
  const [query, setQuery] = useState("");
  const visible = items.filter((item) => item.title.includes(query));

  return (
    <main>
      <h1>Catalog</h1>
      <input value={query} onChange={(event) => setQuery(event.target.value)} />
      <ul>
        {visible.map((item) => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>
    </main>
  );
}
```

Хорошо: parent остаётся на сервере, в client — только поле фильтра. Список можно отфильтровать на сервере или отдать plain `items` в узкий island, если фильтрация обязана быть мгновенной.

```tsx
export function CatalogPage({ items }: { items: { id: string; title: string }[] }) {
  return (
    <main>
      <h1>Catalog</h1>
      <CatalogFilters items={items} />
    </main>
  );
}
```

```tsx
"use client";

export function CatalogFilters({ items }: { items: { id: string; title: string }[] }) {
  const [query, setQuery] = useState("");
  const visible = items.filter((item) => item.title.includes(query));

  return (
    <>
      <input value={query} onChange={(event) => setQuery(event.target.value)} />
      <ul>
        {visible.map((item) => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>
    </>
  );
}
```

Уже такое разделение без дефекта — `KEEP`, не «улучшай» обратным склеиванием. Директиву и место файла не копируй: при переносе модуля — [`frontend-architecture`](../../frontend-architecture/SKILL.md).

## 2. `Date` / class / function против plain props

Плохо: через границу уходят несериализуемые значения. Трек не пронесёт `Date`, метод класса и callback из server parent.

```tsx
export function CatalogPage({ item }: { item: CatalogItem }) {
  return (
    <ItemMeta
      createdAt={item.createdAt}
      model={item}
      onReady={() => undefined}
    />
  );
}

class CatalogItem {
  createdAt = new Date();
  title = "Alpha";
}
```

Хорошо: на сервере снимок в plain-поля. Island получает строки и числа.

```tsx
export function CatalogPage({ item }: { item: { title: string; createdAt: string } }) {
  return <ItemMeta title={item.title} createdAt={item.createdAt} />;
}
```

```tsx
"use client";

export function ItemMeta({ title, createdAt }: { title: string; createdAt: string }) {
  return (
    <p>
      {title} · <time dateTime={createdAt}>{createdAt}</time>
    </p>
  );
}
```

Не протаскивай ORM-модель и не «чини» сериализацию `'use client'` на parent.

## 3. `Date.now()` / `window` на общем пути против стабильного first render

Плохо: server и client рисуют разный текст/ветку. Это hydration mismatch, не form hydration.

```tsx
export function CatalogPage() {
  const hour = new Date().getHours();
  const width = typeof window === "undefined" ? 0 : window.innerWidth;

  return (
    <p>
      {hour}:00 · {width}px
    </p>
  );
}
```

Хорошо: время — snapshot с сервера. Ширина живёт только в island; первый кадр island совпадает с серверным fallback (здесь — «неизвестно», не `0` vs `1440`).

```tsx
export function CatalogPage({ renderedAtHour }: { renderedAtHour: number }) {
  return (
    <p>
      {renderedAtHour}:00 · <ViewportWidthLabel />
    </p>
  );
}
```

```tsx
"use client";

export function ViewportWidthLabel() {
  const [width, setWidth] = useState<number | null>(null);

  useEffect(() => {
    const update = () => setWidth(window.innerWidth);
    update();
    window.addEventListener("resize", update);
    return () => window.removeEventListener("resize", update);
  }, []);

  return <span>{width === null ? "—" : `${width}px`}</span>;
}
```

Не добавляй `console.log` first HTML и не ставь `suppressHydrationWarning` на весь `<p>`, если зона так не делает на этом узле.

## 4. Повторный client fetch против одного источника

Плохо: parent уже загрузил список, island запрашивает тот же ресурс без принятого hydration-контракта.

```tsx
export async function CatalogPage() {
  const items = await fetchItems();
  return <CatalogList items={items} />;
}
```

```tsx
"use client";

export function CatalogList({ items }: { items: { id: string }[] }) {
  const [rows, setRows] = useState(items);

  useEffect(() => {
    fetchItems().then(setRows);
  }, []);

  return <ul>{rows.map((row) => <li key={row.id} />)}</ul>;
}
```

Хорошо: один источник — props с сервера. Клиентский refetch — только если в зоне уже принят dehydrate / `initialData` / framework cache; **как** его писать — [`frontend-api`](../../frontend-api/SKILL.md), не копируй сюда.

```tsx
export async function CatalogPage() {
  const items = await fetchItems();
  return <CatalogList items={items} />;
}
```

```tsx
"use client";

export function CatalogList({ items }: { items: { id: string }[] }) {
  return (
    <ul>
      {items.map((row) => (
        <li key={row.id} />
      ))}
    </ul>
  );
}
```

Проп `items` в «плохо» не оправдывает второй GET. Нет SSR-прецедента — не внедряй `fetchItems` на сервере «из примера».

## 5. `ssr:false` на страницу против browser-only виджета

Плохо: чтобы не чинить mismatch, отключают SSR у всего экрана. Статика и данные теряют трек зоны.

```tsx
const CatalogPage = dynamic(() => import("./catalog-page"), { ssr: false });

export function CatalogRoute() {
  return <CatalogPage />;
}
```

Хорошо: `ssr:false` (или `ClientOnly` / island-директива зоны) только на модуле, которому нужен WebGL/`window` сразу. Обложка страницы остаётся на сервере.

```tsx
const MapCanvas = dynamic(() => import("./map-canvas"), { ssr: false });

export function PlacesPage({ title, places }: { title: string; places: { id: string }[] }) {
  return (
    <main>
      <h1>{title}</h1>
      <ul>
        {places.map((place) => (
          <li key={place.id} />
        ))}
      </ul>
      <MapCanvas places={places} />
    </main>
  );
}
```

Осознанный browser-only chart/editor в узком island при корректном parent — `KEEP`. Не разворачивай его в полный SSR и не распространяй `ssr:false` на соседние экраны.

## 6. Island без слота против `<Suspense>` вокруг hook

Плохо: на треке, где клиентский hook требует boundary, island висит без слота. Страница уходит в CSR-bailout / mismatch.

```tsx
export function CatalogPage() {
  return (
    <main>
      <h1>Catalog</h1>
      <CatalogSearch />
    </main>
  );
}
```

Хорошо: слот — render-контракт. В `fix` оберни island. Имя hook и файл `loading.tsx` сюда не копируй: как писать router API — [`frontend-routing`](../../frontend-routing/SKILL.md).

```tsx
export function CatalogPage() {
  return (
    <main>
      <h1>Catalog</h1>
      <Suspense fallback={<p>…</p>}>
        <CatalogSearch />
      </Suspense>
    </main>
  );
}
```

Второй boundary не дублируй, если родитель уже обернул тот же слот. Цвет/отступы fallback — [`frontend-layout`](../../frontend-layout/SKILL.md).
