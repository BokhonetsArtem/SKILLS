---
name: frontend-architecture
description: Определяет архитектуру текущего frontend-проекта и размещает код — границы модулей, public API, зависимости, state/API, рефакторинг структуры. Use when новый слайс/модуль, куда класть код, границы feature/entity, импорты, циклы, ownership state/API (включая куда класть form state) или реорганизация frontend-структуры. Не для вёрстки, CSS, Figma и не для генерации .ai-factory/ARCHITECTURE.md. Не для HTTP-клиента, endpoint, DTO mapping, cache и retry — это frontend-api. Не для поведения полей, validation и submit — это frontend-forms. Не для route tree, navigation, redirect и URL-state — это frontend-routing. Не для решения, нужен ли странице SSR/RSC/island — это frontend-ssr; куда класть server-модуль vs client island остаётся здесь. Не навязывает FSD и не добавляет второй архитектурный стиль или state manager, если их нет в репозитории.
disable-model-invocation: false
argument-hint: "[слайс | модуль | рефакторинг]"
---

# Frontend architecture

Размещай frontend-код по **фактической архитектуре этого репозитория**: слой/модуль, граница, public API, зависимости, state/API, безопасный перенос. Сначала определи подход проекта, затем найди ближайший аналог, затем клади код рядом с ним. Не выбирай FSD, feature-folders, layered или domain «по умолчанию» и не внедряй второй стиль «для удобства».

Не ограничивай себе инструменты. Для этой работы нужны доступные инструменты чтения, поиска, правки файлов и запуска уже существующих в репо проверок. Узкий read-only набор здесь не подходит.

Правила треков, границ, state/API и примеры **не копируй в этот файл**. Читай только нужный файл из `references/` (пути относительно этой папки скила).

**Не этот скил.** Вёрстка, CSS, токены, Flex/Grid, макет Figma — [`frontend-layout`](../frontend-layout/SKILL.md). HTTP-клиент, endpoint, DTO mapping, cache, retry, abort — [`frontend-api`](../frontend-api/SKILL.md); этот скил владеет только куда класть API-модуль и кто владеет state. Поведение полей, validation, submit и array fields — [`frontend-forms`](../frontend-forms/SKILL.md); этот скил владеет только размещением файлов формы/schema и тем, кто владеет form state. Route tree, navigation, redirect и URL-state — [`frontend-routing`](../frontend-routing/SKILL.md); этот скил владеет только куда класть page-модуль и public API. Системный a11y-аудит / WCAG — [`frontend-a11y`](../frontend-a11y/SKILL.md), не placement. Нужен ли странице SSR/RSC/island — [`frontend-ssr`](../frontend-ssr/SKILL.md); этот скил владеет только куда класть server-модуль vs client island / public API. Стратегический документ `.ai-factory/ARCHITECTURE.md` — `aif-architecture`. Задача только про тесты, CI или инфраструктуру без структуры frontend-кода — остановись, `references/` не читай.

Если задача сразу про **новый слайс и API** — сначала размещение (этот скил), затем HTTP-контракт ([`frontend-api`](../frontend-api/SKILL.md)), затем форма ([`frontend-forms`](../frontend-forms/SKILL.md)) и оболочка ([`frontend-layout`](../frontend-layout/SKILL.md)), если они нужны. Если сразу про **новый слайс и маршрут** — сначала размещение, затем декларация маршрута ([`frontend-routing`](../frontend-routing/SKILL.md)). Если сразу про **новый слайс/модуль и вёрстку** без API — сначала размещение, затем стили. Если сразу про **новый слайс и форму** — сначала размещение, затем поведение, затем оболочка, если она нужна. Не смешивай шаги и не переноси CSS-правила, HTTP HOW, form-binding или route tree сюда.

**Режим** (не грузи лишнее):

| Задача | Что читать |
| --- | --- |
| Архитектура неясна / смешанный проект | [references/principles-and-discovery.md](references/principles-and-discovery.md) |
| Выбран трек (FSD / feature-module / layered / domain) | [references/architecture-tracks.md](references/architecture-tracks.md) — только свой раздел |
| Public API, импорты, циклы | [references/boundaries-and-imports.md](references/boundaries-and-imports.md) |
| State / API / effects | [references/state-api-and-effects.md](references/state-api-and-effects.md) |
| Перенос кода / legacy | [references/refactoring.md](references/refactoring.md) |
| Неясно, как надо | [references/examples.md](references/examples.md) |
| Спорная классификация трека | [references/evaluation.md](references/evaluation.md) |

## Шаг 1. Классифицируй изменение

Опора: сообщение пользователя, `$ARGUMENTS` (если скил вызван с аргументом), открытые файлы, текущая структура `src/` (или аналога). В обычном чате `$ARGUMENTS` может быть пустым — это не стоп.

