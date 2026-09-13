---
name: frontend-api
description: Реализует или изменяет клиентский HTTP/REST по фактическому transport и server-state стеку зоны — client, endpoint, query/mutation, DTO, cache, retry/abort, upload/download. Use when implementing or changing an HTTP client, endpoint, query, mutation, DTO mapping, cache invalidation, retry, abort, upload or download. Не для placement/public API/owner state, form binding, UI-состояний, security-аудита, GraphQL, SSE или WebSocket. Не для route tree, client navigation и URL-state маршрута — это frontend-routing. Не для решения, нужен ли странице SSR/RSC/island — это frontend-ssr; HOW loader/RSC fetch, dehydrate и query keys остаётся здесь. Не ставит второй клиент без новой transport/auth/origin границы.
argument-hint: "[endpoint | query | mutation]"
---

# Frontend API

Реализуй **клиентский HTTP/REST** по **уже доминирующему стеку этой зоны**: transport, endpoint, query/mutation, DTO, cache/invalidation, retry/abort, upload/download. Отдельно определи transport (`fetch`, Axios, Ky, generated client или другой) и server-state слой (нет, RTK Query, TanStack Query, SWR или другой). Не выбирай их «по умолчанию» и не ставь второй слой «для удобства».

Следуй режиму запроса пользователя. При реализации используй чтение, поиск, правку и существующие проверки репозитория; при вопросе, диагностике или аудите оставайся read-only. Runtime-логи request/response, token и PII не добавляй.

Правила транспорта, кэша, ошибок и примеры **не копируй в этот файл**. Читай только нужные файлы из `references/`. Примеры — максимум один для transport и один для server-state слоя, которые реально используются вместе; не открывай все `examples-*`.

Маршруты (формулировка запроса → скил):

- «куда положить API / кто владеет state» → [`frontend-architecture`](../frontend-architecture/SKILL.md)
- «endpoint / query / mutation / DTO / cache / retry» → этот скил
- «route tree / client redirect / params / URL-state» → [`frontend-routing`](../frontend-routing/SKILL.md)
- «submit / field errors на форме» → [`frontend-forms`](../frontend-forms/SKILL.md)
- «скелетон / toast / empty UI» → [`frontend-layout`](../frontend-layout/SKILL.md)
- «аудит хранения токена / public env» → [`frontend-security`](../frontend-security/SKILL.md)
- «OWASP / CSRF / server auth» → [`aif-security-checklist`](../../aif-security-checklist/SKILL.md)
- «нужен ли SSR / гидрация / server-data граница» → [`frontend-ssr`](../frontend-ssr/SKILL.md)

Нужен ли странице SSR/RSC/island — [`frontend-ssr`](../frontend-ssr/SKILL.md); HOW loader/RSC fetch, dehydrate и query keys остаётся здесь. GraphQL, SSE и WebSocket — отдельные транспорты: REST-рецепты к ним не применяй. Если задача сразу про **новый слайс и API** — сначала placement/ownership (architecture), затем HTTP-контракт (здесь), затем forms/layout при необходимости. Security запускай только по явному запросу на аудит.

**Auth.** Подключение `credentials`, auth header и refresh/retry по **уже существующему контракту зоны** — этот скил. Размещение auth/session-модуля — [`frontend-architecture`](../frontend-architecture/SKILL.md). Место хранения токена не выбирай и не переноси без отдельного запроса. Не канонизируй `localStorage`, cookie или иной storage. Выбор или аудит token storage — [`frontend-security`](../frontend-security/SKILL.md); серверные cookie flags / CSRF / authorization — [`aif-security-checklist`](../../aif-security-checklist/SKILL.md).

**Режим** (не грузи лишнее):

| Задача | Что читать |
| --- | --- |
| Стек неясен / смешанный / какие слои / GraphQL·SSE·WS граница | [references/discovery-and-boundaries.md](references/discovery-and-boundaries.md) |
| URL, headers, body, status, DTO mapping | [references/http-transport-and-contracts.md](references/http-transport-and-contracts.md) |
| Query keys/tags, cache, cancellation, pagination | [references/queries-cache-and-concurrency.md](references/queries-cache-and-concurrency.md) |
| Mutation, retry, invalidation, ошибки, side effects | [references/mutations-errors-and-effects.md](references/mutations-errors-and-effects.md) |
| credentials/auth по контракту, upload/download, empty/offline | [references/auth-files-and-edge-cases.md](references/auth-files-and-edge-cases.md) |
| Transport: fetch / Axios / Ky | соответствующий `examples-fetch.md` / `examples-axios.md` / `examples-ky.md` |
| Server-state: RTK Query / TanStack Query / SWR | соответствующий `examples-rtk-query.md` / `examples-tanstack-query.md` / `examples-swr.md` |
| Generated/framework-native/другой слой без examples-файла | [discovery-and-boundaries.md](references/discovery-and-boundaries.md) + тематический reference + локальный аналог и official docs установленной версии |

