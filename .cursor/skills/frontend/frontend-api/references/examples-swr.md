# Примеры: SWR

Читай этот файл как server-state рецепт, если зона использует SWR (`useSWR`, `useSWRMutation`, `mutate`). TanStack/RTK Query examples не открывай. Дополнительно читай один transport example фактического client зоны (fetch/Axios/Ky), если для него есть файл.

Сниппеты — обобщённые рецепты. Не копируй `@/`, имена слайсов и пути `src/` как канон. Fetcher, `SWRConfig` и revalidate-флаги — как у **соседей этой зоны**. Runtime-логи request/response, token и PII не добавляй.

Правила транспорта, cache, mutation и auth — в соседних файлах этой папки. Fetcher ходит в **уже существующий** HTTP-client зоны.

**Версия.** SWR 2: fetcher для array-ключа получает массив целиком `([url, params]) => ...`. До 2.0 аргументы спредались — проверь установленную версию. `useSWRInfinite` и SSR — не default.

Как читать пары: сверху **надо**, ниже **не надо**, затем почему.

## 1. List/detail: ключ со всеми входами, условный fetch

Ключ однозначно кодирует ресурс и фильтры. Нет id — `null` (запрос не стартует). Fetcher проверяет HTTP status, если это `fetch`; для axios/ky следуй их ошибкам, не подменяй клиент.

```ts
// надо
type Thing = { id: string; title: string };
type ThingListParams = { q?: string; page: number };

export const thingKeys = {
  list: (params: ThingListParams) => ['things', 'list', params] as const,
  detail: (id: string) => ['things', 'detail', id] as const,
};

export function useThings(params: ThingListParams) {
  return useSWR(thingKeys.list(params), ([, , listParams]) => getThings(listParams));
}

export function useThing(id: string | undefined) {
  return useSWR(id ? thingKeys.detail(id) : null, ([, , thingId]) => getThing(thingId));
}
```

Не клади в ключ token: смена сессии — сброс cache (`mutate` / `cache.clear` по принятому событию зоны), не секрет в key. Если fetcher зависит от session helper, helper берёт контракт client, а не query string.

```ts
// не надо — неполный / нестабильный ключ; token в URL; useQuery рядом
useSWR('/things', () => getThings({ page, q }));
useSWR(['things', Date.now()], fetcher);
useSWR(`/things/${id}?token=${accessToken}`, fetcher);
useSWR(['/things', token], ([url, accessToken]) =>
  fetch(`${url}?access_token=${accessToken}`).then((r) => r.json()),
);
const { data } = useQuery({ queryKey: ['things'], queryFn: getThings });
```

**Почему.** Строка `'/things'` игнорирует page/q — чужие данные и промах invalidate. `Date.now()` бьёт дедуп. Токен в ключе/URL — утечка и лишний refetch. TanStack рядом — второй cache. В SWR 2 fetcher с array-ключом не `(url, token) =>`.

**Как исправить.** Array-ключ со всеми params. Условный `null`. Fetcher версии 2 принимает кортеж. HTTP — client зоны.

## 2. Mutation: useSWRMutation, revalidate после успеха

POST/PATCH/DELETE — `useSWRMutation` (`swr/mutation`), не `useSWR`. После успеха — `mutate` связанных ключей. Toast — в `onSuccess`, не до `trigger`.

```ts
// надо
import useSWRMutation from 'swr/mutation';
import { useSWRConfig } from 'swr';

type CreateThingBody = { title: string };

async function createThingFetcher(
  _key: string,
  { arg }: { arg: CreateThingBody },
) {
  return createThing(arg);
}

export function useCreateThing() {
  const { mutate } = useSWRConfig();

  return useSWRMutation('/things', createThingFetcher, {
    onSuccess: async () => {
      await mutate((key) => Array.isArray(key) && key[0] === 'things');
    },
  });
}

// вызывающий сценарий
const createThingMutation = useCreateThing();
const thing = await createThingMutation.trigger(body);
onThingCreated(thing.id); // toast / navigate после fulfilled
```

