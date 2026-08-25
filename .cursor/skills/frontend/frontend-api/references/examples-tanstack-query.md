# Примеры: TanStack Query

Читай этот файл как server-state рецепт, если зона использует TanStack Query (`useQuery`, `useMutation`, `QueryClient`). SWR/RTK Query examples не открывай. Дополнительно читай один transport example фактического client зоны (fetch/Axios/Ky), если для него есть файл.

Сниппеты — обобщённые рецепты. Не копируй `@/`, имена слайсов и пути `src/` как канон. `QueryClient`, fetcher и `staleTime` — как у **соседей этой зоны**. Runtime-логи request/response, token и PII не добавляй.

Правила транспорта, cache, mutation и auth — в соседних файлах этой папки. HTTP-вызовы внутри `queryFn` / `mutationFn` идут через **уже существующий** client зоны (fetch/axios/ky), не второй пакет.

**Версия.** Объектный синтаксис `useQuery({ queryKey, queryFn })` — v4.36+/v5. `gcTime` в v5 = `cacheTime` в v4. Проверь установленную версию. Infinite query и SSR/`dehydratedState` — не default.

Как читать пары: сверху **надо**, ниже **не надо**, затем почему.

## 1. List/detail: стабильный queryKey, params, abort

Ключ — сериализуемое дерево **всех** входов: коллекция, id, фильтры, page, session scope. `queryFn` принимает `{ signal }` и передаёт его в client. Нет id — `enabled: false`.

```ts
// надо
type Thing = { id: string; title: string };
type ThingListParams = { q?: string; page: number };

export const thingKeys = {
  all: ['things'] as const,
  list: (params: ThingListParams) => [...thingKeys.all, 'list', params] as const,
  detail: (id: string) => [...thingKeys.all, 'detail', id] as const,
};

export function useThingsQuery(params: ThingListParams) {
  return useQuery({
    queryKey: thingKeys.list(params),
    queryFn: ({ signal }) => getThings(params, signal),
  });
}

export function useThingQuery(id: string | undefined) {
  return useQuery({
    queryKey: thingKeys.detail(id ?? ''),
    queryFn: ({ signal }) => getThing(id!, signal),
    enabled: Boolean(id),
  });
}
```

`getThings` / `getThing` — helper существующего HTTP-клиента с проверкой status (см. example своего transport, не копируй чужой client).

```ts
// не надо — волатильное значение в ключе; нет фильтра; v3-массив; fetch без signal; второй query-слой
useQuery(['things', { page, now: Date.now() }], () => getThings({ page }));
useQuery({ queryKey: ['things'], queryFn: () => getThings({ page, q }) });
useQuery('things', getThings);
useQuery({
  queryKey: thingKeys.detail(id ?? ''),
  queryFn: () => fetch(`/things/${id}`).then((r) => r.json()),
});
const { data } = useSWR('/things', fetcher);
```

**Почему.** `Date.now()` даёт новый ключ на каждом рендере и вечный refetch. Новый plain object с теми же сериализуемыми полями допустим: TanStack Query детерминированно хеширует query keys. Ключ без `q`/`page` отдаёт чужой список. Строковый ключ v3 в v5 не тот API. `fetch` без `ok`/`signal` обходит client зоны. SWR рядом — второй cache.

**Как исправить.** Фабрика ключей. Все params в `queryKey`. `enabled` без id. `signal` в client зоны.

## 2. Mutation: invalidate после успеха, не retry POST

Изменения — `useMutation`. После успешного ответа — `invalidateQueries` по префиксу list/detail. Default retry мутаций в v5 — `0`; не ставь `retry: 3` на неидемпотентное создание.

```ts
// надо
type CreateThingBody = { title: string };

export function useCreateThingMutation() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (body: CreateThingBody) => createThing(body),
    onSuccess: () => {
      void queryClient.invalidateQueries({ queryKey: thingKeys.all });
    },
  });
}

export function useUpdateThingMutation() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, body }: { id: string; body: CreateThingBody }) =>
      updateThing(id, body),
    onSuccess: (thing) => {
      void queryClient.invalidateQueries({ queryKey: thingKeys.detail(thing.id) });
      void queryClient.invalidateQueries({ queryKey: thingKeys.all });
    },
  });
}

// вызывающий сценарий
const createThingMutation = useCreateThingMutation();
const thing = await createThingMutation.mutateAsync(body);
onThingCreated(thing.id); // toast / navigate после fulfilled
```

