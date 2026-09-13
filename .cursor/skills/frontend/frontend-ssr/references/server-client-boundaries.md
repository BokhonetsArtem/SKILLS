# Server-client boundaries

Читай на шаге 4, когда решаешь, где обрывается server parent и что можно отдать в client island. Синтаксис `'use client'` / `useRouter` / `Link` / `redirect` — [`frontend-routing`](../../frontend-routing/SKILL.md) + [next-routing-rules.md](../../frontend-routing/references/next-routing-rules.md) §2; сюда рецепты не копируй.

Placement server-модуля vs island / public API — [`frontend-architecture`](../../frontend-architecture/SKILL.md). Если файлы должны переехать, сначала placement, затем этот файл.

Cookie flags, CSRF, `cookies()` как проверка сессии — [`frontend-security`](../../frontend-security/SKILL.md) + [`aif-security-checklist`](../../../aif-security-checklist/SKILL.md). Здесь cookies/headers — только **render-input**.

Значения cookie/header и токены в чат не выводи: `[REDACTED]`. Runtime-логи границы не добавляй.

## 1. Минимальный client island

Server parent остаётся владельцем данных и статичной разметки. В client уходит только то, чему нужен браузерный API, локальный state или обработчики.

| На сервере (parent) | В island | Не island |
| --- | --- | --- |
| Чтение данных, списки, текст, метаданные страницы как render-результат | Редактор, chart, map, жесты, localStorage, browser-only виджет | Вся страница «на всякий случай» |
| Сборка plain props | Интерактив фильтра/сортировки, которой нужен state | Статическая статья + `'use client'` на корне |
| Выбор ветки разметки по serializable render-input | Подписка на `resize` / `matchMedia` / geolocation | Повторный GET тех же данных без контракта |

Уже корректный parent + узкий island — `KEEP`, не расширяй и не сужай ради вкуса. См. [decision-and-migration.md](decision-and-migration.md) §3.

Не делай:

- `'use client'` (или эквивалент зоны) на page/layout, если браузер нужен одному виджету;
- `ssr:false` / `ClientOnly` / `dynamic(..., { ssr: false })` на **весь** экран;
- второй client tree рядом с уже гидратированным island «для удобства».

Имена директив бери у доказанного трека (`'use client'`, `client:load`, `onMount` + split), не навязывай Next-синтаксис CSR-пакету.

## 2. Serializable props

Через границу server → client проходят только значения, которые переживают сериализацию трека зоны (обычно JSON-подобные).

| Можно передать | Нельзя передать |
| --- | --- |
| `string`, `number`, `boolean`, `null` | `Date`, `Map`, `Set`, `bigint` без принятого прецедента зоны |
| Plain object / array из таких полей | class instance, ORM-модель, function, symbol |
| ISO-строка даты, id, enum-литерал | React element / server component как hidden prop «чтобы открыть на клиенте» |
| Уже сериализованный snapshot (`{ now: "2026-09-13T00:00:00.000Z" }`) | Живой `Request`, cookie store, header bag |

Непроходимый проп — дефект границы (`fix` / аргумент для `MIGRATE`), не повод объявить всю страницу client. Преобразуй на сервере в plain data.

Пары — [examples-good-and-bad.md](examples-good-and-bad.md) §2.

## 3. Server-only imports

Модуль, который читает секреты, ORM, filesystem, `cookies()`/`headers()` как серверный API, или помечен server-only, **не** импортируй из client island.

| На сервере | Через границу | В client |
| --- | --- | --- |
| `db.list()`, `fs`, secret env, server SDK | `items: PlainItem[]` | Рендер списка, обработчики |
| `cookies()` / `headers()` как чтение render-input | `locale: "en"` | Форматирование уже выбранной локали |
| Server action / mutation entry зоны | идентификатор действия, который зона уже отдаёт клиенту | Вызов по контракту зоны, не импорт server-модуля |

Импорт server-only в client — confirmed server-only leak: в `audit` фиксируй, в `fix` разорви import (данные — проп, не модуль). Не чини это переносом секрета в `NEXT_PUBLIC_*` / `VITE_*` — это [`frontend-security`](../../frontend-security/SKILL.md).

## 4. Cookies и headers как render-input

Допустимый render-input: значение, которое **уже** читает server parent, чтобы выбрать первый HTML (локаль, theme hint, A/B-метка, если зона так делает), и передаёт вниз **plain**-полем.

| Этот скил | Не этот скил |
| --- | --- |
| `locale` / `theme` из cookie → проп island | Сессия, роль, «пустить / не пустить» |
| Заголовок `Accept-Language` как вход сортировки строк на сервере | Cookie flags, `Secure`/`HttpOnly`, CSRF |
| Стабильный snapshot, чтобы first render совпал | `cookies()` в client module |

Не цитируй значение cookie/header. В evidence пиши имя ключа и `file:line`. Вызов `cookies()` / `headers()` на server page — также факт подтипа `request-dynamic` ([discovery-and-render-mode.md](discovery-and-render-mode.md) §1.1): не снимай его, чтобы «сделать страницу static». Auth-решение по cookie — hand-off в security / чеклист, даже если оно стоит рядом с SSR page.

`redirect` после чтения сессии — синтаксис и место вызова: [`frontend-routing`](../../frontend-routing/SKILL.md), не этот файл.

## 5. Когда границу трогать нельзя

Стоп и не «упрощай» до одного client-файла:

- island уже минимален, props plain, server-only не протекает — `KEEP`;
- нужен перенос файлов / новый public API — architecture first;
- трека SSR/RSC/islands нет — не создавай границу установкой фреймворка;
- запрос — массовый «переведи всё» — отказ, [decision-and-migration.md](decision-and-migration.md) §6.

First render, mismatch и второй fetch — [hydration-and-data.md](hydration-and-data.md).
