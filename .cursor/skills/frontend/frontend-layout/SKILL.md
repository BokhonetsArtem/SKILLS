---
name: frontend-layout
description: Верстает страницы и блоки UI по стеку репозитория — reuse примитивов, Flex/Grid, токены, сверка с макетом. Use when нужна вёрстка, layout, таблица/карточки, визуальная оболочка формы (сетка, FormField, CSS), стили UI или макет из Figma MCP / Bridge. Не для API, роутинга, задач только про тесты и правок без UI — HTTP/query без UI это frontend-api. Не для form state, validation, submit и array fields — это frontend-forms. Не добавляет CSS Modules, Tailwind, styled-components или UI-кит, если их нет в проекте.
disable-model-invocation: false
argument-hint: "[текст | Figma URL]"
---

# Frontend layout

Верстай страницы и блоки по **тексту задачи** или **макету** (Figma MCP / Bridge). Стек и синтаксис стилей — **только из этого репозитория**. Не выбирай CSS Modules или Tailwind по умолчанию и не внедряй второй способ «для удобства».

Не ограничивай себе инструменты. Для этой работы нужны доступные инструменты чтения и редактирования файлов, а для макета — MCP (`GetDynamicTools` → `CallDynamicTool`). Узкий read-only набор здесь не подходит.

Правила, треки и примеры **не копируй в этот файл**. Читай только нужный файл из `references/` (пути относительно этой папки скила).

**Не этот скил.** Клиентский HTTP/REST/JSON без UI (endpoint, query/mutation, cache) — [`frontend-api`](../frontend-api/SKILL.md). Роутинг без вёрстки, задача только про тесты, рефактор модели. Размещение нового слайса/модуля/границы — зона [`frontend-architecture`](../frontend-architecture/SKILL.md); этот скил владеет стеком стилей, примитивами и Figma. Form state, validation, submit и array fields — [`frontend-forms`](../frontend-forms/SKILL.md); этот скил владеет визуальной оболочкой формы (сетка, `FormField`, CSS, Figma), не поведением полей. Если задача сразу про новый слайс и вёрстку — сначала размещение, затем стили. Если сразу про форму — размещение (architecture) → HTTP-контракт ([`frontend-api`](../frontend-api/SKILL.md)), если нужен endpoint → поведение ([`frontend-forms`](../frontend-forms/SKILL.md)) → оболочка (здесь), если она нужна. Проверка или обновление существующих stories / visual tests как часть UI-задачи допустимы. Если открыт `tsx`/`css`, но задача не про layout — не читай `references/` и не применяй правила вёрстки.

**Масштаб** (не грузи лишнее):

| Задача | Что читать |
| --- | --- |
| Правка существующего блока | [references/pravila.md](references/pravila.md) + файл трека со шага 2 |
| Новая страница / таблица / форма | + [references/patterny.md](references/patterny.md), если в репо уже есть те же оболочки; иначе ближайшая страница той же зоны |
| Макет Figma | [references/figma.md](references/figma.md) на шаге 1, затем стек |
| Неясно, как надо | + [references/primery.md](references/primery.md) |

## Шаг 1. Определи вход

Опора: сообщение пользователя, `$ARGUMENTS` (если скил вызван с аргументом), вложения, выделенный фрейм в Figma. В обычном чате `$ARGUMENTS` может быть пустым — это не стоп.

| Сигнал | Режим |
| --- | --- |
| Figma URL, node id, выделенный фрейм, явное «по макету» | Макет |
| Только текст («сверстай таблицу с фильтрами») | Текст |

**Режим макета.** Сразу прочитай [references/figma.md](references/figma.md) целиком и следуй ему: сначала `GetDynamicTools` по namespace, схему инструмента — до `CallDynamicTool`. Если `namespaceStatus` = `needsAuth` — вызови `mcp_auth`, затем повтори discovery. Нет MCP, auth не прошёл или инструментов нет — кратко сообщи об этом. Продолжай в **текстовом режиме**, только если сообщения, вложений или существующих экранов достаточно; иначе запроси скриншот или описание. Не выдумывай сетку, отступы и цвета макета. Код из MCP — reference (часто React + Tailwind), не копипаста: перенеси иерархию на **стек репозитория**.

**Режим текста.** Верстай по описанию и паттернам проекта. Не подставляй «как в макете», если макета нет.

## Шаг 2. Определи стек

Смотри репозиторий (`package.json`, конфиги, ближайшие UI-файлы той же зоны), не привычку модели. Не выбирай стек из привычки и не внедряй второй способ «для удобства».

| Что видно в репо | Трек | Что читать сверх pravila |
| --- | --- | --- |
| Доминируют `*.module.css` / `import styles from`, нет Tailwind и нет Sass | CSS Modules | [references/css-modules.md](references/css-modules.md) |
| `*.module.scss` / `*.module.sass` (CSS Modules на Sass) | Modules + SCSS | [css-modules.md](references/css-modules.md) + [scss.md](references/scss.md): scoping и `import styles` как Modules, синтаксис (`$`, `@use`, nest) как SCSS. Не переводи на `.module.css` и не добавляй Tailwind |
| Есть `tailwind.config*` / `@tailwind` / `tailwindcss`, утилиты в `className`, нет доминирующих modules | Tailwind | [references/tailwind.md](references/tailwind.md) |
| `unocss` / `windicss` (атомарные классы, не Tailwind) | atomic | [references/tailwind.md](references/tailwind.md) — раздел UnoCSS; синтаксис как у соседей, не подменяй на Tailwind |
| Есть Tailwind **и** много `*.module.css` | hybrid | [css-modules.md](references/css-modules.md) + [tailwind.md](references/tailwind.md): layout утилитами, сложные селекторы — в module |
| `styled-components`, `@emotion/*`, stitches, vanilla-extract, StyleX, Linaria, Panda | CSS-in-JS | [references/css-in-js.md](references/css-in-js.md) |
| Глобальные `*.scss` / `*.sass` / `*.less` или обычный `.css` **без** `.module.` и без Tailwind | SCSS/CSS | [references/scss.md](references/scss.md) |
| Рядом и `*.module.css`, и редкий глобальный `.scss` | как в этой зоне | не переводи всё на SCSS и не смешивай в одном компоненте без прецедента |
| `@mui/*`, `antd`, `@chakra-ui/*`, `@mantine/core`, доминируют `sx` / кит | UI-kit | [references/ui-kit.md](references/ui-kit.md) |
| Vue `scoped` / Svelte `<style>` / иной стек | стек репозитория | чужие файлы трека **не читай**. Копируй синтаксис и токены ближайших файлов |

