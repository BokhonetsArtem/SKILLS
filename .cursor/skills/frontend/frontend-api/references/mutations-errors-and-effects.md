# Mutations, ошибки и эффекты

Читай, когда нужны create/update/delete, retry, invalidation, optimistic update, нормализация ошибок или side effects после запроса. Query cache — [queries-cache-and-concurrency.md](queries-cache-and-concurrency.md). Auth refresh — [auth-files-and-edge-cases.md](auth-files-and-edge-cases.md). Транспорт — [http-transport-and-contracts.md](http-transport-and-contracts.md).

Optimistic update — не default: только при локальном прецеденте или явном запросе.

## 1. Query читает, mutation меняет

| Намерение | Как оформлять |
| --- | --- |
| Чтение без побочных эффектов на сервере | query / GET (безопасный метод, [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) §9.2.1) |
| Создание, изменение, удаление, send-email, logout, clone, toggle | mutation в server-state слое; HTTP-метод — строго по фактическому контракту API |

Не прячь изменение за `build.query` / `useQuery` / `useSWR`, даже если legacy API использует GET. Query-слой может дедупить, кэшировать и перезапускать вызов — для logout или «отправить письмо» это опасно. Оформи вызов как mutation, но не меняй GET на POST без изменения серверного контракта; отдельно предложи исправить unsafe API.

Оформление внутри трека (`build.mutation`, `useMutation`, `trigger`) бери у соседа той же зоны. Имена методов не выдумывай: следуй API.

## 2. Retry и идемпотентность

Автоповтор — только когда операция **безопасна** или **идемпотентна**, либо контракт даёт idempotency key.

| Метод / контракт | Авто-retry при сети/5xx |
| --- | --- |
| GET / HEAD / безопасное чтение | Да, в разумных пределах трека |
| PUT / DELETE (идемпотентны по RFC 9110 §9.2.2) | Обычно да, если повтор не ломает контракт зоны |
| POST без заголовка идемпотентности | **Нет** — повтор может создать второй ресурс / списать дважды |
| PATCH | Не считай идемпотентным по умолчанию |
| POST при уже принятом заголовке идемпотентности зоны | Можно; значение стабильно на одну пользовательскую попытку |

Timeout/сеть на неидемпотентном POST: не крути retry в interceptor «для надёжности». Дай пользователю явную повторную отправку (форма / кнопка), с защитой от double-submit на стороне [`frontend-forms`](../../frontend-forms/SKILL.md).

Abort не ретраи. 4xx (кроме выбранных политикой зоны, например 429) не ретраи как «ещё раз тот же POST».

## 3. Invalidation после успеха

Инвалидация и update кэша живут **у мутации**, не в shared-кнопке и не в UI-примитиве.

- Дождись подтверждённого успеха транспорта, затем invalidate/update. Не чисти кэш в `onClick` до запроса.
- List/detail — [queries-cache-and-concurrency.md](queries-cache-and-concurrency.md) §5.
- Не инвалидируй «все теги мира», если зона уже режет по ресурсу.

Ошибка мутации не должна оставлять кэш в полузаписанном виде: либо не трогали кэш до успеха, либо rollback (§4).

## 4. Optimistic update

Только если в зоне уже есть образец того же типа или пользователь явно просил.

Обязательный каркас:

1. Снимок затронутого кэша (patch result, previous data).
2. Оптимистичная запись.
3. Дождаться результата.
4. Успех — сверка с серверным телом (reconciliation), не молчаливое «уже показали».
5. Ошибка или abort — **rollback** к снимку.

Как не надо: писать в store/кэш до запроса без undo; оставлять оптимистичные данные после 409/422; считать toast заменой rollback.

Не делай optimistic на неидемпотентном create, если повтор/ошибка порождает дубликат и в зоне этого нет.

## 5. Ошибки: нормализация, не сырой payload

HTTP-слой приводит отказ к узкому контракту зоны. UI и форма не парсят сырой body каждый раз заново. Если контракт уже принят — сохрани его. Если проектирует новый — предпочти discriminated shape (`kind: 'http' | 'network' | 'timeout' | 'parse'`, безопасный `message`, `status` только для HTTP, опционально `code`, `retryAfter`, `fieldErrors`) вместо магического `status: 0`.

| Источник | Нормализация |
| --- | --- |
| Фактический JSON зоны | Как у соседних мутаций: `message`, map полей, массив `{ field, message }` |
| RFC 9457, **если** `Content-Type` problem+json | `title`/`detail`/`status`; extensions — только известные поля. Не включать problem+json «на вырост» |
| Пустой / HTML / не-JSON | Общая транспортная ошибка, не текст HTML в toast |
| Abort / timeout / сеть | Отдельные виды; abort — не ошибка формы ([http-transport-and-contracts.md](http-transport-and-contracts.md) §5) |

В `message` для UI — безопасная формулировка. Не клади в toast/status сырой 500, stack, токен, email, внутренний URL, полный объект ответа. PII и токены не логируй.

**Field-level:** API-слой **нормализует** список ошибок полей. Применяет к форме (`setError` / `setFieldError`) — [`frontend-forms`](../../frontend-forms/SKILL.md). Не бинди form state здесь. Неизвестное поле — общая ошибка, не молчаливый toast-only, если бэкенд указал поле.

401 refresh — один in-flight refresh и очередь исходных запросов, без цикла: [auth-files-and-edge-cases.md](auth-files-and-edge-cases.md). Не путай 403 (нет прав) с 401 (нет/протухла сессия), если зона их различает.

## 6. Side effects только после подтверждения

Toast успеха, навигация, analytics, запись в storage, закрытие модалки — **после** fulfilled результата, не в начале мутации и не в `onClick` до `queryFulfilled` / `onSuccess`.

Эффект принадлежит сценарию, который вызвал мутацию (feature / page-handler), если архитектура зоны так делает. Не прячь навигацию внутрь shared-кнопки. Не дублируй один toast и в API-обвязке, и в UI без образца.

Ошибка: общий отказ — статус/toast сценария; поле — forms. Не показывай success-эффект на rejected promise.

UI loading/error/empty не верстай здесь — [`frontend-layout`](../../frontend-layout/SKILL.md).

## 7. Чего не делать

- Side-effecting query / безопасный GET с скрытым POST-смыслом.
- Безусловный retry POST.
- Toast и redirect до fulfilled.
- Optimistic без snapshot/rollback.
- Показ сырого payload, PII и токена.
- Применять field errors к форме из этого скила (только нормализация).
- Запускать security-аудит из обычной обработки 401.