Общая cache invalidation может жить в mutation hook. Toast/navigate — в вызывающем сценарии после `mutateAsync`, не в `onMutate` и не до вызова.

```ts
// не надо — POST через useQuery; retry POST; toast в onMutate; нет invalidate
useQuery({
  queryKey: ['create-thing', body],
  queryFn: () => createThing(body),
  enabled: submitted,
});

useMutation({
  mutationFn: createThing,
  retry: 3,
  onMutate: () => {
    toast.success('Saved');
  },
});
```

**Почему.** Query с POST кэшируется, дедупится и может уйти повторно. Retry POST плодит ресурсы. Toast в `onMutate` срабатывает до ответа и остаётся при ошибке. Без invalidate список stale.

**Как исправить.** Только `useMutation`. `retry` не включай на неидемпотентный POST. Эффект — `onSuccess`. Инвалидируй `thingKeys`.

## 3. Ошибки: не сырой Error в UI

`queryFn` бросает уже нормализованный `ApiError`. В компоненте — `error.message`, не `JSON.stringify(error)` и не поля PII. Cancel/`AbortError` TanStack глотает сам, если `signal` использован; не показывай его как ошибку формы.

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

async function getThing(id: string, signal?: AbortSignal): Promise<Thing> {
  return httpGetThing(id, signal);
}

export function useThingQuery(id: string | undefined) {
  return useQuery({
    queryKey: thingKeys.detail(id ?? ''),
    queryFn: ({ signal }) => getThing(id!, signal),
    enabled: Boolean(id),
    retry: (failureCount, error) => {
      if (isApiError(error) && error.status >= 400 && error.status < 500) {
        return false;
      }
      return failureCount < 3;
    },
  });
}
```

`httpGetThing` бросает `ApiError` после проверки HTTP status (транспорт зоны). 4xx не ретрай как сеть.

```ts
// не надо
useQuery({
  queryKey: thingKeys.detail(id ?? ''),
  queryFn: async () => {
    const response = await fetch(`/things/${id}`);
    return response.json();
  },
  throwOnError: true,
});

if (isError) toast.error(JSON.stringify(error));
```

**Почему.** `json()` без `ok` прячет 404 в «успешных» данных. Сырой объект ошибки в toast светит payload. `throwOnError` — политика зоны, не способ нормализации.

**Как исправить.** Нормализация в HTTP helper. UI — безопасный `message`. Field errors — `frontend-forms`.

## 4. Pagination: page в ключе, не infinite по умолчанию

Специализированный рецепт. Контракт page/size (или cursor зоны) входит в `queryKey`. `useInfiniteQuery` — только при локальном прецеденте бесконечного списка или явном запросе. Не копируй страницы в Zustand/Redux, если cache Query уже есть.

```ts
// надо
export function useThingsPageQuery(params: ThingListParams) {
  return useQuery({
    queryKey: thingKeys.list(params),
    queryFn: ({ signal }) => getThings(params, signal),
    placeholderData: keepPreviousData, // имя helper — проверить версию (keepPreviousData / placeholderData)
  });
}
```

`keepPreviousData` в v5 импортируется из `@tanstack/react-query`. Если в зоне его нет — не добавляй «для UX» без запроса.

```ts
// не надо — infinite без прецедента; дубль server-state; page не в ключе
useInfiniteQuery({
  queryKey: ['things'],
  queryFn: ({ pageParam }) => getThings({ page: pageParam }),
  initialPageParam: 1,
  getNextPageParam: (last) => last.page + 1,
});

useQuery({
  queryKey: ['things'],
  queryFn: () => getThings({ page, q }),
});

const setThings = useThingStore((s) => s.setThings);
setThings(data);
```

**Почему.** Infinite меняет UX и ключи. Ключ без page смешивает страницы. Store дублирует cache и расходится после invalidate.

**Как исправить.** Page в `queryKey`. Infinite — только по прецеденту. Список читай из `useQuery`, не копируй в store.