Применяй только найденный трек. Пакеты из остальных треков не добавляй.

Всегда прочитай [references/pravila.md](references/pravila.md) (layout, gap, токены, a11y — без привязки к синтаксису).

Стек неясен или в зоне два равноправных способа — спроси, не мешай.

## Шаг 3. Найди и переиспользуй примитивы

Не изобретай кнопку, текст, карточку, поле, таблицу, ряд. Сначала `Grep` / `Glob` по `src/` (или аналогу в этом репо). Паттерн бери у **ближайшей страницы той же зоны**, не у первого попавшегося Grep.

Имена и пути примитивов — как **здесь**, не из другого проекта. Типичные кандидаты: layout-ряд, текст, карточка, кнопка, поле формы, таблица, page shell, поиск/фильтры, иконки.

Список ниже — **только если эти импорты уже есть в репозитории**. Не тащи их в проект, где их нет:

- `Flex`, `Text`, `Card`, `Button`, `FormField`
- `PageWrapper`, `SearchSection`, `Table`
- Админ-таблица: `FiltersDrawer` только через `SearchSection` (`renderFilter`)
- Wiki / каталог: колонка фильтров, `FiltersDrawer` на узком экране — как `QuestionsPage`, не как админ-таблица
- Иконки: существующие SVG/компоненты проекта, не рисуй SVG «на глаз»

Токены — откуда их уже берут соседи: CSS `var(--*)`, `@theme` / `tailwind.config`, тема UI-кита, styled theme.

Нет примитива в проекте — не тащи чужой UI-кит, пока пользователь явно не попросил. Собери из существующих или положи UI-блок в уже выбранное место модуля. Новый слайс/модуль/граница — сначала [`frontend-architecture`](../frontend-architecture/SKILL.md), не этот скил.

## Шаг 4. Выбери паттерн и сверстай

По типу задачи читай только нужное (progressive disclosure — не грузи все references сразу):

| Задача | Файл |
| --- | --- |
| Админ-таблица + фильтры, wiki/каталог, карточки, форма, page shell, адаптив — и в репо уже есть такие оболочки | [references/patterny.md](references/patterny.md) |
| В другом проекте нет этих оболочек | ближайшая страница той же зоны в **этом** репо; `patterny.md` не копируй как обязательный канон |
| Неясно, как надо / как не надо | [references/primery.md](references/primery.md) |
| Макет Figma | [references/figma.md](references/figma.md) — уже прочитан на шаге 1 |

Дальше верстай синтаксисом выбранного стека.

## Шаг 5. Чеклист перед сдачей

Если пункт не выполнен — почини, не сдавай. Пункты про конкретный инструмент — только если он **есть** в репозитории.

- **Токены / линтер.** Цвет, фон, `font-size` / `font-weight` / `font-family` — из токенов проекта, не сырой hex/px. Есть stylelint / eslint-стили — соблюдай их, включая порядок свойств.
- **Gap.** Ритм через `gap` на родителе и padding контейнера, не `margin-right` / last-child. Если в проекте layout-примитив с пропом `gap` — используй его.
- **a11y.** Визуальный порядок = DOM (не `order` / `row-reverse` для интерактива); семантика (`main`, кнопки, подписи); keyboard flow; hit-area; видимый focus; смысл не только цветом.
- **Adaptive.** Брейкпоинты и хуки **этого** репозитория. Не канонизируй компоненты, которые страницы не используют.
- **i18n.** Если в проекте есть i18n — новый текст через него, не хардкод. Нет i18n — не внедряй.
- **Ассеты.** Скачанный из Figma файл лежит в каталоге ассетов репозитория, не вне его.
- **Сверка в браузере.** Если менялся UI — проверь поток как пользователь (клик, ввод, навигация), не один скриншот. Нет браузерных инструментов — явно скажи, что визуально не проверил.
- **Проверки репозитория.** Запусти релевантные lint / typecheck / build-команды для затронутой области. Если компонент уже покрыт Storybook / Loki или другим visual regression — проверь его существующим способом; baseline без явной причины не обновляй.

## Если что-то пошло не так

- Нет Figma MCP / `needsAuth` не прошёл → сообщи пользователю; продолжай по тексту только при достаточном контексте, иначе запроси скриншот или описание.
- Стек неясен → спроси. Не подставляй CSS Modules, Tailwind, styled-components или UI-кит наугад.
- Примитив не найден → ищи соседние слои/папки **этого** репо, не копируй из другого проекта.
- Нужны form state, validation, submit или array fields → это [`frontend-forms`](../frontend-forms/SKILL.md), не этот скил.
- Нужен только HTTP-клиент, endpoint, query/mutation или cache без UI → это [`frontend-api`](../frontend-api/SKILL.md), не этот скил.
