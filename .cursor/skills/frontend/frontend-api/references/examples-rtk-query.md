# Примеры: RTK Query

Читай этот файл как server-state/transport-adapter рецепт, если зона использует RTK Query (`createApi`, `fetchBaseQuery`, generated hooks). TanStack Query/SWR examples не открывай. Transport example нужен дополнительно только при существующем custom baseQuery на fetch/Axios/Ky.

Сниппеты — обобщённые рецепты. Не копируй `@/`, имена слайсов, tag enums и пути `src/` как канон. `baseApi`, `tagTypes` и auth helper — как у **соседей этой зоны**. Runtime-логи request/response, token и PII не добавляй.

Правила транспорта, cache, mutation и auth — в соседних файлах этой папки.

Optimistic update, infinite и codegen — не default. Token storage не выбирай и не переноси: подключай уже существующий `prepareHeaders` / credentials.

Как читать пары: сверху **надо**, ниже **не надо**, затем почему.

## 1. List/detail: injectEndpoints, params, tags, skip

Один `baseApi`. Новые endpoint — `injectEndpoints`, не второй `createApi`. Аргумент query кодирует все входы (фильтр, page, id). Нет id — `skip` / `skipToken`. `providesTags` стыкует list и detail.

```ts
// надо
type Thing = { id: string; title: string };
type ThingListParams = { q?: string; page: number };
type ThingListResponse = { items: Thing[] };

export const thingsApi = baseApi.injectEndpoints({
  endpoints: (build) => ({
    getThings: build.query<ThingListResponse, ThingListParams>({
      query: (params) => ({ url: '/things', params }),
      providesTags: (result) =>
        result
          ? [
              ...result.items.map(({ id }) => ({ type: 'Thing' as const, id })),
              { type: 'Thing', id: 'LIST' },
            ]
          : [{ type: 'Thing', id: 'LIST' }],
    }),
    getThing: build.query<Thing, string>({
      query: (id) => ({ url: `/things/${encodeURIComponent(id)}` }),
      providesTags: (_result, _error, id) => [{ type: 'Thing', id }],
    }),
  }),
});

export const { useGetThingsQuery, useGetThingQuery } = thingsApi;

function ThingDetails({ id }: { id: string | undefined }) {
  const { data } = useGetThingQuery(id ?? skipToken);
  return data ? <p>{data.title}</p> : null;
}
```

Identity-`transformResponse` не добавляй, если wire совпадает с `Thing`.

```ts
// не надо — второй createApi; голый fetch; нет skip; identity-mapper; нестабильный arg
const thingsApi = createApi({
  reducerPath: 'thingsApi',
  baseQuery: fetchBaseQuery({ baseUrl: 'https://example.com' }),
  endpoints: (build) => ({
    getThing: build.query<Thing, { id: string; now: number }>({
      query: ({ id }) => `/things/${id}`,
      transformResponse: (dto: Thing) => dto,
    }),
  }),
});

useGetThingQuery({ id, now: Date.now() });
useGetThingQuery(id as string); // id ещё undefined
await fetch('/things');
```

**Почему.** Второй `createApi` — отдельный store slice, tags и refresh. `Date.now()` в аргументе убивает дедуп. Запрос без id бьёт `/things/undefined`. Identity-mapper — шум. `fetch` обходит baseQuery.

**Как исправить.** `injectEndpoints` в существующий `baseApi`. Стабильный аргумент. `skipToken`, пока нет id.

## 2. Mutation: invalidation после успеха

Create/update — `build.mutation`. Tags инвалидируй на мутации. Toast/navigate — **после** `queryFulfilled`, не в `onClick` и не до await.

