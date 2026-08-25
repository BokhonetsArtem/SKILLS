# Примеры: Axios

Читай этот файл как transport-рецепт, если зона использует Axios. Другие transport examples (fetch/Ky) не открывай. Если поверх Axios есть TanStack Query/SWR или RTK Query с custom baseQuery, дополнительно читай только соответствующий server-state example.

Сниппеты — обобщённые рецепты. Не копируй `@/`, имена слайсов и пути `src/` как канон. Instance, interceptors и `withCredentials` — как у **соседей этой зоны**. Runtime-логи request/response, token и PII не добавляй.

Правила транспорта, cache, mutation и auth — в соседних файлах этой папки.

AbortSignal и upload progress — не повод ставить второй клиент. `CancelToken` — legacy; в актуальном Axios используй `signal` (проверить установленную версию).

Как читать пары: сверху **надо**, ниже **не надо**, затем почему.

## 1. List/detail: один instance, params, signal

Базовый client зоны — единственная точка `baseURL`, credentials и default headers. Params — опция `params`, не строковая склейка. Axios по умолчанию **отклоняет** промис на 4xx/5xx — это не отмена проверки контракта и DTO.

```ts
// надо
import { http } from './http'; // существующий client этой origin/auth границы

type Thing = { id: string; title: string };
type ThingListParams = { q?: string; page: number };

export async function getThings(
  params: ThingListParams,
  signal?: AbortSignal,
): Promise<Thing[]> {
  const { data } = await http.get<Thing[]>('/things', { params, signal });
  return data;
}

export async function getThing(id: string, signal?: AbortSignal): Promise<Thing> {
  if (!id) {
    throw new Error('Missing thing id');
  }

  const { data } = await http.get<Thing>(
    `/things/${encodeURIComponent(id)}`,
    { signal },
  );
  return data;
}
```

Не клади token в `params`. Header/cookie — в instance (`interceptors.request` / существующий helper), не в каждом вызове.

```ts
// не надо — второй client; чужой транспорт; секрет в query; CancelToken «заодно»
const local = axios.create({ baseURL: 'https://example.com' });
const { data } = await fetch(`/things?token=${accessToken}`).then((r) => r.json());
const source = axios.CancelToken.source();
await axios.get('/things', { params: { page, accessToken }, cancelToken: source.token });
```

**Почему.** Второй instance разъезжается с credentials и refresh. `fetch` в axios-зоне — второй транспорт. Токен в query — история и Referer. `CancelToken` устарел относительно `AbortSignal` в текущем Axios (проверить версию).

**Как исправить.** Импортируй уже существующий `http`. `params` без секретов. Отмена — `signal`.

## 2. Create/update: без авто-retry POST, эффект после успеха

Мутация идёт через тот же instance. Retry в interceptor — только для идемпотентных/безопасных методов или при `Idempotency-Key` зоны.

```ts
// надо
type CreateThingBody = { title: string };

export async function createThing(
  body: CreateThingBody,
  signal?: AbortSignal,
): Promise<Thing> {
  const { data } = await http.post<Thing>('/things', body, { signal });
  return data;
}

export async function updateThing(
  id: string,
  body: CreateThingBody,
  signal?: AbortSignal,
): Promise<Thing> {
  const { data } = await http.patch<Thing>(
    `/things/${encodeURIComponent(id)}`,
    body,
    { signal },
  );
  return data;
}

// сценарий — после await
await createThing(body);
onCreated(); // toast / navigate только здесь
```

```ts
// не надо — retry любого POST; toast до await; validateStatus: () => true без разбора
http.interceptors.response.use(undefined, async (error) => {
  if (error.config) {
    return http.request(error.config);
  }
  return Promise.reject(error);
});

toast.success('Saved');
await http.post('/things', body);

const { data } = await http.post<Thing>('/things', body, {
  validateStatus: () => true,
});
```

**Почему.** Слепой повтор POST создаёт дубликаты. Toast до ответа врёт при 409/422. `validateStatus: () => true` превращает 500 в «успешный» `data`.

**Как исправить.** POST не ретрай в interceptor. Сначала `await`, потом side effect. Статус-ошибки оставь default Axios (или политике зоны), затем нормализуй.

## 3. Ошибки: нормализация в одном месте

Перехвати `AxiosError` в helper или response interceptor зоны. `CanceledError` / `axios.isCancel` — не UI-ошибка. В `message` — безопасная формулировка, не `error.response.data` целиком.

```ts
// надо
import axios, { AxiosError } from 'axios';

type ApiError = {
  status: number;
  message: string;
  fieldErrors?: Record<string, string>;
};

export function toApiError(error: unknown): ApiError {
  if (axios.isCancel(error)) {
    throw error;
  }

  if (axios.isAxiosError(error)) {
    const status = error.response?.status ?? 0;
    if (!error.response) {
      return { status, message: 'Request failed' };
    }
    return {
      status,
      message: readSafeMessage(error.response.data),
      fieldErrors: readFieldErrors(error.response.data),
    };
  }

  return { status: 0, message: 'Request failed' };
}

http.interceptors.response.use(
  (response) => response,
  (error: AxiosError) => Promise.reject(toApiError(error)),
);
```

Проверь установленную версию: `axios.isCancel` и `error.code === 'ERR_CANCELED'` — актуальные признаки отмены.

```ts
// не надо — сырой payload / PII в toast; cancel как 500
toast.error(JSON.stringify(error.response.data));
toast.error(error.response.data.email);
catch (error) {
  showFormError(String(error));
}
```

**Почему.** Тело 500, validation object и email не для toast. Отмена не должна открывать error-state формы.

**Как исправить.** Узкий `ApiError`. Field errors — в `frontend-forms`. Cancel проглатывай в caller.

## 4. Upload: FormData, boundary браузера

Не ставь `Content-Type: multipart/form-data` вручную. Axios сам выставит header с boundary, если body — `FormData` ([официальные docs Axios](https://github.com/axios/axios#using-multipartform-data-format)). Если у instance уже стоит `application/json` по умолчанию — сбрось Content-Type на этом вызове (`undefined` / как принято в установленной версии), не подменяй ручным multipart.

```ts
// надо
export async function uploadThingFile(
  file: File,
  signal?: AbortSignal,
): Promise<Thing> {
  const body = new FormData();
  body.append('file', file);

  const { data } = await http.post<Thing>('/things/file', body, { signal });
  return data;
}
```

Progress (`onUploadProgress`) — только если этот instance его уже использует; не ставь другой клиент ради прогресса.

```ts
// не надо — ручной multipart; File в JSON; token в params download
await http.post('/things/file', body, {
  headers: { 'Content-Type': 'multipart/form-data' },
});

await http.post('/things/file', { file });

await http.get(`/things/${id}/file`, { params: { token: accessToken } });
```

**Почему.** Header без boundary ломает multipart. JSON не передаёт `File`. Токен в query — та же утечка, что в §1.

**Как исправить.** `FormData` без ручного Content-Type. Download — `responseType: 'blob'` и auth instance, не секрет в URL.
