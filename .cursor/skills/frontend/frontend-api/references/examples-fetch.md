# Примеры: fetch

Читай этот файл как transport-рецепт, если зона использует нативный `fetch` / `Request`. Другие transport examples (Axios/Ky) не открывай. Если поверх fetch есть TanStack Query/SWR/RTK Query, дополнительно читай только соответствующий server-state example.

Сниппеты — обобщённые рецепты. Не копируй `@/`, имена слайсов и пути `src/` как канон. Импорты, `credentials` и разбор ошибок — как у **соседей этой зоны**. Runtime-логи request/response, token и PII не добавляй.

Правила транспорта, cache, mutation и auth — в соседних файлах этой папки, не дублируй их сюда целиком.

AbortSignal, download и SSR loader — не default: прокидывай `signal`, если consumer его уже даёт; download/loader — только при локальном прецеденте или явном запросе.

Как читать пары: сверху **надо**, ниже **не надо**, затем почему. Копируй только приёмы fetch.

## 1. List/detail: типы, params, cancellation

Переиспользуй общий `fetch` helper зоны, если он есть; иначе вынеси повторяющиеся base URL, credentials и error parsing при втором use case. Query — через `URLSearchParams`. HTTP 4xx/5xx — не успех: `fetch` **не** бросает на них ([MDN `Response.ok`](https://developer.mozilla.org/en-US/docs/Web/API/Response/ok)). `AbortSignal` доходит до `fetch`.

```ts
// надо
type Thing = { id: string; title: string };
type ThingListParams = { q?: string; page: number };

export async function getThings(
  params: ThingListParams,
  signal?: AbortSignal,
): Promise<Thing[]> {
  const search = new URLSearchParams();
  search.set('page', String(params.page));
  if (params.q) search.set('q', params.q);

  const response = await fetch(`/things?${search}`, {
    method: 'GET',
    signal,
    credentials: 'include', // как у существующего контракта зоны
  });

  if (!response.ok) {
    throw await toApiError(response);
  }

  return readJson<Thing[]>(response);
}

export async function getThing(id: string, signal?: AbortSignal): Promise<Thing> {
  if (!id) {
    throw new Error('Missing thing id');
  }

  const response = await fetch(`/things/${encodeURIComponent(id)}`, {
    method: 'GET',
    signal,
    credentials: 'include',
  });

  if (!response.ok) {
    throw await toApiError(response);
  }

  return readJson<Thing>(response);
}
```

`readJson` / `toApiError` — в §3. Consumer отменяет предыдущий запрос при смене params / размонтировании. `AbortError` — не ошибка загрузки.

```ts
// не надо — нет проверки ok; axios «заодно»; token в query; слепой as
const response = await axios.get(`/things?page=${page}&token=${accessToken}`);
const thing = (await fetch(`/things/${id}`).then((r) => r.json())) as Thing;
```

**Почему.** `fetch` резолвит промис и на 404/500: без `ok` UI рисует ошибку как данные. `axios` в fetch-зоне — второй HTTP-клиент. Токен в query попадает в историю, Referer и логи. Слепой `as Thing` не проверяет status и Content-Type.

**Как исправить.** Тот же helper, что у соседей. Params — `URLSearchParams`, не конкатенация. Секреты — в header/cookie существующего контракта, не в URL. `signal` прокинь, если caller его имеет.

## 2. Create/update: подтверждённый успех, пустое тело

Изменение — отдельная функция с методом по контракту. Toast/навигация — **после** успешного ответа. 204 не парси как JSON.

```ts
// надо
type CreateThingBody = { title: string };

export async function createThing(
  body: CreateThingBody,
  signal?: AbortSignal,
): Promise<Thing | void> {
  const response = await fetch('/things', {
    method: 'POST',
    signal,
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });

  if (!response.ok) {
    throw await toApiError(response);
  }

  if (response.status === 204) {
    return;
  }

  return readJson<Thing>(response);
}

export async function updateThing(
  id: string,
  body: CreateThingBody,
  signal?: AbortSignal,
): Promise<Thing> {
  const response = await fetch(`/things/${encodeURIComponent(id)}`, {
    method: 'PATCH',
    signal,
    credentials: 'include',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });

  if (!response.ok) {
    throw await toApiError(response);
  }

  return readJson<Thing>(response);
}
```

```ts
// не надо — toast до ответа; retry POST; json() на 204
toast.success('Saved');
await fetch('/things', { method: 'POST', body: JSON.stringify(body) });

async function createThingWithRetry(body: CreateThingBody): Promise<Thing> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch('/things', {
      method: 'POST',
      body: JSON.stringify(body),
    });
    return response.json();
  }
  throw new Error('unreachable');
}
```

**Почему.** Toast до `ok` врёт пользователю, если сервер ответил 422. Повтор неидемпотентного создания может дать второй ресурс. `json()` на 204 даёт SyntaxError.

**Как исправить.** Сначала дождись `ok`, потом эффект. POST не ретрай автоматически. Пустое тело — `void`.

## 3. Ошибки: узкий контракт, не сырой payload

HTTP-слой приводит отказ к `ApiError`. В UI — безопасный `message`. Field-level список **нормализуй** здесь; на форму его применяет `frontend-forms`.

```ts
// надо
type ApiError = {
  status: number;
  message: string;
  fieldErrors?: Record<string, string>;
};

function isJsonContentType(contentType: string): boolean {
  const mediaType = contentType.split(';', 1)[0].trim().toLowerCase();
  return mediaType === 'application/json' || mediaType.endsWith('+json');
}

async function readJson<T>(response: Response): Promise<T> {
  const contentType = response.headers.get('content-type') ?? '';
  if (!isJsonContentType(contentType)) {
    throw await toApiError(response);
  }
  return response.json() as Promise<T>;
}

async function toApiError(response: Response): Promise<ApiError> {
  const status = response.status;
  const contentType = response.headers.get('content-type') ?? '';

  if (!isJsonContentType(contentType)) {
    return { status, message: 'Request failed' };
  }

  const payload: unknown = await response.json().catch(() => undefined);
  return {
    status,
    message: readSafeMessage(payload),
    fieldErrors: readFieldErrors(payload),
  };
}

function isAbortError(error: unknown): boolean {
  return error instanceof DOMException && error.name === 'AbortError';
}
```

`readSafeMessage` берёт известное поле контракта зоны (`message` / `title`), не `JSON.stringify` и не email/token/stack. RFC 9457 — только если `Content-Type` действительно `application/problem+json`.

```ts
// не надо — сырой body в toast; PII; abort как ошибка формы
toast.error(JSON.stringify(await response.json()));
toast.error(payload.email);
if (!response.ok) setFormError(await response.text());
```

**Почему.** Сырой 500, HTML шлюза и PII не место в UI. Abort — отмена, не отказ валидации.

**Как исправить.** Один `toApiError`. Abort глотай в caller. Field errors отдай форме, общий `message` — сценарию.

## 4. Upload: FormData без ручного multipart header

Специализированный рецепт. Download (`blob` + `Content-Disposition` + revoke) — только если зона уже качает файлы или пользователь явно просил.

```ts
// надо
export async function uploadThingFile(
  file: File,
  signal?: AbortSignal,
): Promise<Thing> {
  const body = new FormData();
  body.append('file', file);

  const response = await fetch('/things/file', {
    method: 'POST',
    signal,
    credentials: 'include',
    body,
  });

  if (!response.ok) {
    throw await toApiError(response);
  }

  return readJson<Thing>(response);
}
```

Не ставь `Content-Type`: boundary выставляет браузер.

```ts
// не надо — ручной multipart; File через JSON; token в URL download
const body = new FormData();
body.append('file', file);

await fetch('/things/file', {
  method: 'POST',
  headers: { 'Content-Type': 'multipart/form-data' },
  body,
});

await fetch('/things/file', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ file }),
});

window.location.href = `/things/${id}/download?token=${accessToken}`;
```

**Почему.** `Content-Type: multipart/form-data` без boundary ломает разбор тела. `JSON.stringify(File)` не передаёт файл. Токен в query download утекает так же, как в list.

**Как исправить.** Голый `FormData`. Auth — тот же credentials/header, что у JSON. Download — `blob()` и существующий auth, не секрет в URL.