| Сигнал | Режим |
| --- | --- |
| Новый экран, фича, слайс, модуль, домен «с нуля» | Новая возможность / слайс |
| Добавление в уже существующий модуль без новой границы | Расширение модуля |
| Нарушение направления зависимостей, deep import, цикл, код не в своём слое | Исправление нарушения |
| Перенос файлов, нарезка god-module, выравнивание legacy | Рефакторинг структуры |
| Куда класть query/mutation, store, form state, side effects | Проектирование API / state flow |
| HTTP-клиент, endpoint, DTO, cache, retry без новой границы | [`frontend-api`](../frontend-api/SKILL.md) — не этот скил |
| Route tree, navigation, redirect, URL-state без новой границы | [`frontend-routing`](../frontend-routing/SKILL.md) — не этот скил |
| Нужен ли SSR/RSC/island без новой границы модуля | [`frontend-ssr`](../frontend-ssr/SKILL.md) — не этот скил |

Несколько сигналов сразу — веди по самому широкому: сначала граница и владелец, потом маршрут ([`frontend-routing`](../frontend-routing/SKILL.md)), если нужен, потом HTTP-контракт ([`frontend-api`](../frontend-api/SKILL.md)), если нужен, потом перенос. Вёрстку на этом шаге не начинай.

## Шаг 2. Обнаружь архитектуру и enforcement

Смотри репозиторий, не привычку модели. Не классифицируй подход только по имени папки.

Что проверить (достаточно сигналов, не полный аудит репо):

- дерево `src/` (или аналога) и соседние слайсы/модули той же зоны;
- alias и config: `tsconfig`/`jsconfig` `paths`, Vite/Webpack alias;
- манифест: `package.json` / lockfile — `@feature-sliced/*`, слои, state/data-fetching библиотеки;
- enforcement: `steiger`, `@feature-sliced/*`, `eslint-plugin-boundaries`, Nx `enforce-module-boundaries`;
- реальные импорты в зоне изменения, не только README.

Библиотеку state и data fetching бери из репозитория (то, чем уже пользуются соседи). Не подставляй Redux, RTK Query, Zustand, TanStack Query или другой клиент, если его нет в проекте. Как писать HTTP/query — [`frontend-api`](../frontend-api/SKILL.md).

Каркас фреймворка (`app/` у Next.js App Router, `routes/` у Remix и т.п.) — не слой FSD только из-за имени. Если неясно, это framework-owned каталог или архитектурный слой — читай [principles-and-discovery.md](references/principles-and-discovery.md).

| Что видно в репо | Трек | Что читать |
| --- | --- | --- |
| Живые FSD-слои со слайсами и импортами вниз. Enforcement (`steiger`, `@feature-sliced/*`) **усиливает**, но не обязателен. Одни имена папок — мало | FSD | [architecture-tracks.md](references/architecture-tracks.md) — раздел FSD; матрица — [principles-and-discovery.md](references/principles-and-discovery.md). Слой `processes/` не создавай, если его нет. Соглашения **этого** репо, не «идеальный FSD» |
| Вертикальные фичи/модули (`features/`, `modules/`, `src/cart/` с колокацией UI+API), без FSD-слоёв и без внутренних `domain/`+`application/` | feature / module | [architecture-tracks.md](references/architecture-tracks.md) — раздел feature-module |
| Технические папки: `pages`/`screens` + `components` + `hooks`/`services`/`store` | layered | [architecture-tracks.md](references/architecture-tracks.md) — раздел layered. Не заводи `domain/`+`application/` |
| Вертикальный домен с внутренними слоями (`src/<domain>/` + `domain`, `application`, `ui`, `data`) | domain | [architecture-tracks.md](references/architecture-tracks.md) — раздел domain. Не путай с feature-module |
| Два равноправных стиля или сигналов мало | смешанный / неясный | [principles-and-discovery.md](references/principles-and-discovery.md), сверка — [evaluation.md](references/evaluation.md), затем спроси пользователя |

Смешанный или неизвестный подход — **не** мигрируй проект на FSD и **не** заводи второй архитектурный стиль. Спроси, в каком стиле вести новое место, и клади код как ближайшие соседи выбранной зоны.

## Шаг 3. Найди ближайший аналог

Паттерн бери у **ближайшего модуля той же зоны и того же типа**, не у первого попавшегося Grep и не из другого репозитория.

Ищи существующий слайс/фичу/страницу с той же ролью: список, форма создания, карточка сущности, API-обвязка, store. Скопируй у аналога: место в дереве, public API, сегменты/папки, направление импортов, где живут query/mutation и UI-композиция.

Имена, alias и сегменты — как **здесь**. Пути вроде `@/entities/skill` или `@/shared/config` не переноси в проект, где их нет.

