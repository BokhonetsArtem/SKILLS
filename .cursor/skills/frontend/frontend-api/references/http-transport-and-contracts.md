# HTTP-транспорт и контракты

Читай, когда нужны URL, params, headers, body, HTTP status, Content-Type или DTO mapping. Transport и server-state слои уже выбраны ([discovery-and-boundaries.md](discovery-and-boundaries.md)). Синтаксис бери максимум из одного transport example и одного реально используемого server-state example.

Рекомендации ниже **библиотечно-нейтральны**. Опора: [MDN Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API), [AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController), [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS), [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) (HTTP Semantics), [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html) (Problem Details). Axios/Ky/query-слой могут бросать на 4xx/5xx сами — это отличие трека, не отмена правил status и DTO.

Не ставь schema library «для валидации ответа». Не заводи второй HTTP-клиент внутри той же origin/auth/transport границы.

## 1. Один base client на trust boundary

Базовый HTTP/query-клиент совместимой `origin + auth + transport` границы (instance, `baseApi`, `fetcher`) — единая точка `baseUrl`, credentials, default headers и refresh. Новый экземпляр не заводи, если подходящий уже есть. Для другой origin или auth-политики отдельный client может быть необходим: общий instance не должен отправлять внутренние credentials стороннему API.

- Endpoint зоны импортирует этот client, не создаёт локальный `axios.create` / второй `createApi` / голый `fetch` «только здесь».
- Таймаут, interceptors и `prepareHeaders` расширяй в существующем client, не копируй их в фичу.
- Чужой alias и путь client из другого репо не переноси.

Как не надо: `fetch` рядом с подходящим axios instance; второй `createApi` только ради нового endpoint; копипаста headers в каждый вызов; общий auth-interceptor для доверенного и стороннего origin.

## 2. Status, тело, Content-Type