## Шаг 1. Классифицируй задачу

Опора: сообщение пользователя, `$ARGUMENTS` (если скил вызван с аргументом), открытые файлы. В обычном чате `$ARGUMENTS` может быть пустым — это не стоп.

Используй routing-таблицу выше. Несколько сигналов сразу — сначала граница и владелец (architecture), затем HTTP-контракт (здесь), потом forms/layout. GraphQL, SSE и WebSocket не переводи на REST.

## Шаг 2. Обнаружь transport и server-state стек

Смотри репозиторий, не привычку модели. Не выбирай transport или server-state слой по популярности.

Что проверить (достаточно сигналов, не полный аудит репо):

- манифест и lockfile: `package.json` — `axios`, `ky`, `@reduxjs/toolkit`, `@tanstack/react-query`, `swr`, generated/framework-native client;
- центральный client зоны (`baseApi`, axios instance, `fetcher`, `apiClient`) и его импорты;
- импорты в **зоне изменения** (`createApi`, `fetchBaseQuery`, `useQuery`, `useMutation`, `useSWR`, `ky`, `axios`, `fetch`);
- framework loaders (`loader`, `server action`) и generated client — только если зона ими уже пользуется;
- как соседи подключают auth (credentials/header/refresh) и нормализуют ошибки.

| Что видно в зоне | Слой | Что читать |
| --- | --- | --- |
| Нативный `fetch` / `Request` | transport: fetch | [examples-fetch.md](references/examples-fetch.md) |
| Доминирует `axios` | transport: Axios | [examples-axios.md](references/examples-axios.md) |
| Доминирует `ky` | transport: Ky | [examples-ky.md](references/examples-ky.md) |
| `createApi`, `fetchBaseQuery`, RTK Query хуки | server-state + transport adapter: RTK Query | [examples-rtk-query.md](references/examples-rtk-query.md); transport example — только при custom baseQuery |
| `@tanstack/react-query`, `useQuery`, `QueryClient` | server-state: TanStack Query | [examples-tanstack-query.md](references/examples-tanstack-query.md) + transport зоны |
| `swr`, `useSWR` | server-state: SWR | [examples-swr.md](references/examples-swr.md) + transport зоны |
| GraphQL-клиент, `EventSource`, `WebSocket` / socket.io | вне HTTP/REST | [discovery-and-boundaries.md](references/discovery-and-boundaries.md) — граница; рецепты REST не читай |
| Generated/framework-native/другой слой без `examples-*.md` | существующий слой зоны | discovery + тематический reference + локальный аналог + official docs **установленной версии** |
| В репо несколько transport/client, в этой зоне один прецедент | transport зоны и его trust boundary | examples только выбранных слоёв |
| Сигналов мало или в зоне два равноправных способа | спроси | не ставь пакет и не переключай зону на известный скилу слой |

Не устанавливай `axios`, `ky`, RTK Query, TanStack Query, SWR, codegen или дублирующий client той же origin/auth/transport границы, пока пользователь явно не попросил. Следуй уже стоящему клиенту зоны: не подставляй fetch в axios-проект и наоборот.

Если библиотека распознана, но отдельного examples-файла для неё нет, это **не** повод переводить зону на fetch / Axios / Ky / RTK Query / TanStack Query / SWR. Сохрани её модель client/query/mutation, возьми API из установленной версии и ближайшего локального аналога, а универсальные требования к транспорту, cache, ошибкам и auth — из тематических references этого скила.

## Шаг 3. Выбери references и аналог

По типу задачи читай только нужное. Для `TanStack Query + Axios`, например, нужны `examples-tanstack-query.md` и `examples-axios.md`, но не остальные examples.

| Задача | Тематический reference сверх уже выбранных файлов режима |
| --- | --- |
| URL/params/headers/body, status, Content-Type, DTO mapping | [http-transport-and-contracts.md](references/http-transport-and-contracts.md) |
| keys/tags, stale, dedupe, abort, pagination, SSR loader | [queries-cache-and-concurrency.md](references/queries-cache-and-concurrency.md) |
| mutation, idempotency/retry, invalidation, optimistic, ошибки | [mutations-errors-and-effects.md](references/mutations-errors-and-effects.md) |
| credentials/header/refresh, upload/download, empty, offline | [auth-files-and-edge-cases.md](references/auth-files-and-edge-cases.md) |