Нет близкого аналога — читай [examples.md](references/examples.md) и раздел выбранного трека. Всё ещё пусто или спорно — спроси, не выдумывай канон.

## Шаг 4. Выбери слой/модуль и public API

По выбранному треку и аналогу реши:

- куда кладётся новый код (слой, домен, модуль, слайс);
- какие **явно поддерживаемые** публичные точки входа уже есть (barrel, `package.json` `exports`, разрешённые subpath) — это public API;
- что остаётся внутренним и не импортируется снаружи.

Детали — [boundaries-and-imports.md](references/boundaries-and-imports.md) и раздел трека. Для **куда** класть state, query/mutation и эффекты — [state-api-and-effects.md](references/state-api-and-effects.md). **Как** писать HTTP-клиент, DTO, cache и retry — [`frontend-api`](../frontend-api/SKILL.md); правила транспорта сюда не копируй.

Не раздувай `shared`/common свалкой доменного UI и бизнес-логики. Не дроби модуль без причины, если аналог в этой зоне живёт крупнее. Не тащи доменный код на page/container, если в репо страница уже только композирует.

После размещения, если нужна декларация маршрута / navigation / URL-state — передай её в [`frontend-routing`](../frontend-routing/SKILL.md). Если нужен HTTP-контракт (endpoint, query/mutation, cache) — передай его в [`frontend-api`](../frontend-api/SKILL.md). Если нужна вёрстка экрана — передай её в [`frontend-layout`](../frontend-layout/SKILL.md). Если нужно поведение формы (validation, submit, array fields) — передай его в [`frontend-forms`](../frontend-forms/SKILL.md). Шаги не смешивай.

## Шаг 5. Проверь зависимости и интеграцию

Перед сдачей размещения:

- направление зависимостей совпадает с треком и аналогом (не наоборот);
- снаружи модуля нет deep import мимо public API, если репо это запрещает или аналоги так не делают;
- нет нового цикла между модулями;
- кросс-импорт идёт через принятый в репо контракт (public API, `@x`, facade — как у соседей, не «как в учебнике»);
- новый экран/маршрут/store подключён там, где в этом репо подключают такие вещи — по аналогу, не в случайный файл.

Перенос существующего кода — [refactoring.md](references/refactoring.md): сначала сохрани публичный контракт, потом двигай внутренности. Не рефакторь соседние модули «заодно».

## Шаг 6. Статические проверки

Запускай только то, что **уже есть** в репозитории, и только по затронутой области. Watch-режим не запускай.

Есть архитектурный checker (`steiger`, `eslint-plugin-boundaries`, Nx boundaries) — одноразовый прогон, не watch. Если npm-скрипт включает `--watch` — вызови CLI напрямую без watch, не этот скрипт.

Нет архитектурного линтера — не добавляй его в этом скиле. Typecheck/lint затронутых файлов — если команда уже есть и уместна.

Не ужесточай конфиг enforcement (не включай выключенные правила, не расширяй scope), пока пользователь явно не попросил.

## Отчёт о размещении

Перед сдачей — коротко, не эссе:

- **Трек:** FSD / feature-module / layered / domain / смешанный (и что спросили у пользователя).
- **Доказательства:** 2–3 факта из репо (дерево зоны, характерный импорт, enforcement или его отсутствие).
- **Аналог:** путь ближайшего модуля того же типа.
- **Размещение:** куда положен код и какой public API (какие точки входа, что внутреннее).
- **Проверки:** что запустили по затронутой области / чего в репо нет.

Спорная классификация — сверься с [evaluation.md](references/evaluation.md), не угадывай канон.

## Если что-то пошло не так

- Архитектура неясна или в зоне два стиля → спроси. Не подставляй FSD, feature-folders, layered или domain наугад.
- Нет аналога → ищи соседние слои **этого** репо; чужие пути и сегменты не копируй.
- Пользователь не просил миграцию → не переводи проект на FSD и не создавай параллельный стиль рядом со старым.
- Пользователь не просил новый state manager → не добавляй второй клиент к уже существующему.
- Нужна только вёрстка без новой границы модуля → это [`frontend-layout`](../frontend-layout/SKILL.md), не этот скил.
- Нужны HTTP-клиент, endpoint, cache или retry без новой границы → это [`frontend-api`](../frontend-api/SKILL.md), не этот скил.
- Нужны validation, submit или array fields без новой границы → это [`frontend-forms`](../frontend-forms/SKILL.md), не этот скил.
- Нужны route tree, navigation, redirect или URL-state без новой границы модуля → это [`frontend-routing`](../frontend-routing/SKILL.md), не этот скил.
- Нужен ли странице SSR/RSC/island без новой границы модуля → это [`frontend-ssr`](../frontend-ssr/SKILL.md), не этот скил.
- Нужен стратегический документ архитектуры проекта → это `aif-architecture`, не этот скил.
