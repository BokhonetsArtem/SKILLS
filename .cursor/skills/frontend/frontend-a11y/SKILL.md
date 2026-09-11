---
name: frontend-a11y
description: >-
  Аудирует и исправляет доступность клиентского UI по фактическому стеку
  зоны — landmarks, семантика, accessible names, keyboard, focus, ARIA,
  SPA-анонсы. Use when accessibility, a11y, WCAG, ARIA, keyboard, focus,
  screen reader, «проверь доступность», «аудит доступности», «сделай
  доступным». Не для form lifecycle, validation и submit — это
  frontend-forms. Не для вёрстки, CSS и визуальных токенов — это
  frontend-layout. Не для route tree, navigation и URL-state — это
  frontend-routing. Не для XSS / unsafe HTML / preview — это
  frontend-security. Не для размещения слайса — это
  frontend-architecture. Не ставит axe, pa11y, Lighthouse или второй набор
  UI-примитивов.
disable-model-invocation: false
argument-hint: "[аудит | исправить | scope]"
---

# Frontend a11y

Аудируй или исправляй **доступность клиентского UI** по **фактическому стеку зоны**. Линза — **WCAG 2.2 AA**, не энциклопедия критериев: номера SC и полный каталог в этот файл не копируй. Сначала классифицируй режим (`audit` / `fix` / `NOT_APPLICABLE`), затем определи framework, UI-kit и уже установленные lint/a11y-инструменты. Не ставь `axe`, `pa11y`, Lighthouse, новый ESLint-плагин или второй набор UI-примитивов.

Инструменты зависят от режима (таблица шага 1). В `audit` — только чтение, как [`frontend-security`](../frontend-security/SKILL.md): файлы не меняй, `--fix` не запускай. В `fix` — точечные правки в заявленном scope, как forms/layout: массовый рефакторинг не делай. Runtime-логи, значения полей, DOM-текст с PII и токены в чат не выводи: редактируй как `[REDACTED]`.

Правила категорий, evidence и примеры **не копируй в этот файл**. Читай только нужный файл из `references/` (пути относительно этой папки скила).

**Не этот скил.** Form lifecycle, validation, submit, field binding — [`frontend-forms`](../frontend-forms/SKILL.md); локальный form a11y при реализации формы остаётся там, системный a11y/WCAG-аудит — здесь. Вёрстка, CSS, визуальные токены, Flex/Grid, макет Figma — [`frontend-layout`](../frontend-layout/SKILL.md); hit-area / видимый focus / DOM-порядок при вёрстке остаются там. Route tree, navigation, redirect, URL-state — [`frontend-routing`](../frontend-routing/SKILL.md); scroll / focus / title после client navigation при работе над маршрутом остаются там. XSS, unsanitized HTML, unsafe preview — [`frontend-security`](../frontend-security/SKILL.md), не «чинить» через `innerHTML`. Куда класть слайс / public API — [`frontend-architecture`](../frontend-architecture/SKILL.md). Нет клиентского UI — `references/` не читай.

Маршруты (формулировка запроса → скил):

- «проверь a11y / доступность / WCAG» / «аудит» без «исправь» → этот скил, `audit`
- «почини / сделай доступным / проверь и исправь» → этот скил, `fix`
- «label / htmlFor / submit / form state» без системного аудита → [`frontend-forms`](../frontend-forms/SKILL.md)
- «сетка / токены / hit-area при вёрстке» → [`frontend-layout`](../frontend-layout/SKILL.md)
- «route tree / URL-state / scroll-focus-title при работе над маршрутом» → [`frontend-routing`](../frontend-routing/SKILL.md)
- «XSS / dangerouslySetInnerHTML / unsafe HTML / preview» → [`frontend-security`](../frontend-security/SKILL.md)
- «куда положить слайс / public API» → [`frontend-architecture`](../frontend-architecture/SKILL.md)

Пересечение с forms / layout / routing **не** нулевое: при реализации фичи сосед ведёт свой локальный чеклист; при системном a11y/WCAG-запросе этот скил забирает аудит и точечный fix. XSS всегда остаётся у security.

**Матрица владения:**

| Тема | Кто ведёт при реализации | Кто ведёт при системном a11y-запросе |
| --- | --- | --- |
| label / `htmlFor` / `aria-invalid` / focus первой ошибки | [`frontend-forms`](../frontend-forms/SKILL.md) + [fields-and-accessibility.md](../frontend-forms/references/fields-and-accessibility.md) | этот скил |
| DOM-порядок, семантика, hit-area, видимый focus, reduced motion | [`frontend-layout`](../frontend-layout/SKILL.md) + [pravila.md](../frontend-layout/references/pravila.md) §6 | этот скил |
| scroll / focus / title после client navigation | [`frontend-routing`](../frontend-routing/SKILL.md) + [navigation-state-and-boundaries.md](../frontend-routing/references/navigation-state-and-boundaries.md) §7 | этот скил |
| XSS / unsafe HTML / preview | [`frontend-security`](../frontend-security/SKILL.md) | [`frontend-security`](../frontend-security/SKILL.md) (a11y только hand-off) |

