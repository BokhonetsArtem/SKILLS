---
name: frontend-security
description: >-
  Ищет уязвимости клиентской части по доказательствам source-to-sink и выдаёт
  приоритизированный отчёт без правок кода. Use when frontend/client audit: DOM
  XSS, unsafe URL, postMessage, iframe/worker, client tokens, public env,
  client-only authorization; XSS/unsafe preview поля. Не для generic «is this
  secure?», pre-deploy/OWASP, CSP headers, server auth, CSRF, SQL/command
  injection, rate limiting, infrastructure и dependency CVE — это
  aif-security-checklist. Не для обычной валидации и submit формы — это
  frontend-forms, не повод начинать аудит. Не ставит пакеты и не открывает
  сырые secret-файлы.
---

# Frontend security

Аудит **клиентской** части по доказательствам source-to-sink. Итог — приоритизированный отчёт. Файлы не меняй, пакеты не ставь, автоисправления не предлагай патчем и не применяй. Логи и телеметрию не добавляй: в чат выводи **только итог аудита**.

Правила sinks, data/auth и evidence **не копируй в этот файл**. Читай только нужный файл из `references/` (пути относительно этой папки скила).

**Не этот скил.** Generic «is this secure?», pre-deploy / OWASP-чеклист, CSP-заголовки, серверная auth, CSRF, SQL/command injection, rate limiting, инфраструктура, CVE зависимостей — [`aif-security-checklist`](../../aif-security-checklist/SKILL.md); `references/` не читай. Вёрстка — [`frontend-layout`](../frontend-layout/SKILL.md). Размещение модулей — [`frontend-architecture`](../frontend-architecture/SKILL.md). Обычная валидация и submit формы — [`frontend-forms`](../frontend-forms/SKILL.md), не повод начинать аудит. XSS / unsafe preview поля — этот скил.

Пересечение (XSS, секреты в клиенте) **не** объявляй нулевым: этот скил владеет глубиной source-to-sink в браузерном/клиентском коде; чеклист — общей pre-deploy/OWASP-проверкой. Спорный сигнал без клиентского data flow отдай чеклисту, не дублируй его gate.

**Режим** (не грузи лишнее):

| Этап | Что читать |
| --- | --- |
| Scope, фреймворк, корни приложения, публичные env-префиксы, исключения | [references/discovery-and-scope.md](references/discovery-and-scope.md) |
| Browser sinks (DOM/HTML, URL/navigation, postMessage, iframe/worker, upload/preview, service worker, third-party scripts) | [references/browser-vulnerabilities.md](references/browser-vulnerabilities.md) |
| Клиентские данные и auth (public env, хранение токенов, URL, client-only authorization, PII/ошибки/телеметрия) | [references/client-data-and-auth.md](references/client-data-and-auth.md) |
| Severity, уверенность, evidence, неподтверждённые сигналы, формат отчёта | [references/evidence-and-reporting.md](references/evidence-and-reporting.md) |

## Шаг 1. Выбери scope

Опора: сообщение пользователя, `$ARGUMENTS` (если скил вызван с аргументом), открытые клиентские файлы. В обычном чате `$ARGUMENTS` может быть пустым — это не стоп.

| Сигнал | Режим |
| --- | --- |
| Явный путь к файлу/каталогу | Указанный path — только клиентские файлы внутри него |
| `changed`, `diff`, PR, незакоммиченные правки | Изменённые frontend-файлы (git diff / status), затем фильтр по клиентскому дереву |
| Нет сужения | Клиентское дерево проекта, найденное на шаге 2 |

Scope пустой, только backend/infra или нет клиентского кода — верни `NOT_APPLICABLE`, `references/` дальше не читай, укажи [`aif-security-checklist`](../../aif-security-checklist/SKILL.md), если запрос про общую безопасность.

Детали корней, монорепо и исключений — [discovery-and-scope.md](references/discovery-and-scope.md).

Явный path ограничивает **проверяемые sinks**, но не обрывает доказательство: разрешено read-only проследить импорт/вызов за пределы path ровно настолько, чтобы подтвердить источник, sanitizer или downstream consumer. Эти файлы укажи как evidence dependencies и не считай расширением audit scope.

## Шаг 2. Обнаружь стек и границы клиента

Смотри репозиторий, не привычку модели. Не навязывай React, TypeScript или scanner, которого нет в lockfile/манифесте.

Что проверить для **discovery** (достаточно сигналов, не полный аудит на этом этапе):

- манифест и конфиги: `package.json` / workspace, Vite/Webpack/Next/Nuxt/Angular/SvelteKit;
- корни приложения: `src/`, `app/`, `apps/*`, `packages/*` с клиентской точкой входа;
- публичные env-префиксы **по доказательствам репо** (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `NUXT_PUBLIC_` и аналоги), не по догадке;
- исключения: generated, vendor, `node_modules`, тесты, фикстуры, docs, markdown-примеры.

