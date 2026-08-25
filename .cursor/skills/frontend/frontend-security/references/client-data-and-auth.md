# Client data and auth

Читай на шаге 4 после [discovery-and-scope.md](discovery-and-scope.md). Серверную сессию, cookie flags, CSRF сервера и API authorization **не** аудируй — `aif-security-checklist`.

Сырые dotenv, vault, SSH-ключи и credential stores **не открывай**. Ищи попадание секрета в **клиентский исходник** и в публичные префиксы, которые бандлер уже вшивает.

## Public env и секреты в бандле

Секрет — ключ, токен, пароль, private endpoint credential, который по документации провайдера, scope или назначению не должен быть у любого, кто открыл DevTools. Публичный идентификатор по одному имени `key` секретом не становится.

| Что видно в клиентском модуле | Как квалифицировать |
| --- | --- |
| `NEXT_PUBLIC_*` / `VITE_*` / `REACT_APP_*` / `NUXT_PUBLIC_*` / `PUBLIC_*` / `EXPO_PUBLIC_*` с доказанным секретом | Подтверждённая утечка: repo config подтверждает, что значение вшивается в клиент |
| Импорт из `environment*.ts` (Angular), define/runtime config или константа в клиентском графе | Утечка, если bundler подставляет значение и это секрет, а не публичный origin/identifier |
| Строка API key / `sk-` / private PEM / SMTP password в prod client source | Утечка после проверки типа, контекста и provider semantics; не полагайся только на префикс |
| Публичный `BASE_URL`, идентификатор аналитики, публичный Stripe **pk_** | Не секрет; не флагь как credential |

Не открывай сырые dotenv, vault и credential stores. Если секрет только там и **не** импортирован в клиент — здесь не находка; при необходимости hand-off в чеклист (секреты на диске), без чтения файла.

`process.env` без публичного префикса в `'use client'`-модуле не доказывает утечку: Next/Vite и другие bundlers могут заменить значение на `undefined` или не экспортировать его. Подтверди `DefinePlugin`/env config/runtime injection, import graph или уже существующий deploy artifact. Не уверен в подстановке — неподтверждённый сигнал.

В отчёте никогда не копируй значение секрета. Пиши только идентификатор/тип, scope (если виден без раскрытия значения) и `file:line`; значение — `[REDACTED]`.

## Токены: storage и URL

| Место | Когда это находка |
| --- | --- |
| `localStorage` / `sessionStorage` / `IndexedDB` | Access/refresh token, session id или долгоживущий credential доступны любому script того же origin; severity зависит от lifetime/scope и наличия XSS/third-party scripts |
| Query: `token`, `access_token`, `id_token` остаются после логина | История, Referer, analytics, screenshots и логи URL |
| Fragment: token остаётся после логина | Не уходит в HTTP Referer/server logs, но доступен scripts, browser history и screenshots |
| OAuth authorization `code` | Учитывай одноразовость, TTL, PKCE/state и очистку URL; не приравнивай автоматически к access token |
| Поле формы / hidden input с долгоживущим токеном, которое уходит в HTML | Клиентский leak в DOM |
| `document.cookie` запись не-HttpOnly сессии из JS | Клиентская копия сессии; флаги HttpOnly/Secure сервера не проверяй здесь |

Короткоживущий CSRF *request* token в памяти приложения — не access token. Не путай с чеклистом CSRF.

Передача Bearer только в `Authorization` из памяти (переменная модуля, не storage и не URL) — само по себе не FAIL. Для callback URL проверь своевременный `history.replaceState`/эквивалент после обработки. Рядом с подтверждённым XSS учитывай доступность токена, но не выдумывай второе звено.

## Client-only authorization

Проверки в браузере не заменяют сервер, но отсутствие server code в frontend scope **не доказывает** отсутствие API authorization.

Не подтверждай finding только по:

- `Navigate`/router guard/`v-if="isAdmin"`/скрытой кнопке — это может быть UX;
- `jwt-decode`/`atob` для отображения UI — подпись обязан проверять сервер;
- наличию чувствительного API-вызова за client guard — endpoint может корректно вернуть 403.

Подтверждай frontend finding, когда доверие к изменяемому client state само раскрывает уже доставленные чувствительные данные/секрет, включает локально поставляемую privileged capability или формирует подписываемое/доверяемое решение без trusted boundary. Иначе это неподтверждённый сигнал: укажи конкретный endpoint/action для hand-off в `aif-security-checklist`, не утверждай, что API открыт.

Серверные `middleware.ts`, Nest guards, cookie session — не этот файл.

## PII, ошибки, телеметрия

| Sink | Риск |
| --- | --- |
| UI / error boundary / toast | Стек, SQL, внутренний URL, токен или лишние поля объекта, показанные не тому получателю/без необходимости |
| `console.*` в проде с PII или токеном | Утечка в DevTools; в dev-only ветке (`import.meta.env.DEV`) — слабее |
| Клиентский логгер / analytics / replay, которому в payload кладут token, password, полный request | Credential/PII у third-party или лишнего получателя; token/password оценивай выше обычной PII |
| Сообщение валидации с сырым ответом 500 | Внутренности бэкенда на клиенте |

Не добавляй логи и telemetry этим скилом. Только зафиксируй существующий поток.

Легитимный показ владельцу запрошенных данных — не утечка сам по себе. Подтверди лишнего/неавторизованного получателя, чрезмерный payload/DOM, third-party destination, ненужное persistence/retention. Маскированный email в UI — не находка; session token в DOM/analytics — credential exposure.

## Что отдать чеклисту, не дублируя FAIL

- Секреты только в CI/infra, не в клиентском графе.
- Авторизация эндпоинта, refresh на сервере, rotation.
- CORS, CSP headers, cookie `Secure`/`SameSite`.
- CVE зависимостей.

Спорный сигнал без клиентского data flow — неподтверждённый, плюс одна строка hand-off. Не копируй gate чеклиста.