`fetch` **не** бросает на HTTP 4xx/5xx: сеть/CORS/abort — rejection; HTTP-ошибка — `response.ok === false`. Статус 200–299 (включая 204) — `ok === true` ([MDN `Response.ok`](https://developer.mozilla.org/en-US/docs/Web/API/Response/ok)). Клиенты вроде Axios по умолчанию трактуют 4xx/5xx как ошибку — следуй API выбранного трека, но **не** считай «промис зарезолвился» успехом без проверки status/`ok`.

| Ответ | Как читать тело |
| --- | --- |
| 204 / 205 / пустой body | Не вызывай `json()` на пустом теле. Успех без payload — `void` / `undefined`, как у соседей |
| `application/json` (или принятый JSON-подтип зоны) | Parse JSON **после** проверки успеха или в error-нормализаторе |
| `text/plain`, HTML-ошибка шлюза | `text()`, не слепой `json()` |
| Файл / download | `blob()` / `arrayBuffer()`, не JSON |
| Нет / чужой Content-Type при ожидании JSON | Не парси наугад; нормализуй как transport error |

Не делай `as Thing` / `as ApiError` на сыром body без типа и без принятого parse. Слепой каст не заменяет контракт.

RFC 9457 (`application/problem+json`, поля `type` / `title` / `status` / `detail` / `instance`) применяй **только** когда API зоны действительно так отвечает. Иначе нормализуй фактический error contract (`message`, `{ errors: [] }`, код+текст) — не подменяй его Problem Details «по стандарту».

GET/HEAD без тела запроса: не прикручивай JSON-`Content-Type` и body «на всякий случай» ([RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) не задаёт семантику body у GET).

## 3. URL, params, заголовки

Собирай URL штатными helpers зоны (`URL`, `URLSearchParams`, `params`-опция клиента, route-helper). Не конкатенируй строки и не клеи `?a=` + encode вручную, если client это умеет.

- Каждый динамический path segment — через route helper зоны или корректное кодирование отдельного segment. Не кодируй весь path целиком и не интерполируй сырой `id`, если он может содержать `/`, `?` или `#`.
- Query — объект params клиента или `URLSearchParams`. Не клади токен, пароль, session id и прочие секреты в query string (история, Referer, логи). Идентификатор вроде `email` для side-effecting действия — в body, не в URL, если контракт это позволяет; существующий query-контракт читай, но новый секрет в query не добавляй.
- `Authorization` / cookie — как в base client, не дублируй в каждом endpoint. Подробности — [auth-files-and-edge-cases.md](auth-files-and-edge-cases.md).
- CORS на клиенте не «чинят» заголовками `Access-Control-*`. Режим `credentials` / CORS — как у существующего контракта; не переключай `omit`/`include` без запроса. Аудит CORS — не этот скил.

## 4. DTO: типы, mapper, validation

Типизируй Request и Response отдельно от UI и form values. Имена — как у соседей зоны (`ThingDto`, `Thing`, `CreateThingBody`), не третий стиль рядом.

| Ситуация | Что делать |
| --- | --- |
| Wire JSON совпадает с domain shape | Тип ответа и есть контракт. **Identity-mapper не создавай** (`toThing = (dto) => dto`, пустой `transformResponse`) |
| Имена, nullability, вложенность отличаются | Явный mapper / `transformResponse` **в одну сторону** (wire → domain и обратно на write). Без «универсального» `mapAny` |
| Form values отличаются от API DTO | Маппинг form ↔ DTO — на границе submit ([`frontend-forms`](../../frontend-forms/SKILL.md)); HTTP-слой принимает уже API DTO |
| Generated client уже даёт типы | Используй их; не дублируй ручными интерфейсами «чтобы было проще» |

```ts
// иллюстрация, не предписание fetch
type ThingWire = { id: string; display_name: string };
type Thing = { id: string; title: string };

function toThing(dto: ThingWire): Thing {
  return { id: dto.id, title: dto.display_name };
}
```

Runtime validation ответа (`zod.parse`, valibot и т.п.) — **только** если в зоне уже есть принятый механизм на HTTP-ответах или пользователь явно попросил. Schema library не ставь. Client-side parse **не** замена серверной валидации. Не пиши, что «схема на клиенте достаточно».

Не склеивай form values, schema формы и API DTO в один тип «на всякий случай».

## 5. AbortSignal: cancel, timeout, сеть, HTTP

`AbortSignal` (или эквивалент трека: `CancelToken` старше Axios, `signal` у ky/query) обязан доходить от consumer до транспорта.

| Исход | Как отличать | Что делать |
| --- | --- | --- |
| Отмена пользователя / размонтирование / смена ключа | `AbortError` / `CanceledError` / имя `AbortError` у `DOMException` | Не показывать как HTTP-ошибку; не retry |
| Timeout | `AbortSignal.timeout`, timeout опции клиента, `TimeoutError` трека | Отдельный тип; retry — только если операция безопасна ([mutations-errors-and-effects.md](mutations-errors-and-effects.md)) |
| Сеть / DNS / CORS fail | `TypeError` у fetch; network code клиента | Транспортная ошибка без тела HTTP |
| HTTP 4xx/5xx | `ok === false` / error status клиента | Нормализовать по контракту зоны; не путать с abort |

Не глотай abort как `toast.error`. Не оставляй «висящий» fetch без signal, если consumer уже умеет его передать. Query-слой зоны, который сам отменяет при смене ключа, — переиспользуй, не дублируй ручным `AbortController` рядом, если это ломает дедуп клиента.

## 6. FormData и multipart

Тело файла собирай через `FormData`. Boundary выставляет **браузер** вместе с `Content-Type`. Не ставь вручную `Content-Type: multipart/form-data` (без boundary запрос ломается).

JSON-поля + файл — как у ближайшего upload зоны: либо всё в `FormData`, либо отдельный upload endpoint. Не сериализуй `File` через `JSON.stringify`.

Download, empty body, 429 и offline — [auth-files-and-edge-cases.md](auth-files-and-edge-cases.md).

## 7. Чего не делать

- Второй HTTP/query-клиент внутри той же origin/auth/transport границы «только для этого экрана».
- Считать resolved promise успехом без проверки HTTP status / `ok`.
- `response.json()` на 204 и пустом теле.
- Секреты и токены в query string.
- Identity-mapper и слепой `as Thing`.
- Ставить zod/yup на каждый ответ, если зона этого не делает.
- Подменять фактический error JSON на RFC 9457 без прецедента.
- Логировать request/response, token и PII.
