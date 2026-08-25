# Discovery и границы

Читай, когда transport/server-state стек неясен, в репо несколько клиентов, установленный слой **без** `examples-*.md`, или транспорт не HTTP/REST. Выбирай transport и server-state независимо: максимум один examples-файл каждого реально используемого слоя. Транспорт, cache, mutation и auth — соседние файлы этой папки.

Не ставь `axios`, `ky`, RTK Query, TanStack Query, SWR, codegen или дублирующий client той же origin/auth/transport границы, пока пользователь явно не попросил.

## 1. Это не задача этого скила

Слово «API» в UI не равно HTTP-контракту. Если нет клиента, endpoint, query/mutation, DTO, cache/invalidation, retry/abort, upload/download — остановись и отдай другому скилу.

| Что видно | Куда |
| --- | --- |
| Куда класть модуль, public API, кто владеет server/client/form/session state | [`frontend-architecture`](../../frontend-architecture/SKILL.md) |
| Form state, submit binding, применение field errors | [`frontend-forms`](../../frontend-forms/SKILL.md) |
| UI loading / error / empty, скелетон, toast-вёрстка | [`frontend-layout`](../../frontend-layout/SKILL.md) |
| Явный source-to-sink аудит token storage / public env / client token flow | [`frontend-security`](../../frontend-security/SKILL.md) |
| Generic OWASP, server auth, CSRF, cookie flags, CVE | [`aif-security-checklist`](../../../aif-security-checklist/SKILL.md) |
| GraphQL-клиент, `EventSource` / SSE, `WebSocket` / socket.io как транспорт | вне scope — §6; REST-рецепты не применяй |

Обычное подключение `credentials`, auth header и refresh/retry по **уже существующему** контракту зоны — этот скил ([auth-files-and-edge-cases.md](auth-files-and-edge-cases.md)). Размещение auth/session-модуля — architecture. Место хранения токена не выбирай и не переноси.

Смешанная задача: сначала размещение (architecture), затем HTTP-контракт (здесь), затем форма (forms) и оболочка (layout), если они нужны. Security — только по явному запросу на аудит.

## 2. Сначала репозиторий, затем решение

Смотри **зону изменения**, не популярность библиотеки и не «дефолт модели». Достаточно сигналов, не полный аудит репо.

Порядок:

1. Манифест и lockfile: `package.json` — `axios`, `ky`, `@reduxjs/toolkit`, `@tanstack/react-query`, `swr`, generated client (`openapi-fetch`, orval, hey-api и аналоги).
2. Центральный client зоны: `baseApi`, axios instance, `fetcher`, `apiClient`, ky-инстанс — и его импорты.
3. Импорты **в зоне изменения**: `createApi`, `fetchBaseQuery`, `useQuery`, `useMutation`, `useSWR`, `ky`, `axios`, `fetch`.
4. Framework loaders (`loader`, server action, RSC fetch) и generated client — **только** если зона ими уже пользуется.
5. Как соседи подключают auth (credentials / header / refresh) и нормализуют ошибки.

Имя файла `*Api*` само по себе ничего не доказывает. Смотри вызов сети, query-слой и base client.

## 3. Две независимые оси стека

Сначала выбери transport, затем опциональный server-state слой. `TanStack Query + Axios` и `SWR + fetch` — нормальные комбинации, а не «смешанный трек».

| Что видно в зоне | Слой | Что читать |
| --- | --- | --- |
| Нативный `fetch` / `Request` | transport: fetch | `examples-fetch.md` |
| Доминирует `axios` | transport: Axios | `examples-axios.md` |
| Доминирует `ky` | transport: Ky | `examples-ky.md` |
| Generated/framework-native HTTP client | transport: существующий | локальный аналог + official docs версии |
| `createApi`, `fetchBaseQuery`, RTK Query хуки | server-state + transport adapter: RTK Query | `examples-rtk-query.md`; transport example — только при custom baseQuery |
| `@tanstack/react-query`, `useQuery`, `QueryClient` | server-state: TanStack Query | `examples-tanstack-query.md` + transport зоны |
| `swr`, `useSWR` | server-state: SWR | `examples-swr.md` + transport зоны |
| GraphQL-клиент, `EventSource`, `WebSocket` / socket.io | вне HTTP/REST | §6; рецепты REST не читай |
| Другой HTTP/query-слой **без** `examples-*.md` | существующий слой зоны | этот файл + тематический reference + локальный аналог + official docs **установленной версии** |
| В репо несколько клиентов, в этой зоне один прецедент | transport его origin/auth границы | examples выбранных слоёв |
| Сигналов мало или в зоне два равноправных способа | спроси | не ставь пакет и не переключай зону на известный скилу слой |