**Режим** (не грузи лишнее):

| Задача | Что читать |
| --- | --- |
| Scope, framework, UI-kit, i18n, существующие линтеры/примитивы, исключения | [references/discovery-and-scope.md](references/discovery-and-scope.md) |
| Landmarks, headings, native semantics, accessible names, изображения, таблицы | [references/semantics-and-names.md](references/semantics-and-names.md) |
| Tab/Enter/Space/Escape, focus trap/restore, roving tabindex, live regions, loading/status | [references/keyboard-focus-and-dynamics.md](references/keyboard-focus-and-dynamics.md) |
| Границы с forms/routing/layout; route announcer, контраст, reduced motion, медиа | [references/forms-navigation-and-perception.md](references/forms-navigation-and-perception.md) |
| Evidence, lint-как-гипотеза, `INCOMPLETE`, severity, формат отчёта, порядок fix | [references/audit-and-fix.md](references/audit-and-fix.md) |
| Пары Плохо / Хорошо (нейтральный Web Platform) | [references/examples-good-and-bad.md](references/examples-good-and-bad.md) |

Form-поля, визуальные токены и route-transition **не копируй** из соседних references: для них — дельта аудита и отсылка к forms / layout / routing.

## Шаг 1. Классифицируй запрос и scope

Опора: сообщение пользователя, `$ARGUMENTS` (если скил вызван с аргументом), открытые клиентские файлы. В обычном чате `$ARGUMENTS` может быть пустым — это не стоп. Сначала классифицируй **режим** по намерению (сообщение и `$ARGUMENTS`), затем path как scope. Path (`src/Foo.tsx`, каталог, `changed`) сам по себе не включает `fix`. Неясный запрос без fix-сигнала — `audit`.

| Сигнал | Режим | Инструменты |
| --- | --- | --- |
| «проверь» / «аудит» / «WCAG» без «исправь»; `$ARGUMENTS` = `аудит` или path без fix-сигнала | `audit` | только чтение, как [`frontend-security/SKILL.md`](../frontend-security/SKILL.md); без записи файлов и без `eslint --fix` |
| «почини» / «сделай доступным» / «проверь и исправь»; `$ARGUMENTS` = `исправить` или path вместе с fix-сигналом | `fix` | точечные правки в scope, как forms/layout; без массового рефакторинга |
| нет клиентского scope / не UI | `NOT_APPLICABLE` | `references/` не читать |

Явный path, открытый компонент/экран, каталог или `changed` / `diff` / PR — заявленный scope. Без понятного клиентского scope **спроси** компонент, экран, каталог или diff; не сканируй монорепозиторий наугад и не подставляй «всё дерево» по умолчанию.

Пустой scope, только backend/infra или нет UI — верни `NOT_APPLICABLE`, `references/` дальше не читай.

Явный path ограничивает **проверяемые узлы**, но не обрывает доказательство: в `audit` разрешён read-only просмотр импорта/примитива за пределы path ровно настолько, чтобы подтвердить роль, имя или keyboard path. Эти файлы укажи как evidence dependencies и не считай расширением scope. В `fix` правь только заявленный scope.

## Шаг 2. Обнаружь стек и tooling

Смотри репозиторий, не привычку модели. Не навязывай React, TypeScript или a11y-scanner, которого нет в lockfile.

Что проверить для **discovery** (достаточно сигналов, не полный аудит на этом этапе):

- манифест и конфиги: `package.json` / workspace, Vite/Webpack/Next/Nuxt/Angular/SvelteKit/Vue;
- корни UI: `src/`, `app/`, `apps/*`, `packages/*` с клиентской точкой входа;
- UI-kit и примитивы зоны (`Button`, `FormField`, `Dialog` и аналоги) — переиспользуй, параллельный a11y-кит не заводи;
- form/router stack зоны — чтобы не дублировать forms/routing и знать, куда hand-off;
- i18n: если уже есть — новый accessible name через него; нет — не внедряй;
- исключения: generated, vendor, `node_modules`, тесты, фикстуры, docs, markdown-примеры;
- уже установленный tooling: `eslint-plugin-jsx-a11y`, vue-a11y, axe (или аналог) **в lockfile**. Только в `package.json` без lockfile — не запускай.

Уже установленный **клиентский** a11y/lint-инструмент (есть в lockfile) — одноразовый read-only прогон по scope, не watch. Запускай executable/официальную check-команду без `--fix`, `--write`, `--cache` и package scripts с неизвестными побочными эффектами. Нельзя уверенно гарантировать отсутствие записи — пропусти инструмент и отметь ограничение. Нет такого инструмента — не добавляй зависимость, иди вручную по категориям шага 3.