Сырые secret-файлы (dotenv, vault, SSH, credential stores) **не открывай**. Утечку ищи в клиентском исходнике и в публичных префиксах, которые уже попали в бандл.

Уже установленный **клиентский** инструмент (например eslint-плагин про DOM/unsanitized, если он есть в манифесте) — одноразовый read-only прогон по scope, не watch. Запускай executable/официальную check-команду без `--fix`, `--write`, `--cache` и package scripts с неизвестными побочными эффектами. Нельзя уверенно гарантировать отсутствие записи — пропусти инструмент и отметь ограничение. Нет такого инструмента — не добавляй зависимость, иди вручную по source-to-sink.

Не делай: установка пакетов, `npm audit`, классификация CVE зависимостей, правки lockfile.

Фреймворк и unsafe API бери из [discovery-and-scope.md](references/discovery-and-scope.md); escaping-гарантии конкретного стека не подменяй «как в React».

## Шаг 3. Проверь browser sinks

Читай [browser-vulnerabilities.md](references/browser-vulnerabilities.md), если scope включает DOM, URL/navigation, messaging, iframe/worker, upload/preview, service worker или third-party scripts. Для узкого запроса только про env/token/client auth этот reference не нужен. Читай только разделы стека и browser API, найденных на шаге 2.

Ищи цепочки **источник → поток → sink** в scope. Имена API и escaping — как у этого фреймворка, не копируй React-only правила в Vue/Svelte/Angular/vanilla.

Не вставляй в отчёт и в чат готовые exploit/shell-инструкции. Описывай sink и поток данными репозитория.

## Шаг 4. Проверь клиентские данные и auth

Читай [client-data-and-auth.md](references/client-data-and-auth.md), если scope включает public env, credentials, token/session data, client-side access checks, PII, ошибки, логи или telemetry. Для узкого запроса только про DOM/URL sink без этих данных reference не нужен.

Смотри: попадание секретов в клиентский бандл через публичные env; токены в `localStorage` / URL / открытых полях; проверки доступа только на клиенте; PII и внутренности ошибок в UI/телеметрии.

Серверную сессию, cookie flags, CSRF-токены сервера и API authorization не аудируй здесь — это чеклист.

## Шаг 5. Собери доказательства и отчёт

Читай [evidence-and-reporting.md](references/evidence-and-reporting.md) перед финальным статусом.

Подтверждённая находка — только с `file:line`, источником, sink, data flow, impact, preconditions и уровнем уверенности. Без этого — неподтверждённый сигнал, не уязвимость.

Не считай уязвимостью без дополнительных доказательств: штатный escape фреймворка, markdown-примеры, fixtures/mocks, поток через **проверенный** sanitizer/Trusted Types policy, подходящий конкретному sink и контексту.

Формат итога (только в чат, файл отчёта не создавай):

- статус: `PASS` | `WARN` | `FAIL` | `INCOMPLETE` | `NOT_APPLICABLE`;
- scope и фреймворк;
- coverage: проверенные корни/классы рисков и ограничения;
- находки по severity;
- неподтверждённые сигналы;
- безопасная remediation-рекомендация для каждой находки, без патча и exploit PoC;
- hand-off в `/aif-security-checklist` для серверных/OWASP/infra/CVE тем.

`PASS` означает только «в проверенном scope нет confirmed findings», не «клиент безопасен целиком». Ошибка инструмента, недоступные файлы или оборванный обязательный data flow → `INCOMPLETE`; пустой/не-клиентский scope → `NOT_APPLICABLE`.

Значения секретов, токенов и PII в отчёт не копируй. Указывай тип/идентификатор и `file:line`, значение редактируй как `[REDACTED]`.

Не используй занятый `gate: security` и блок `aif-gate-result`. Не пиши `SECURITY.md` и любые persistent reports. Не добавляй waivers/allowlists.

## Запрещено

- менять файлы проекта, скилов, конфигов агента;
- ставить пакеты и менять зависимости;
- автоисправлять находки;
- добавлять логи, telemetry, debug-вывод в код или в скил;
- `npm audit` и CVE-классификация зависимостей;
- открывать сырые secret-файлы;
- копировать exploit PoC или attack procedures.

## Если что-то пошло не так

- Запрос generic «надежно ли это?» / pre-deploy / OWASP → [`aif-security-checklist`](../../aif-security-checklist/SKILL.md), этот скил дальше не веди.
- Обычная валидация / submit формы без XSS-аудита → [`frontend-forms`](../frontend-forms/SKILL.md), этот скил дальше не веди.
- Клиентское дерево не находится → спроси path; не сканируй весь монорепо наугад.
- Стек смешанный → аудируй каждый найденный клиентский корень своим API; не унифицируй на React.
- Находка только серверная → не флагь как frontend FAIL; отдай в чеклист.
- Пользователь просит «сразу починить» → отчёт всё равно audit-only; правки только если отдельно попросили вне этого скила.