В lockfile одна библиотека, в **этой** зоне импорты другой — transport **зоны**, не «главный» пакет репо.

Fetch внутри Axios/Ky — не отдельный transport проекта. `fetchBaseQuery` — transport adapter RTK Query; custom baseQuery может использовать существующий Axios/Ky client. Не подставляй «голый» `fetch` в Axios-проект и наоборот.

## 4. Fallback без examples-файла

Если библиотека распознана, но отдельного `examples-*.md` для неё нет, это **не** повод переводить зону на fetch / Axios / Ky / RTK Query / TanStack Query / SWR.

1. Сохрани модель client / query / mutation **этой** библиотеки.
2. API и имена бери из установленной версии и ближайшего локального аналога того же типа.
3. Универсальные требования к транспорту, cache, ошибкам и auth — из тематических files этой папки.
4. Version-specific детали сверяй с official docs установленной версии; не опирайся на «как обычно в индустрии».

Generated client, который уже стоит в зоне, — часть стека, не второй клиент. Новый codegen не добавляй без явного запроса.

## 5. Смешанный и неясный стек

| Ситуация | Решение |
| --- | --- |
| В репо несколько клиентов, в зоне один устойчивый прецедент | Transport той же origin/auth границы; server-state выбери отдельно |
| В одном endpoint уже смешаны два клиента / query-слоя | Не усугубляй. Новую логику — в уже выбранный способ этого endpoint |
| Framework loader уже владеет fetch этой страницы | Сохрани loader/hydration контракт зоны; не создавай параллельный client query «на всякий случай» |
| Два равноправных способа в зоне или сигналов мало | Спроси. Не ставь пакет и не выбирай «как принято» |
| Пользователь явно просит другую библиотеку | Можно ставить только после явного запроса; не мигрируй соседние endpoint заодно |

Паттерн бери у **ближайшего HTTP/query того же типа в этой зоне** (list, detail, create, upload), не у первого попавшегося поиска и не из другого репозитория. Переиспользуй base client совместимой `origin + auth + transport` границы. Для другой origin/auth-политики отдельный client может быть необходим; не отправляй внутренние credentials стороннему origin. Имена и импорты — как **здесь**. Чужие пути и alias не переноси туда, где их нет.

Reuse **не** значит копировать дыры: второй client, запрос без проверки HTTP status, token в URL, безусловный retry неидемпотентного POST, side-effecting query. Локальные anti-patterns не закрепляй как good path.

Optimistic update, infinite query, SSR loaders и codegen — не default: только при локальном прецеденте или явном запросе.

## 6. Non-HTTP транспорт

GraphQL, SSE (`EventSource`) и WebSocket / socket.io — **другие транспорты**, не «REST с другим синтаксисом».

- Не переводи их на HTTP/JSON «для единообразия».
- Не применяй query keys/tags, `response.ok` и JSON-DTO рецепты этого скила как канон для этих транспортов.
- Глубокого руководства здесь нет: остановись, зафиксируй границу, спроси или отдай владельцу уже существующего GraphQL/WS-слоя зоны.
- Смешанный экран (REST list + WS presence) — REST-часть веди этим скилом, realtime-часть не переписывай в REST.

## 7. Что читать дальше

| Нужно | Файл |
| --- | --- |
| URL/params/headers/body, status, Content-Type, DTO mapping | [http-transport-and-contracts.md](http-transport-and-contracts.md) |
| keys/tags, stale, dedupe, abort, pagination, SSR loader | [queries-cache-and-concurrency.md](queries-cache-and-concurrency.md) |
| mutation, idempotency/retry, invalidation, optimistic, ошибки | [mutations-errors-and-effects.md](mutations-errors-and-effects.md) |
| credentials/header/refresh, upload/download, empty, offline | [auth-files-and-edge-cases.md](auth-files-and-edge-cases.md) |
| Сниппеты | максимум один transport example + один server-state example; нет файла — тематические references + аналог + official docs версии |
