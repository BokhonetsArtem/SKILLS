# Discovery and scope

Читай на шаге 1–2. Дальше грузи только разделы того стека, который **доказан** манифестом, конфигом или точкой входа. Не навязывай React, TypeScript или scanner, которого нет в lockfile.

Этот файл не меняет код и не ставит пакеты. Сырые dotenv, vault, SSH-ключи и credential stores **не открывай**.

## 1. Режим scope

Берётся из [`SKILL.md`](../SKILL.md) (явный путь / changed frontend / клиентское дерево). Здесь — как не выйти за клиент.

| Сигнал | Что входит | Что отсечь сразу |
| --- | --- | --- |
| Явный путь | Клиентские исходники внутри пути | Backend-only пакеты, infra, `node_modules` |
| Diff / PR | Изменённые клиентские файлы | Сгенерированные артефакты диффа (`dist/`, `.next/`) |
| Нет сужения | Клиентские корни из §2 | Весь монорепо «на всякий случай» |

Пустой клиентский scope → остановись, этот файл дальше не применяй, укажи `aif-security-checklist`.

## 2. Клиентские корни

Ищи **точку входа браузера**, не папку с удобным именем.

| Доказательство | Корень |
| --- | --- |
| `index.html` + бандлер (Vite/Webpack/Rspack/Parcel) | Каталог с этим HTML и `src/` рядом |
| Next `app/` или `pages/` с клиентскими компонентами | Это дерево; серверные `route.ts` / `pages/api` **не** аудируй здесь |
| Nuxt `app/` / `pages/` / `layouts/` | Клиентские vue/js; `server/` — не этот скил |
| SvelteKit `src/routes/` | `+page.svelte` / `+layout.svelte` / `+page.ts`; `+page.server.ts` — не этот скил |
| Angular `angular.json` `sourceRoot` / `projects.*.architect.build` | Указанный `sourceRoot` |
| Remix `app/routes` + `entry.client` | Клиентские модули и `entry.client*` |
| Astro/Qwik и island-архитектура | Только hydrated/client islands, browser entry и подключаемые scripts |
| Workspace `apps/*`, `packages/*` | Только пакеты с браузерной точкой входа (см. `package.json` `dependencies` + entry) |

Внутри найденного корня не забудь `public/`/static HTML, web manifest, service worker и browser workers, если они реально деплоятся. Монорепо: несколько клиентских корней аудируй **по отдельности**, каждый своим API. Пакет без DOM/бандлера (API, workers на Node, CLI) пропусти.

## 3. Как узнать фреймворк

Смотри `package.json` / workspace, lockfile, конфиги. Имя папки `app/` само по себе ничего не доказывает.

| Доказательство в репо | Стек для [browser-vulnerabilities.md](browser-vulnerabilities.md) |
| --- | --- |
| `react` + `react-dom`; часто `next` | React / Next (SSR+client) |
| `vue`; часто `nuxt` | Vue / Nuxt |
| `svelte`; часто `@sveltejs/kit` | Svelte / SvelteKit |
| `@angular/core` | Angular |
| `solid-js`, `lit`, `jquery` | Не React: бери **их** unsafe API (ниже), не `dangerouslySetInnerHTML` |
| Нет UI-фреймворка, есть `index.html` + свой JS | Vanilla DOM |
| Nest/Express/Go **без** клиентского бандла в scope | Клиента нет → чеклист, не этот скил |

Дополнительно, только если пакет **есть** в манифесте:

- jQuery: `.html()`, `.append()`, `.attr('href')` — как vanilla sink, не как React.
- Lit: `unsafeHTML`, `unsafeSVG`.
- Solid: свойство `innerHTML` / `textContent` по факту кода.

TypeScript не обязателен: те же правила для `.js` / `.jsx` / `.vue` / `.svelte`.

## 4. SSR: что здесь, что нет

Нужен ли SSR/RSC/island, mismatch first render и запрет server-only import в client — [`frontend-ssr`](../../frontend-ssr/SKILL.md). Этот раздел — браузерные sinks после гидрации, не решение render-mode.

Клиент после гидрации, `'use client'`, браузерные хуки, `next/script` / `ClientOnly` / `onMount` — здесь.

Не аудируй в этом скиле: SQL/ORM, серверные `headers()`, cookie flags, CSRF-токен сервера, `getServerSideProps` как API-auth, rate limit, CSP в nginx. Это `aif-security-checklist`.

HTML, который сервер отдал в тот же компонент и затем гидратирует в DOM-sink — клиентский поток, если sink браузерный.

## 5. Публичные env-префиксы

Только префиксы, которые **уже приняты бандлером этого репо**. Не угадывай.

| Доказательство | Обычно попадает в клиентский бандл |
| --- | --- |
| Vite / `vite.config.*` | `VITE_` |
| Next | `NEXT_PUBLIC_` |
| CRA / многие webpack-шаблоны | `REACT_APP_` |
| Nuxt 3 | `NUXT_PUBLIC_` / runtime `public` в `nuxt.config` |
| SvelteKit | `PUBLIC_` |
| Expo / RN web | `EXPO_PUBLIC_` |
| Angular | значения из `environment*.ts` / `fileReplacements` — это исходник, его читать можно |

Секрет без публичного префикса, который импортирован в клиентский модуль, — находка по [client-data-and-auth.md](client-data-and-auth.md), не по чтению сырого dotenv.

## 6. Исключения (не аудировать как прод)

Не трать scope и не поднимай находки **только** из:

- `node_modules/`, vendor-бандлы, скопированные third-party dist;
- generated: `generated/`, `graphql-codegen`, `.next/`, `dist/`, `build/`, `coverage/`, `.svelte-kit/`;
- тесты: `__tests__`, `*.test.*`, `*.spec.*`, `cypress/`, `e2e/`, `playwright/`;
- фикстуры и моки: `__mocks__`, `fixtures/`, `msw` handlers, Storybook `*.stories.*`;
- docs и примеры: `*.md`, `docs/`, комментарии с учебным HTML.

Код рядом с тестом, но в прод-модуле — аудируй прод-модуль.

Исключение для подтверждения: точечно читай уже существующий deploy artifact/source map, если нужно доказать suspected env/secret exposure или публикацию исходников. Не сканируй весь generated bundle как первичный source и не запускай build ради этого.

## 7. Инструменты

Уже установленный **клиентский** линтер/сканер (например плагин про unsanitized DOM, если он в манифесте) — одноразовый read-only прогон по scope, не watch. Не используй `--fix`, `--write`, cache и package script, если его побочные эффекты не проверены; при сомнении пропусти и укажи limitation.

Не делай: добавление зависимости, `npm audit`, классификация CVE, правка lockfile. Хит инструмента без source-to-sink — неподтверждённый сигнал, см. [evidence-and-reporting.md](evidence-and-reporting.md).