```ts
// надо
type CreateThingBody = { title: string };

export const createThingApi = baseApi.injectEndpoints({
  endpoints: (build) => ({
    createThing: build.mutation<Thing, CreateThingBody>({
      query: (body) => ({ url: '/things', method: 'POST', body }),
      invalidatesTags: (result) =>
        result ? [{ type: 'Thing', id: 'LIST' }] : [],
    }),
    updateThing: build.mutation<Thing, { id: string; body: CreateThingBody }>({
      query: ({ id, body }) => ({
        url: `/things/${encodeURIComponent(id)}`,
        method: 'PATCH',
        body,
      }),
      invalidatesTags: (result, _error, { id }) =>
        result
          ? [
              { type: 'Thing', id },
              { type: 'Thing', id: 'LIST' },
            ]
          : [],
    }),
  }),
});

export const { useCreateThingMutation, useUpdateThingMutation } = createThingApi;

// вызывающий сценарий
const [createThing] = useCreateThingMutation();
const thing = await createThing(body).unwrap();
onThingCreated(thing.id); // toast / navigate после fulfilled
```

`onThingCreated` — сценарий (toast/navigate), не API endpoint и не shared-кнопка. Ошибку не логируй телом ответа.

```ts
// не надо — toast до queryFulfilled; нет invalidation; retry утилита на POST
async onQueryStarted(_arg, { queryFulfilled }) {
  toast.success('Saved');
  await queryFulfilled;
}

createThing: build.mutation<Thing, CreateThingBody>({
  query: (body) => ({ url: '/things', method: 'POST', body }),
}),

const api = createApi({
  baseQuery: retry(fetchBaseQuery({ baseUrl: apiBaseUrl }), { maxRetries: 5 }),
  endpoints: (build) => ({
    createThing: build.mutation<Thing, CreateThingBody>({
      query: (body) => ({ url: '/things', method: 'POST', body }),
    }),
  }),
});
```

**Почему.** Toast до fulfilled срабатывает и при последующем reject, либо врёт, если не дождались. Без `invalidatesTags` список остаётся stale. `retry()` вокруг всего `baseQuery` повторит неидемпотентный POST.

**Как исправить.** Сначала `await queryFulfilled`. Tags на мутации. `retry` не вешай на POST; `maxRetries: 0` на mutation, если retry уже стоит глобально.

## 3. Ошибки: узкий контракт, не сырой `error.data`

Нормализуй в `transformErrorResponse` или helper зоны. UI получает `status` + безопасный `message` + `fieldErrors`. Не toast-ай весь `error.data`, email, token.

```ts
// надо
type ApiError = {
  status: number;
  message: string;
  fieldErrors?: Record<string, string>;
};

function toApiError(error: FetchBaseQueryError): ApiError {
  if (error.status === 'FETCH_ERROR' || error.status === 'TIMEOUT_ERROR') {
    return { status: 0, message: 'Request failed' };
  }

  if (typeof error.status !== 'number') {
    return { status: 0, message: 'Request failed' };
  }

  return {
    status: error.status,
    message: readSafeMessage(error.data),
    fieldErrors: readFieldErrors(error.data),
  };
}

getThing: build.query<Thing, string>({
  query: (id) => `/things/${encodeURIComponent(id)}`,
  transformErrorResponse: (error) => toApiError(error),
}),
```

Field errors только нормализуй; `setError` формы — `frontend-forms`.

```ts
// не надо — сырой payload; PII; console тела ошибки
catch (error) {
  toast.error(JSON.stringify(error));
  console.error(error);
}

transformErrorResponse: (error) => error.data,
```

**Почему.** `error.data` часто содержит внутренний текст, поля пользователя и лишние детали. Лог тела — PII/token risk.

**Как исправить.** `toApiError`. В UI — `message`. В форму — `fieldErrors`.

## 4. Query читает, mutation меняет

Logout, clone, send-email и другие side effects — **mutation** в RTK Query, даже если legacy endpoint использует GET. Query кэширует и может перезапуститься (refetch on focus / mount). HTTP-метод сохраняй по фактическому контракту; не меняй GET на POST только на клиенте.

```ts
// надо
logout: build.mutation<void, void>({
  query: () => ({ url: '/logout', method: 'POST' }),
  async onQueryStarted(_arg, { queryFulfilled, dispatch }) {
    await queryFulfilled;
    dispatch(baseApi.util.resetApiState());
  },
}),

createThing: build.mutation<Thing, CreateThingBody>({
  query: (body) => ({ url: '/things', method: 'POST', body }),
  invalidatesTags: [{ type: 'Thing', id: 'LIST' }],
}),
```