Не делай: установка пакетов, правка lockfile, второй UI-кит, запуск Lighthouse/pa11y/axe, которых нет в репо.

Детали корней, монорепо и контракта инструмента — [discovery-and-scope.md](references/discovery-and-scope.md).

## Шаг 3. Выбери references и категории

По типу задачи читай только нужное (progressive disclosure — не грузи все references сразу). Категории проверки — линза WCAG 2.2 AA, не чеклист всех SC:

| Категория | Reference |
| --- | --- |
| landmarks и headings, native semantics, accessible names, изображения, таблицы/списки | [semantics-and-names.md](references/semantics-and-names.md) |
| keyboard, focus, ARIA виджетов, dynamic announcements | [keyboard-focus-and-dynamics.md](references/keyboard-focus-and-dynamics.md) |
| forms (дельта аудита), SPA navigation (дельта аудита), media, contrast/motion | [forms-navigation-and-perception.md](references/forms-navigation-and-perception.md) |
| evidence, статус, порядок fix, повторная проверка | [audit-and-fix.md](references/audit-and-fix.md) |
| как надо / как не надо | [examples-good-and-bad.md](references/examples-good-and-bad.md) |

Узкий запрос (только имя кнопки, только модалка, только live region) — читай один тематический файл, не все. Form-рецепты, токены и route-transition API **не копируй**: отсылай к [fields-and-accessibility.md](../frontend-forms/references/fields-and-accessibility.md), [pravila.md](../frontend-layout/references/pravila.md) §6, [navigation-state-and-boundaries.md](../frontend-routing/references/navigation-state-and-boundaries.md) §7.

Паттерн бери у **ближайшего UI той же зоны**. Имена и импорты — как **здесь**. Чужие product path и обязательные библиотеки из examples скила в репозиторий не переноси.

## Шаг 4. Аудит или исправление

Перед статусом читай [audit-and-fix.md](references/audit-and-fix.md).

**Доказательства.** Lint warning и инструментальный хит — **гипотеза**, не confirmed finding без контекста в коде. Подтверждённая находка — только с ролью, источником accessible name, клавиатурным путём, impact для AT, `file:line` и confidence. Поля security (`source` / `sink` / exploit) **не копируй**.

Неподтверждённый сигнал — не выдуманный `FAIL`. Непроверяемое восприятие — в `Limitations` / секцию `INCOMPLETE`, не вместо уже доказанного `FAIL`/`WARN`:

- контраст «на глаз» из CSS без измеримого токена/инструмента зоны;
- реальный screen reader;
- восприятие motion без `prefers-reduced-motion` / токена / инструмента зоны;
- ошибка инструмента, недоступные файлы, оборванный обязательный keyboard path.

**XSS-handoff.** `dangerouslySetInnerHTML`, unsanitized HTML в live region или в accessible name — отдай [`frontend-security`](../frontend-security/SKILL.md). Не «чини» доступность через `innerHTML` и не предлагай вставить сырой HTML в имя или live region.

**PII.** Значения полей, DOM-текст с PII и токены в отчёт не копируй. Указывай тип/идентификатор и `file:line`, значение — `[REDACTED]`.

### Режим `audit`

Файлы, конфиги агента и зависимости не меняй. Автоисправление не предлагай патчем и не применяй. Итог — только в чат, persistent a11y-отчёт не создавай.

Формат итога:

- статус: `PASS` | `WARN` | `FAIL` | `INCOMPLETE` | `NOT_APPLICABLE`;
- scope, framework, UI-kit, какой инструмент гоняли (или почему вручную);
- coverage: проверенные категории и ограничения;
- находки: роль, источник имени, клавиатурный путь, AT impact, `file:line`, confidence;
- неподтверждённые сигналы и `INCOMPLETE`;
- безопасная рекомендация без патча; XSS — hand-off в security.

`PASS` означает только «в проверенном scope нет confirmed findings», не «весь UI доступен». `FAIL` — confirmed high или несколько medium, которые закрывают сценарий. `WARN` — confirmed только medium/low. Непроверяемое восприятие и ошибка инструмента — секция `Limitations` / `INCOMPLETE`, не замена `FAIL`/`WARN`. Итоговый `INCOMPLETE` — только если обязательную проверку нельзя закрыть и confirmed findings нет. Пустой/не-UI scope → `NOT_APPLICABLE`.

### Режим `fix`

Найди подтверждённые проблемы в заявленном scope, исправь минимально, затем **повторно проверь этот же scope**.