Паттерн бери у **ближайшего HTTP/query того же типа в этой зоне**, не у первого попавшегося поиска и не из другого репозитория. Переиспользуй client той же `origin + auth + transport` границы. Отдельный client допустим для доказанно другой origin/auth-политики; не отправляй внутренние credentials стороннему origin. Имена и импорты — как **здесь**.

Чужие пути и alias не переноси в проект, где их нет. Нет близкого аналога — читай examples выбранных transport/server-state слоёв. Нет examples-файла — тематические references + локальный аналог + official docs установленной версии. Всё ещё пусто или спорно — спроси, не выдумывай канон и не переключай стек.

Optimistic update, infinite query, SSR loaders и codegen — не default: только при локальном прецеденте или явном запросе.

## Шаг 4. Реализуй HTTP-контракт

По выбранным слоям и аналогу:

- один base client на совместимую `origin + auth + transport` границу; новый экземпляр не заводи, если подходящий уже есть;
- query — чтение; изменение — mutation; side-effecting query не делай;
- TypeScript: typed Request/Response DTO; JavaScript: JSDoc/runtime contract по локальному стилю. Явный mapper только при различии wire/domain shape, identity-mapper не создавай;
- runtime validation ответа — только если в зоне уже есть принятый механизм или пользователь явно попросил; schema library не ставь;
- `AbortSignal` (или эквивалент трека) от consumer до транспорта;
- `credentials` / auth header / refresh — по существующему контракту, без смены storage;
- field-level ошибки нормализуй здесь, к форме применяет [`frontend-forms`](../frontend-forms/SKILL.md);
- UI loading/error/empty не верстай здесь — [`frontend-layout`](../frontend-layout/SKILL.md).

Не копируй локальные anti-patterns даже из этого репо: дублирующий client той же границы, запрос без проверки HTTP status, token в URL, безусловный retry неидемпотентного POST. Как надо / как не надо — в examples выбранных слоёв (или в тематических references, если examples-файла нет).

После транспорта, если нужна форма — передай поведение в [`frontend-forms`](../frontend-forms/SKILL.md). Куда лежат файлы — уже решено architecture; здесь файлы не переезжают.

## Шаг 5. Чеклист перед сдачей

Если пункт не выполнен — почини, не сдавай. Пункты про конкретный инструмент — только если он **есть** в репозитории.

- **Стек.** Отдельно определены transport и server-state слой. Новый HTTP/query-клиент и codegen не установлены без явного запроса; прочитано не больше одного examples-файла на каждый реально используемый слой.
- **Границы.** Placement/public API не решались здесь. Form submit/field errors не биндились здесь. UI-состояния не верстались здесь. Security-аудит не запускался без явного запроса.
- **Транспорт.** Один base client на совместимую origin/auth границу; DTO/контракт описаны по языку зоны; identity-mapper нет; секреты не в query string.
- **Auth.** Credentials/header/refresh — по существующему контракту; storage токена не выбран и не перенесён.
- **Данные.** Request/response, token и PII не логируй.
- **Условные возможности.** Optimistic / infinite / SSR / codegen — только при прецеденте или явном запросе.
- **Проверки репозитория.** Запусти существующие targeted tests, lint и typecheck по затронутой области. При изменении поведения обнови тесты в принятом стеке; новый test runner и runtime-логи не добавляй.

Перед сдачей — коротко, не эссе:

- **Стек:** transport + server-state слой (и что спросили у пользователя).
- **Доказательства:** 2–3 факта из репо (зависимость, импорт зоны, ближайший client).
- **Fallback:** если examples-файла не было — какие тематические references, аналог и official docs версии использованы.
- **Что не копировали:** если в зоне были anti-patterns второго client / status / retry / token-in-URL — что сделали иначе.

## Если что-то пошло не так

- Стек неясен или в зоне два равноправных способа → спроси; не подставляй знакомый модели слой.
- Нет examples-файла → тематический reference + локальный аналог + official docs установленной версии; стек не переключай.
- Нет аналога → ищи соседний HTTP/query этого репо; чужие импорты и alias не копируй.
- Пользователь не просил новый слой → не добавляй второй client/query cache и не ставь codegen.
- Scope ушёл в соседнюю область → вернись к routing-таблице; GraphQL/SSE/WebSocket не конвертируй в REST.
- Client navigation / route tree / URL-state без HTTP-контракта → [`frontend-routing`](../frontend-routing/SKILL.md), не этот скил.
- Нужен ли SSR / гидрация / server-data граница, а не HOW dehydrate → [`frontend-ssr`](../frontend-ssr/SKILL.md), не этот скил.
- Пользователь не просил сменить token storage → не выбирай и не переноси `localStorage`, cookie или иной storage.
