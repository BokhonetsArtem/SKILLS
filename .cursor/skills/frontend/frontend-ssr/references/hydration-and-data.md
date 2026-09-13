# Hydration and data

Читай на шаге 4 в `audit`/`fix` и когда в запросе есть «гидрация», mismatch или двойной fetch. Слово «hydration» **в формах** (default values, edit `reset`) — [`frontend-forms`](../../frontend-forms/SKILL.md) + [lifecycle-and-submit.md](../../frontend-forms/references/lifecycle-and-submit.md) §1; этот файл тогда не применяй.

HOW loader / RSC fetch / `dehydrate` / query keys — [`frontend-api`](../../frontend-api/SKILL.md) + [queries-cache-and-concurrency.md](../../frontend-api/references/queries-cache-and-concurrency.md) §6. Здесь — **нужен ли** один источник на first render, не как писать кэш.

Skeleton / fallback как визуал — [`frontend-layout`](../../frontend-layout/SKILL.md). `Suspense` / streaming как **render-контракт** — этот файл.

Runtime-логи, дамп first-render HTML и PII в чат не выводи. Debug-`console` в примеры и прод-код не добавляй.

## 1. Два разных слова «hydration»

| Контекст | Смысл | Кто ведёт |
| --- | --- | --- |
| SSR/RSC | Сверка server HTML с первым клиентским деревом | этот скил |
| Форма | Заполнить `defaultValues` / `reset` из сущности | [`frontend-forms`](../../frontend-forms/SKILL.md) |

Запрос «почини hydration формы редактирования / default values» без server/client mismatch → `NOT_APPLICABLE`, hand-off в forms. Не лечи `reset(mappedValues)` сменой render-mode.

Запрос «hydration mismatch» на дереве, где в разметке `Date.now()` / `window` — этот файл, даже если дерево внутри `<form>`.

## 2. Deterministic first render

Первый проход server и первый проход client на **общем** дереве должны дать одно и то же.

| Запрещено на общем пути | Как закрыть |
| --- | --- |
| `Date.now()`, `new Date()` без snapshot, `Math.random()`, `crypto.randomUUID()` на каждый render | Считать на сервере, передать serializable snapshot пропом |
| `window`, `document`, `localStorage`, `innerWidth` вне island / эффекта | Унести в client island; на сервере — fallback, который совпадёт с первым клиентским кадром, или не рендерить расходящийся кусок до mount **только в island** |
| Locale/timezone из среды клиента на server path | Locale как render-input с сервера (plain string), не `Intl` от браузера в общем дереве |
| Ветка `typeof window !== "undefined" ? A : B`, если A и B — разная разметка first paint | Одна разметка до гидрации; расхождение — после mount в island |

Расхождение текста/атрибута/числа детей — confirmed hydration mismatch: в `audit` фиксируй, в `fix` сделай first render детерминированным. Не глуши warning обёрткой `suppressHydrationWarning` без принятого прецедента зоны на этом узле (например time-ago, который зона уже так помечает). Нет прецедента — не добавляй suppress как fix.

Пары — [examples-good-and-bad.md](examples-good-and-bad.md) §3.

## 3. Один источник данных

Если страница уже получает данные на сервере (RSC `async`, `loader`, `getServerSideProps`, server parent), клиент **не** повторяет тот же GET, пока в зоне нет принятого hydration-контракта (`dehydrate` / `initialData` / framework cache).

| Доказано в зоне | Этот скил | Не этот скил |
| --- | --- | --- |
| Server fetch есть, client `useQuery`/`useEffect(fetch)` на тот же ресурс без initial data | Дефект двойного fetch: `MIGRATE`/`fix` | Писать query key / `dehydrate` — api |
| Контракт гидрации кэша уже принят у соседей | Не ломай его вторым клиентским GET «на всякий случай» | Копировать рецепт dehydrate сюда |
| CSR-only, серверных данных нет | Не внедряй loader «для SEO» | — |

Секреты и cookie-only fetch остаются на сервере. Не протаскивай credential в client, чтобы «убрать» второй запрос — [`frontend-security`](../../frontend-security/SKILL.md).

Пары — [examples-good-and-bad.md](examples-good-and-bad.md) §4.

## 4. Suspense и streaming

`Suspense` / streaming boundary — часть render-контракта: что сервер может отложить и чем заменить слот на время потока.

| Здесь | Не здесь |
| --- | --- |
| Где обрывается server tree, какой fallback **слот** обязателен, чтобы не уйти в CSR-bailout на доказанном треке | Цвет/отступы skeleton, «красивый» placeholder — layout |
| Не дублировать второй boundary, если родитель уже обернул тот же слот | Анимация и токены fallback |

На треке, где клиентский hook (search/params или эквивалент зоны) требует boundary:

- **факт** «нужен слот» — этот скил;
- в `fix` оберни расходящийся island в `<Suspense>` (или слот трека), если родителя с тем же слотом ещё нет;
- **как** писать `loading.tsx` / router API / сам hook — [`frontend-routing`](../../frontend-routing/SKILL.md). Рецепт файла и hook не копируй.

В `audit` без «почини» дефект только зафиксируй. Не отдавай весь баг соседу: слот — render-контракт, его ставит этот скил.

Не ставь streaming, которого нет в треке зоны. Не заменяй детерминированный first render бесконечным fallback.

## 5. Browser-only виджет

Editor / chart / map / canvas, которым нужен WebGL или `window` сразу — осознанный client island, часто с `ssr:false` **на модуле виджета**. Страница вокруг остаётся на треке зоны.

`ssr:false` на всю страницу, чтобы «не было mismatch» — антипаттерн, не fix. См. [examples-good-and-bad.md](examples-good-and-bad.md) §5.

Такой экран при корректном island — `KEEP`, не `MIGRATE` на полный SSR виджета.

## 6. Что чинить в `fix`

Только confirmed дефекты в заявленном scope:

1. Недетерминированный общий путь (§2).
2. Server-only import в client ([server-client-boundaries.md](server-client-boundaries.md) §3).
3. Двойной fetch без контракта (§3).
4. Лишний page-level `ssr:false`, если трек позволяет оставить parent на сервере.
5. CSR-bailout / mismatch из-за island без слота (§4): поставь `<Suspense>` (или слот трека) вокруг island. Не создавай и не правь `loading.tsx`.

После правок повторно проверь **тот же** scope. Form `defaultValues` не сбрасывай «заодно». Не рефакторь соседей. Не ставь SSR-фреймворк.

Итог `fix` в чат: что подтвердили (`file:line`), что изменили, результат повторной проверки, оставшиеся `INCOMPLETE`. Persistent отчёт не создавай.