Не копируй server-state из query-кэша в отдельный slice «чтобы было под рукой», если хук уже отдаёт `data`.

```ts
// не надо — side-effecting query; дубль в slice; initiate() как скрытый POST
logout: build.query<void, void>({
  query: () => '/logout',
}),

createThing: build.query<Thing, CreateThingBody>({
  query: (body) => ({ url: '/things', method: 'POST', body }),
}),

onQueryStarted(_arg, { queryFulfilled, dispatch }) {
  const { data } = await queryFulfilled;
  dispatch(thingsSlice.actions.setList(data.items));
}
```

**Почему.** `useLogoutQuery()` уйдёт при mount/refetch и разлогинит повторно. Create-query дедупится и кэшируется. Параллельный slice расходится с RTK cache.

**Как исправить.** Побочный эффект — `build.mutation`. Список читай из query hook / tags, не дублируй.

## 5. 401: один refresh на всех, без цикла

Если зона уже обновляет сессию при 401 — **single-flight**: общий promise, очередь исходных запросов, refresh-endpoint не триггерит сам себя. Не канонизируй storage токена: запись/чтение — существующий helper зоны. Официальный образец RTK — mutex/`waitForUnlock` ([Automatic re-authorization](https://redux-toolkit.js.org/rtk-query/usage/customizing-queries#automatic-re-authorization-by-extending-fetchbasequery)); тот же смысл можно собрать на одном `Promise` без нового пакета.

```ts
// надо — схема, не предписание storage
let refreshPromise: Promise<boolean> | null = null;

const rawBaseQuery = fetchBaseQuery({
  baseUrl: apiBaseUrl,
  credentials: 'include',
  prepareHeaders: (headers, api) => {
    applyExistingAuth(headers, api.getState);
    return headers;
  },
});

const baseQueryWithReauth: BaseQueryFn = async (args, api, extraOptions) => {
  let result = await rawBaseQuery(args, api, extraOptions);
  const url = typeof args === 'string' ? args : args.url;

  if (result.error?.status === 401 && url !== '/refresh') {
    if (!refreshPromise) {
      refreshPromise = rawBaseQuery(
        { url: '/refresh', method: 'POST' },
        api,
        extraOptions,
      )
        .then((refreshResult) => {
          if (!refreshResult.error) {
            // Cookie refresh может успешно вернуть 204 без data.
            if (refreshResult.data !== undefined) {
              applyExistingSession(refreshResult.data);
            }
            return true;
          }
          api.dispatch(sessionBroken());
          return false;
        })
        .catch(() => {
          // Custom baseQuery не должен выбрасывать наружу.
          api.dispatch(sessionBroken());
          return false;
        })
        .finally(() => {
          refreshPromise = null;
        });
    }

    const refreshed = await refreshPromise;
    if (refreshed) {
      result = await rawBaseQuery(args, api, extraOptions);
    }
  }

  return result;
};
```

`baseQueryWithReauth` всегда возвращает result и не выбрасывает rejection наружу: RTK Query ожидает `{ data }` или `{ error }`. Успех refresh проверяй по отсутствию `error`, а не по truthy `data`, потому что cookie endpoint может вернуть 204.

```ts
// не надо — refresh на каждый 401; refresh как query; цикл на сам /refresh
const baseQueryWithReauth: BaseQueryFn = async (args, api, extraOptions) => {
  let result = await rawBaseQuery(args, api, extraOptions);
  if (result.error?.status === 401) {
    await rawBaseQuery({ url: '/refresh', method: 'GET' }, api, extraOptions);
    result = await rawBaseQuery(args, api, extraOptions);
  }
  return result;
};

refresh: build.query<Session, void>({
  query: () => '/refresh',
}),
```

**Почему.** Параллельные 401 без очереди бьют refresh много раз и гоняют гонки записи сессии. Refresh, оформленный как `build.query`, кэшируется и перезапускается. Повтор refresh на 401 самого `/refresh` — бесконечный цикл.

**Как исправить.** Один in-flight refresh. Исключи refresh URL из reauth. Сессию подключай существующим контрактом, storage не выбирай здесь. Нет refresh в зоне — не выдумывай.
