# Decision, evidence and migration

Читай на шаге 3–4 перед статусом и перед любой записью. Без полей ниже статус **не подтверждай**. Рецепты роутера, `dehydrate` и form `defaultValues` сюда не копируй.

Файлы в `audit` и `decision` не меняй. Отчёт только в чат. Persistent SSR-отчёт, waiver и runtime-логи не создавай. Cookie/header values и PII — `[REDACTED]`.

## 1. `decision` ≠ `audit` ≠ `migrate`

| Режим | Scope | Выход | Запись |
| --- | --- | --- | --- |
| `audit` | Все страницы/экраны заявленного scope | Сетка статусов + evidence | Нет |
| `decision` | **Одна** страница/экран («нужен ли здесь SSR/RSC/island?») | Тот же статус и те же поля, что у одной строки `audit` | Нет |
| `migrate` | Только страницы, которые уже имеют `MIGRATE` | Точечная граница | Да, узко |
| `fix` | Confirmed hydration / server-client дефекты | Патч + повторная проверка того же scope | Да, узко |

`decision` не обходит соседей «заодно» и не заменяет `migrate`. `audit` не применяет патч. Path / открытый `page.tsx` не включает `migrate`.

Неясный запрос без «переведи/почини» остаётся `audit`.

## 2. Три условия `MIGRATE`

Ставь `MIGRATE`, только если верны **все три**:

1. В зоне уже есть SSR / RSC / loader / islands-трек (см. [discovery-and-render-mode.md](discovery-and-render-mode.md)).
2. Есть конкретная польза **или** подтверждённый дефект (лишний client bundle на статике, hydration mismatch, CSR-bailout без слота, server-only leak в client, двойной fetch без контракта).
3. Границу можно сузить безопасно: server parent + узкий client island, serializable props, без смены фреймворка и без переноса файлов «в новую схему».

Нет любого пункта — это не `MIGRATE`. Открытый `app/page.tsx` или «так принято в SSR-статье» пункта не заменяют.

## 3. Остальные статусы

| Статус | Когда | Что не делать |
| --- | --- | --- |
| `KEEP` | Чистый CSR по замыслу зоны; преимущественно browser-only UI (editor/chart/canvas); **уже корректный** server parent + client island; нет SSR-прецедента и install запрещён; страница уже `static`/`isr` и граница корректна | Не «улучшать» корректный island; не предлагать `ssr:false` на всю страницу; не ставить фреймворк. CSR-only / нет прецедента при вопросе про render-mode → в рекомендации явная строка «SSR не нужен / фреймворк не ставить». `KEEP` здесь значит «оставь текущий режим», не «оставь SSR» |
| `INCOMPLETE` | Смешанный стек без доказанного трека этого корня; нельзя подтвердить first render / границу / data-контракт; файлы недоступны | Не повышать гипотезу до `MIGRATE`; не выдумывать дефект |
| `NOT_APPLICABLE` | Нет страницы/UI; не render-surface; запрос про form values, XSS/auth cookies, route syntax, placement, визуал skeleton, системный a11y | Не читать остальные references по этому узлу; отдай соседу. Не ставь на CSR-only вместо `KEEP` |

Корректный hybrid (server parent отдаёт данные и статику, island — только браузерный виджет, props plain, нет повторного GET без контракта) — **`KEEP`**, даже если пользователь «хочет ещё больше SSR». Это не отказ от скила, это отказ трогать работающую границу. Уже статическая страница на доказанном треке — тоже `KEEP`: HTML на билде не требует runtime SSR.

## 4. Evidence (обязательные поля)

На каждую страницу в scope — все поля:

| Поле | Что писать |
| --- | --- |
| `file:line` | Page/layout/island или стык границы (оба, если на разных строках) |
| Трек | CSR-only / classic SSR / RSC / islands / hybrid — по репо, не по желанию |
| Когда HTML | `static` / `isr` / `request-dynamic` / `н/п` — подтип, не трек ([discovery-and-render-mode.md](discovery-and-render-mode.md) §1.1) |
| Польза или дефект | Конкретный эффект **или** «дефекта нет» для `KEEP` |
| Граница | Где server, где client, какие props пересекают стык |
| Confidence | `high` / `medium` / `low` |
| Блокеры | Чего не хватает для смены статуса |
| Evidence dependencies | Импорт/родитель за path, который читали |
| Hand-off | routing / api / architecture / security / forms / layout / a11y — или «нет» |

Confidence:

- **high** — трек, граница и польза/дефект видны в scope;
- **medium** — часть контракта в импорте за path, но импорт однозначен;
- **low** — только имя файла или «похоже на RSC»; в статус `MIGRATE` не поднимай.