Фильтр `mutate(key => ...)` — SWR 2; в своей версии сверь API. Bound `mutate` с `useSWR` обновляет только этот ключ — для list после create обычно нужен global `mutate`.

```ts
// не надо — POST через useSWR; mutate до ответа; retry POST вручную
useSWR(submitted ? ['create', body] : null, () => createThing(body));

async function onSave() {
  toast.success('Saved');
  await mutate(thingKeys.list(params), [...items, optimistic], false);
  await trigger(body);
}

for (let i = 0; i < 3; i += 1) {
  try {
    await trigger(body);
    break;
  } catch {
    /* repeat POST */
  }
}
```

**Почему.** `useSWR` на create уйдёт при revalidate/focus. Оптимистичный `mutate` до сервера без rollback оставляет фейковые данные. Повтор POST плодит записи. Toast до `trigger` врёт.

**Как исправить.** Только `useSWRMutation`. Revalidate list/detail после успеха; сценарный toast/navigation — после `await trigger` в caller. POST не ретрай циклом.

## 3. Ошибки: fetcher бросает ApiError

SWR кладёт thrown error в `error`. Бросай узкий `ApiError`, не сырой Response. Не показывай abort/отмену как отказ, если зона отменяет вручную.

```ts
// надо
type ApiError = {
  status: number;
  message: string;
  fieldErrors?: Record<string, string>;
};

function isApiError(error: unknown): error is ApiError {
  return typeof error === 'object' && error !== null && 'status' in error && 'message' in error;
}

export async function getThing(id: string): Promise<Thing> {
  return httpGetThing(id);
}

export function useThing(id: string | undefined) {
  const { data, error, isLoading } = useSWR(
    id ? thingKeys.detail(id) : null,
    ([, , thingId]) => getThing(thingId),
    {
      shouldRetryOnError: (err) => !(isApiError(err) && err.status >= 400 && err.status < 500),
    },
  );

  return { data, error: error as ApiError | undefined, isLoading };
}
```

`shouldRetryOnError` / `errorRetryCount` сверяй с версией. 4xx не ретрай как сеть. Имя опции не выдумывай — читай docs установленной версии.

```ts
// не надо — json без status; сырой payload в UI
const fetcher = (url: string) => fetch(url).then((r) => r.json());
useSWR('/things/1', fetcher);

if (error) toast.error(JSON.stringify(error));
```

**Почему.** `fetch` + `json()` без `ok` прячет 500 в data или кидает нечитаемый SyntaxError. Сырой error в toast светит payload/PII.

**Как исправить.** Проверка status в HTTP helper. UI — `error.message`. Field errors — `frontend-forms`.

## 4. Pagination: page в ключе, не infinite по умолчанию

Специализированный рецепт. `page` (или cursor зоны) — часть ключа. `useSWRInfinite` — только при прецеденте бесконечного списка или явном запросе. Не дублируй список в глобальный store.

```ts
// надо
export function useThingsPage(params: ThingListParams) {
  return useSWR(thingKeys.list(params), ([, , listParams]) => getThings(listParams));
}
```

Флаги вроде `revalidateIfStale` зависят от версии — не копируй опции с документации другого мажора и не подставляй `keepPreviousData` из TanStack Query.

```ts
// не надо — infinite без запроса; page вне ключа; копия в store
useSWRInfinite((index) => ['things', index + 1], ([, page]) => getThings({ page }));

useSWR('things', () => getThings({ page, q }));

useThingStore.getState().setItems(data.items);
```

**Почему.** Infinite меняет модель страниц. Ключ без page смешивает ответы. Store расходится с SWR cache после `mutate`.

**Как исправить.** Params в ключе. Infinite — по прецеденту. Читай `data` из `useSWR`.
