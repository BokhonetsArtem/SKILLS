# Queries, cache и concurrency

Читай, когда нужны query keys/tags, кэш, отмена гонок, pagination или framework loaders. Транспорт тела и DTO — [http-transport-and-contracts.md](http-transport-and-contracts.md). Изменения данных — [mutations-errors-and-effects.md](mutations-errors-and-effects.md). Стек — [discovery-and-boundaries.md](discovery-and-boundaries.md).

Синтаксис ключей/tags бери у выбранного трека (RTK `providesTags`, TanStack `queryKey`, SWR key). Правила ниже — про смысл ключа и слои кэша, не про «правильную» библиотеку.

Infinite query, SSR loaders и codegen — не default: только при локальном прецеденте или явном запросе.

## 1. Два разных кэша

| Слой | Что это | Кто им владеет |
| --- | --- | --- |
| HTTP / browser cache | `Cache-Control`, ETag, диск браузера | Браузер и серверные заголовки. Клиентский query-слой его **не заменяет** |
| Server-state cache | Нормализованный ответ list/detail в памяти приложения | RTK Query / TanStack Query / SWR / аналог зоны |

Не дублируй server-state в глобальный store «чтобы было под рукой», если query-кэш уже есть ([`frontend-architecture`](../../frontend-architecture/SKILL.md) — владелец state). Не настраивай HTTP cache заголовками с клиента и не считай `staleTime` заменой `Cache-Control`.

Conditional requests (`If-None-Match` / ETag) подключай только если зона ими уже пользуется. Не добавляй ETag-логику «для правильности».

## 2. Keys и tags: все входы запроса

Ключ (или tag + аргументы) должен однозначно кодировать **все входы**, от которых зависит ответ:

- идентификатор ресурса и коллекция (`things` vs `things/${id}`);
- фильтры, сортировка, pagination (`page` / `cursor` / `limit`);
- locale / content variation, если payload от них меняется;
- session scope (текущий пользователь, tenant, active workspace), если ответ персональный.

Запрещены значения, которые реально меняются без изменения запроса: `Date.now()`, `new Date()`, random id, mutable/non-serializable class instance или функция, если трек не описывает её сериализацию. Новый plain object с теми же сериализуемыми полями допустим в современных TanStack Query, RTK Query и SWR: они строят стабильный cache key по значению, а не по identity. Для старой или другой библиотеки сверь установленную версию.

Неполный ключ (забытый фильтр или user id) → чужие данные на экране или промах инвалидации. Волатильное значение (`Date.now()`/random) → нет дедупа и постоянный refetch.

Tags (RTK) / query filters (TanStack) для list/detail должны стыковаться: create инвалидирует список; patch — detail и при необходимости list. Не вешай один глобальный tag на все ресурсы «на всякий случай», если зона уже режет по типам.

## 3. Stale, fresh, conditional query

Политику stale/fresh бери у соседей зоны (`keepUnusedDataFor`, `staleTime`, `revalidateOnFocus`), не абстрактный «лучший TTL».

- Условный query (`skip` / `enabled`) — пока нет обязательного id или сессии. Не стреляй `/things/${id}` с пустым id.
- Placeholder/cached data показывать можно, если трек это умеет; не подменяй это ручным `useState` копии ответа.
- После смены session scope (login / logout / switch user) **сбрось** кэш запросов, которые были про другого субъекта. Не оставляй list предыдущего пользователя до фонового refetch.

## 4. Dedupe, гонки, stale response

Одинаковый in-flight запрос не должен уходить дважды: штатный дедуп query-слоя переиспользуй. Голый `fetch` без слоя — не изобретай второй query-клиент «чтобы дедупить»; повтори локальный аналог зоны или спроси.

Смена ключа / размонтирование: используй штатную cancellation/stale-result семантику трека. Если query-layer передаёт `AbortSignal`, доведи его до транспорта. Некоторые слои намеренно не отменяют unused query, чтобы сохранить результат в cache; не ломай эту политику ручным `AbortController`. В любом случае устаревший ответ не должен перезаписать более новый state.

Если отмена невозможна (нет signal в старом аналоге), игнорируй устаревший response по generation/seq номеру запроса. Не копируй anti-pattern «последний `then` всегда побеждает» без проверки актуальности.

Abort ≠ HTTP error: [http-transport-and-contracts.md](http-transport-and-contracts.md) §5.

## 5. List, detail, pagination

| Операция | Кэш |
| --- | --- |
| Create | Инвалидировать (или обновить) list; не оставлять stale список |
| Update | Обновить detail; list — если элемент там виден или изменились поля списка |
| Delete | Убрать из list и detail; не ждать ручной reload страницы, если трек умеет invalidate |

Pagination — **тот контракт, что у endpoint зоны** (page+size, offset, cursor). Не подменяй page на infinite scroll без прецедента или явного запроса.

Infinite / `useInfiniteQuery` / `useSWRInfinite`:

- только если в зоне уже есть бесконечный список того же типа или пользователь явно просил;
- page param / cursor следующей страницы бери из ответа, не вычисляй «на глаз»;
- ключ включает все фильтры, как у конечного списка.

## 6. SSR и framework loaders

Нужна ли server-data граница и нет ли двойного fetch без контракта — [`frontend-ssr`](../../frontend-ssr/SKILL.md); здесь — HOW `dehydrate` / query keys / loader fetch, не решение render-mode.

Только если зона уже грузит данные в `loader`, RSC, server action или аналоге:

- не дублируй тот же GET и в loader, и в client query без принятой гидрации (`dehydrate` / initial data / framework cache);
- `AbortSignal` / timeout — как у соседнего loader;
- секреты и cookie-only fetch остаются на сервере, если так устроено; не копируй server-only credential в клиентский вызов.

Нет SSR-прецедента — не внедряй loader «для SEO» в чисто клиентскую зону. Codegen не подключай без запроса.

## 7. Чего не делать

- Класть в ключ нестабильные значения или забывать фильтр / session scope.
- Дублировать query-кэш в Redux/Zustand/`useState` без причины.
- Side-effecting GET/query (logout, send-email, clone) — это mutation: [mutations-errors-and-effects.md](mutations-errors-and-effects.md).
- Считать browser cache и query cache одним механизмом.
- Включать infinite/SSR/codegen по умолчанию.
- Показывать abort/stale reject как ошибку загрузки.