Нет `file:line`, трека и границы — это неподтверждённый сигнал, не `MIGRATE`.

## 5. Формат итога (`audit` / `decision`, только чат)

```
Frontend SSR: сетка статусов (не один PASS/FAIL)
Режим: audit | decision
Scope: <path или diff или экран>
Трек зоны: <доказанный трек + 2–3 факта из репо>
Limitations: <недоступные файлы / смешанный корень; «нет», если ограничений нет>

Страницы:
- [MIGRATE|KEEP|INCOMPLETE|NOT_APPLICABLE] [high|medium|low confidence] <file:line>
  трек: <...>
  когда HTML: <static | isr | request-dynamic | н/п>
  польза/дефект: <...>
  граница: <server … / client …>
  рекомендация: <стратегия без патча; для CSR-only / нет прецедента — «SSR не нужен / фреймворк не ставить»>
  блокеры: <… или «нет»>
  hand-off: <сосед или «нет»>

Evidence dependencies:
- <path> — зачем читали

Неподтверждённые сигналы:
- <file:line> — чего не хватает для §4
```

В `decision` блок «Страницы» содержит **одну** строку. Соседние экраны не добавляй.

Нет дефекта — напиши это явно и ставь `KEEP`, не выдумывай работу. Патч в `audit`/`decision` не предлагай как готовый diff и не применяй.

## 6. Отказ массовой миграции

«Переведи всё на SSR», «мигрируй весь `app/`», «заодно соседние экраны», «переведи продукт на RSC», «поставь Next/Remix и переведи» — **отказ**.

- Не переходи в `migrate`.
- Не расширяй scope за заявленные страницы.
- Не ставь фреймворк и не меняй Pages ↔ App / CSR ↔ SSR-стек. «Поставь Next и переведи» — install + смена трека, не точечный `migrate`.
- Если рядом просили «проверь» — оставайся в `audit` и размечай статусы, включая `KEEP`.
- Если просили только массовый перенос — остановись: режим не `migrate`, файлы не трогай.

Массовый отказ — не `NOT_APPLICABLE` на каждую страницу: страницы по-прежнему можно разметить в `audit`. Отказ относится к **записи и расширению scope**.

## 7. Безопасная точечная миграция

Только после явного «переведи/примени/мигрируй» **и** статуса `MIGRATE` на этой странице (из текущего `audit`/`decision` или из явно подтверждённого вердикта в запросе).

Порядок:

1. Оставь server parent точкой данных и статичной разметки.
2. Вынеси в client только виджет с браузерным API / обработчиками.
3. Через границу — serializable props; server-only модуль в client не импортируй. См. [server-client-boundaries.md](server-client-boundaries.md).
4. Файлы должны сменить папку / public API — **стоп**, сначала [`frontend-architecture`](../../frontend-architecture/SKILL.md) на placement, затем граница здесь.
5. Второй fetch / dehydrate / query keys — [`frontend-api`](../../frontend-api/SKILL.md), не изобретай data-слой.
6. `'use client'`, `useRouter`, `Link`, `redirect`, `loading.tsx`, `generateMetadata` как синтаксис — [`frontend-routing`](../../frontend-routing/SKILL.md); не копируй рецепт оттуда, чтобы «включить» island.

Не мигрируй страницы со статусом `KEEP` / `INCOMPLETE` / `NOT_APPLICABLE`. Не вешай `'use client'` на весь page-модуль, если достаточно island. Не ставь `ssr:false` на страницу целиком. Как надо / как не надо — [examples-good-and-bad.md](examples-good-and-bad.md).

## 8. Hand-off вместо чужой работы

| Запрос или находка | Куда |
| --- | --- |
| Как писать `'use client'` / `useRouter` / `Link` / `redirect` / `loading` / metadata | [`frontend-routing`](../../frontend-routing/SKILL.md) |
| Как писать loader / RSC fetch / dehydrate / query keys | [`frontend-api`](../../frontend-api/SKILL.md) |
| Куда положить server-модуль vs island | [`frontend-architecture`](../../frontend-architecture/SKILL.md) |
| XSS / token в бандле / `cookies()` как auth | [`frontend-security`](../../frontend-security/SKILL.md) |
| Default values / edit-гидрация формы | [`frontend-forms`](../../frontend-forms/SKILL.md) |
| Skeleton / fallback как визуал | [`frontend-layout`](../../frontend-layout/SKILL.md) |
| Системный a11y/WCAG | [`frontend-a11y`](../../frontend-a11y/SKILL.md) |

Этот файл решает **нужна ли** миграция и **можно ли** сузить границу, не HOW соседней зоны.
