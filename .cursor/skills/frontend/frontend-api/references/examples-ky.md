# Примеры: Ky

Читай этот файл как transport-рецепт, если зона использует Ky. Другие transport examples (fetch/Axios) не открывай. Если поверх Ky есть TanStack Query/SWR или RTK Query с custom baseQuery, дополнительно читай только соответствующий server-state example.

Сниппеты — обобщённые рецепты. Не копируй `@/`, имена слайсов и пути `src/` как канон. Instance и hooks — как у **соседей этой зоны**. Runtime-логи request/response, token и PII не добавляй.

Правила транспорта, cache, mutation и auth — в соседних файлах этой папки.

**Версия.** Проверь установленную Ky: v1 — опция `prefixUrl`, путь **без** ведущего `/`. v2 — `prefix` / `baseUrl` (см. README своей версии). Сниппеты ниже — `prefixUrl`; подставь имена опций своей версии. `.json()` в текущих версиях бросает на пустом теле и 204 — не вызывай его, если ответа нет.

Как читать пары: сверху **надо**, ниже **не надо**, затем почему.

## 1. List/detail: один instance, searchParams, signal

Ky по умолчанию бросает `HTTPError` на статусах вне 200–299 и отдаёт `.json<T>()`. Params — `searchParams`, не склейка строки. `signal` доходит до запроса.

```ts
// надо
import { http } from './http'; // существующий client этой origin/auth границы

type Thing = { id: string; title: string };
type ThingListParams = { q?: string; page: number };

export function getThings(params: ThingListParams, signal?: AbortSignal) {
  return http
    .get('things', {
      signal,
      searchParams: {
        page: params.page,
        q: params.q,
      },
    })
    .json<Thing[]>();
}

export function getThing(id: string, signal?: AbortSignal) {
  if (!id) {
    return Promise.reject(new Error('Missing thing id'));
  }

  return http.get(`things/${encodeURIComponent(id)}`, { signal }).json<Thing>();
}
```

В `searchParams` не клади token. `undefined` в объекте params у Ky удаляет ключ (проверить версию) — не отправляй `'undefined'` строкой.

```ts
// не надо — fetch/axios рядом; ведущий / с prefixUrl v1; token в query
await fetch(`${apiBaseUrl}/things?token=${accessToken}`);
await http.get('/things', { searchParams: { page, token: accessToken } });
const thing = (await http.get('things/1')) as Thing;
```

**Почему.** Второй клиент разъезжается с hooks и retry. В Ky v1 ведущий `/` ломает `prefixUrl`. Токен в query — история и Referer. Ky возвращает `Response`/JSON через `.json()`, а не «уже Thing».

**Как исправить.** Один `ky.create` зоны. Пути — по правилам своей версии. Секреты — в header/cookie контракта.

## 2. Create/update: JSON body, без retry POST

Default retry Ky — для `get` / `put` / `head` / `delete` / `options` / `trace` / `query`, **не** для POST ([README Ky](https://github.com/sindresorhus/ky#retry)). Не включай POST в `retry.methods`. Тело JSON — опция `json`, не ручной `JSON.stringify` + header, если instance так умеет.

```ts
// надо
type CreateThingBody = { title: string };

export function createThing(body: CreateThingBody, signal?: AbortSignal) {
  return http.post('things', { json: body, signal }).json<Thing>();
}

export function updateThing(
  id: string,
  body: CreateThingBody,
  signal?: AbortSignal,
) {
  return http
    .patch(`things/${encodeURIComponent(id)}`, { json: body, signal })
    .json<Thing>();
}

export async function deleteThing(id: string, signal?: AbortSignal): Promise<void> {
  const response = await http.delete(`things/${encodeURIComponent(id)}`, { signal });
  if (response.status === 204) {
    return;
  }
}

// сценарий
await createThing(body);
onCreated();
```

```ts
// не надо — retry POST; toast до await; .json() на 204
export const http = ky.create({
  prefixUrl: apiBaseUrl,
  retry: { limit: 3, methods: ['get', 'post'] },
});

toast.success('Saved');
await http.post('things', { json: body });

await http.delete(`things/${id}`).json();
```

**Почему.** Повтор неидемпотентного создания даёт дубликаты. Toast до fulfilled врёт при 422. `.json()` на 204 в актуальном Ky бросает.

**Как исправить.** Не расширяй `retry.methods` на POST. Эффект — после await. 204 — без `.json()`.

## 3. Ошибки: HTTPError, не сырой body

Лови `HTTPError` (в новых версиях есть и `isHTTPError` — проверить). Для UI бери `status` и безопасный `message`. Не клади `await error.response.json()` в toast. Timeout/abort отличай от HTTP.

```ts
// надо
import ky, { HTTPError, TimeoutError } from 'ky';

type ApiError = {
  status: number;
  message: string;
  fieldErrors?: Record<string, string>;
};

export async function toApiError(error: unknown): Promise<ApiError> {
  if (error instanceof DOMException && error.name === 'AbortError') {
    throw error;
  }

  if (error instanceof TimeoutError) {
    return { status: 0, message: 'Request timed out' };
  }

  if (error instanceof HTTPError) {
    const status = error.response.status;
    const payload: unknown = await error.response
      .clone()
      .json()
      .catch(() => undefined);
    return {
      status,
      message: readSafeMessage(payload),
      fieldErrors: readFieldErrors(payload),
    };
  }

  return { status: 0, message: 'Request failed' };
}
```

Имена классов ошибок сверяй с установленной версией (`HTTPError` / `TimeoutError` / type guards).

```ts
// не надо — сырой payload; PII; abort как ошибка экрана
catch (error) {
  if (error instanceof HTTPError) {
    toast.error(await error.response.text());
    toast.error((await error.response.json()).email);
  }
}
```

**Почему.** HTML шлюза, stack и email не для UI. Abort не должен открывать error-state.

**Как исправить.** Узкий `ApiError`. Field errors — в `frontend-forms`. Abort проглатывай.

## 4. Upload: FormData без ручного Content-Type

Ky принимает `body: FormData`. Не ставь `Content-Type: multipart/form-data` — boundary выставит браузер. Не сериализуй `File` через `json:`.

```ts
// надо
export function uploadThingFile(file: File, signal?: AbortSignal) {
  const body = new FormData();
  body.append('file', file);

  return http.post('things/file', { body, signal }).json<Thing>();
}
```

Для большого streaming body README Ky советует `retry: { limit: 0 }` — не ретрай upload без запроса.

```ts
// не надо — ручной multipart; json: { file }; token в searchParams
await http.post('things/file', {
  body,
  headers: { 'Content-Type': 'multipart/form-data' },
});

await http.post('things/file', { json: { file } });

await http.get('things/file', { searchParams: { id, token: accessToken } });
```

**Почему.** Ручной multipart ломает boundary. `json` не отправит файл. Токен в query — утечка.

**Как исправить.** `FormData` в `body`. Auth — credentials/header instance. Download — `.blob()` и тот же auth, только если зона уже качает файлы.