- Переиспользуй примитивы зоны (`Button`, `FormField`, `Dialog` и аналоги). Параллельный a11y-кит не заводи.
- Native-first: не ставь `role="button"` на `div` вместо `<button>`; ARIA — только если нативного элемента недостаточно.
- Новый видимый или accessible-текст — через i18n, если i18n уже есть; нет — не внедряй.
- Безопасный порядок: семантика → имя → клавиатура/focus → ARIA → анонсы. Детали — [audit-and-fix.md](references/audit-and-fix.md).
- Не выходи за scope, не рефакторь соседние экраны «заодно», не ставь инструменты.
- XSS-находку не закрывай разметкой: hand-off в [`frontend-security`](../frontend-security/SKILL.md).

Как надо / как не надо — [examples-good-and-bad.md](references/examples-good-and-bad.md), не рецепты конкретного проекта.

## Шаг 5. Чеклист перед сдачей

Если пункт не выполнен — не сдавай. Пункты про конкретный инструмент — только если он **есть** в репозитории.

**Общее**

- Режим выбран по таблице шага 1; tool policy режима не нарушен.
- Scope явный (компонент / экран / каталог / diff); монорепо наугад не сканировали.
- Стек, UI-kit и tooling взяты из репо; новые пакеты не установлены.
- WCAG 2.2 AA использован как линза, не скопирован каталогом SC.
- XSS / unsanitized HTML отданы в [`frontend-security`](../frontend-security/SKILL.md).
- PII, значения полей и токены — `[REDACTED]`. Runtime-логи не добавлялись.

**Только `audit`**

- Файлы не менялись; `eslint --fix` / `--write` не запускались.
- Lint/инструментальный хит не повышен до finding без контекста в коде.
- Непроверяемый контраст / screen reader / motion — секция ограничений, не выдуманный `FAIL` и не перебивает `FAIL`/`WARN`.
- Отчёт только в чат: статус, coverage, находки с ролью / именем / keyboard path / AT impact / `file:line` / confidence.
- Persistent a11y-отчёт, waiver и evaluation-scenarios не создавались.

**Только `fix`**

- Правились только confirmed findings в заявленном scope.
- Примитивы зоны переиспользованы; `role="button"` не заменил `<button>`.
- Accessible name при наличии i18n прошёл через него.
- Scope повторно проверен после правок; массового рефакторинга не было.
- Запущены уже существующие lint / typecheck по затронутой области (без `--fix`, если цель — только проверка).

Перед сдачей — коротко, не эссе:

- **Режим:** `audit` / `fix` / `NOT_APPLICABLE`.
- **Доказательства стека:** 2–3 факта из репо (framework, UI-kit, lint/a11y-инструмент или его отсутствие).
- **Scope:** путь или diff.
- **XSS / PII:** был ли hand-off; что редактировали как `[REDACTED]`.

## Запрещено

- ARIA-overuse и `role="button"` / `role="link"` вместо `<button>` / `<a>`;
- массовые правки за пределами scope и параллельный a11y-кит;
- установка `axe`, `pa11y`, Lighthouse, eslint-плагина или UI-примитивов, которых нет в lockfile;
- запись файлов, `--fix` / `--write` в режиме `audit`;
- persistent a11y-отчёты, waiver, evaluation-scenarios;
- runtime-логи, debug-вывод DOM, значений полей или PII;
- копирование всего WCAG / всех SC в `SKILL.md` или в чат;
- «чинить» live region / accessible name через unsanitized HTML;
- копировать security-поля `source` / `sink` / exploit и готовые PoC.

## Если что-то пошло не так

- Запрос только про form lifecycle / validation / submit без системного a11y → [`frontend-forms`](../frontend-forms/SKILL.md), этот скил дальше не веди.
- Запрос только про вёрстку / токены / сетку без системного a11y → [`frontend-layout`](../frontend-layout/SKILL.md), этот скил дальше не веди.
- Запрос только про route tree / navigation / URL-state без системного a11y → [`frontend-routing`](../frontend-routing/SKILL.md), этот скил дальше не веди.
- `dangerouslySetInnerHTML` / unsanitized HTML / unsafe preview → [`frontend-security`](../frontend-security/SKILL.md), не чини через innerHTML.
- Нужно только размещение слайса → [`frontend-architecture`](../frontend-architecture/SKILL.md), не этот скил.
- Scope неясен → спроси компонент, экран, каталог или diff; не сканируй весь монорепо.
- Стек смешанный → проверяй каждый найденный UI-корень своим API; не унифицируй на React.
- Инструмента нет в lockfile (в том числе только в `package.json`) или нельзя гарантировать read-only → иди вручную, ограничение укажи в coverage.
- Контраст / screen reader / motion нельзя подтвердить → секция `INCOMPLETE`, не выдумывай `FAIL` и не меняй уже поставленный `FAIL`/`WARN`.
- Запрос «проверь» / «аудит» / path без «исправь» → оставайся в `audit`, даже если хочется сразу патч.
