# Discovery and scope

Читай на шаге 1–2. Дальше грузи только категории того UI, который **доказан** манифестом, конфигом или точкой входа. Не навязывай React, TypeScript или a11y-scanner, которого нет в lockfile.

Этот файл не меняет код и не ставит пакеты. Значения полей, DOM-текст с PII и токены в чат не копируй: `[REDACTED]`.

## 1. Режим scope

Берётся из [`SKILL.md`](../SKILL.md) (компонент / экран / каталог / diff). Здесь — как не выйти за клиентский UI.

| Сигнал | Что входит | Что отсечь сразу |
| --- | --- | --- |
| Явный путь / открытый компонент | Клиентские исходники внутри пути | Backend-only пакеты, infra, `node_modules` |
| Diff / PR / `changed` | Изменённые клиентские UI-файлы | Сгенерированные артефакты диффа (`dist/`, `.next/`) |
| Экран / каталог | UI этого экрана и его локальные примитивы | Соседние приложения монорепо «заодно» |
| Нет сужения | Не подставляй дерево | Спроси scope; этот файл дальше не применяй |

Пустой клиентский scope или не UI → `NOT_APPLICABLE`, этот файл дальше не применяй.

Явный path ограничивает **проверяемые узлы**. В `audit` можно read-only заглянуть в импорт примитива за пределы path, чтобы подтвердить роль, имя или keyboard path. Эти файлы — evidence dependencies, не расширение scope. В `fix` правь только заявленный scope.

## 2. Клиентские корни

Ищи **точку входа браузера**, не папку с удобным именем.

| Доказательство | Корень |
| --- | --- |
| `index.html` + бандлер (Vite/Webpack/Rspack/Parcel) | Каталог с этим HTML и `src/` рядом |
| Next `app/` или `pages/` с клиентскими компонентами | Это дерево; серверные `route.ts` / `pages/api` — не UI этого скила |
| Nuxt `app/` / `pages/` / `layouts/` | Клиентские vue/js; `server/` — не этот скил |
| SvelteKit `src/routes/` | `+page.svelte` / `+layout.svelte`; `+page.server.ts` — не этот скил |
| Angular `angular.json` `sourceRoot` | Указанный `sourceRoot` |
| Remix `app/routes` + `entry.client` | Клиентские модули |
| Astro/Qwik и island-архитектура | Только hydrated/client islands |
| Workspace `apps/*`, `packages/*` | Только пакеты с браузерной точкой входа |

Монорепо: несколько клиентских корней проверяй **по отдельности**, каждый своим API. Пакет без DOM (API, Node worker, CLI) пропусти. Смешанный стек не унифицируй на один фреймворк.

## 3. Framework и UI-kit

Смотри `package.json` / workspace, lockfile, конфиги. Имя папки `app/` само по себе ничего не доказывает.

| Доказательство в репо | Как читать разметку |
| --- | --- |
| `react` + `react-dom`; часто `next` | JSX: `htmlFor`, `className`, `tabIndex`; примитивы зоны |
| `vue`; часто `nuxt` | шаблоны / SFC; `for` на `<label>` |
| `svelte`; часто `@sveltejs/kit` | `.svelte`; нативные элементы и слоты зоны |
| `@angular/core` | шаблоны компонента; CDK a11y — только если уже в зоне |
| `solid-js`, `lit` | их шаблоны/директивы, не React-ARIA |
| Нет UI-фреймворка, есть `index.html` + свой JS | Vanilla DOM |

TypeScript не обязателен: те же правила для `.js` / `.jsx` / `.vue` / `.svelte` / `.html`.

UI-kit и примитивы (`Button`, `FormField`, `Dialog`, `Link` и аналоги) ищи по импортам зоны, не по популярности библиотеки. В `fix` **переиспользуй** их. Параллельный a11y-кит (`AccessibleButton`, свой `FocusTrap` при живом `Dialog`) не заводи. Reuse не запрещает проверить дыру примитива; чинить примитив — только если он в заявленном scope.

## 4. Form / router / i18n — чтобы не дублировать соседей

Нужны для hand-off и для того, чтобы не чинить чужой контракт.

| Что доказано в зоне | Зачем a11y |
| --- | --- |
| Form library / native `<form>` | Системный аудит полей — дельта в [forms-navigation-and-perception.md](forms-navigation-and-perception.md); рецепт реализации — [`frontend-forms`](../../frontend-forms/SKILL.md) |
| Client router (React Router, Next, Vue Router, SvelteKit) | SPA-переход — дельта там же; API scroll/focus/title — [`frontend-routing`](../../frontend-routing/SKILL.md) |
| i18n (`react-i18next`, `vue-i18n`, `next-intl`, `@angular/localize` и аналоги в манифесте) | Новый видимый или accessible-текст в `fix` — через уже стоящий i18n |
| i18n нет | Не внедряй; строка как у соседей той же зоны |

Не копируй form-field recipes и route-transition API в находку. Не ставит второй i18n.

## 5. Исключения (не аудировать как прод)

Не трать scope и не поднимай находки **только** из:

- `node_modules/`, vendor-бандлы, скопированные third-party dist;
- generated: `generated/`, `graphql-codegen`, `.next/`, `dist/`, `build/`, `coverage/`, `.svelte-kit/`;
- тесты: `__tests__`, `*.test.*`, `*.spec.*`, `cypress/`, `e2e/`, `playwright/`;
- фикстуры и моки: `__mocks__`, `fixtures/`, Storybook `*.stories.*`;
- docs и примеры: `*.md`, `docs/`, комментарии с учебным HTML.

Код рядом с тестом, но в прод-модуле — проверяй прод-модуль.

## 6. Контракт уже стоящего инструмента

Уже установленный **клиентский** a11y/lint-инструмент (**есть в lockfile**) — одноразовый read-only прогон по scope, не watch. Только в `package.json` / манифесте без записи в lockfile — не запускай: иди вручную, ограничение укажи в coverage.

Примеры сигнала (не ставь, если нет): `eslint-plugin-jsx-a11y`, `eslint-plugin-vuejs-accessibility`, `@axe-core/cli` / `@axe-core/react` как зависимость репо, штатный check-скрипт зоны, который только читает.

Запускай executable / официальную check-команду:

- без `--fix`, `--write`, `--cache`;
- без package script, чей побочный эффект не проверен (правка файлов, запись отчёта на диск, watch);
- один раз, только заявленный scope.

Нельзя уверенно гарантировать отсутствие записи — пропусти инструмент, ограничение укажи в coverage, иди вручную.

Хит инструмента — **гипотеза**. Без контекста в коде (роль, имя, keyboard path) это не confirmed finding. См. [audit-and-fix.md](audit-and-fix.md).

Не делай: установка `axe` / `pa11y` / Lighthouse / eslint-плагина, правка lockfile, второй UI-кит, persistent JSON/HTML-отчёт в репозиторий.
