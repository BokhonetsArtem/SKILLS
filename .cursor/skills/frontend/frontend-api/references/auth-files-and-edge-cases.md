# Auth, файлы и крайние случаи

Читай, когда нужны credentials / auth header / refresh по существующему контракту, upload/download, пустые ответы, 429 или offline. Ошибки мутаций — [mutations-errors-and-effects.md](mutations-errors-and-effects.md). Транспорт — [http-transport-and-contracts.md](http-transport-and-contracts.md). Стек — [discovery-and-boundaries.md](discovery-and-boundaries.md).

Этот файл **не** аудит. Явный source-to-sink разбор token storage / public env / client token flow — [`frontend-security`](../../frontend-security/SKILL.md). Cookie flags, CSRF, server authorization, CORS-политика сервера — [`aif-security-checklist`](../../../aif-security-checklist/SKILL.md).

Не канонизируй `localStorage`, cookie, память или иной storage. Не выбирай и не переноси место хранения токена без отдельного запроса.

## 1. Подключай контракт зоны, не «лучший storage»

Как соседи уже отправляют сессию — так и новый endpoint:

| Уже принято в base client | Что делать |
| --- | --- |
| `credentials: 'include'` / cookie-сессия | Сохрани; не дублируй Bearer «на всякий случай» |
| `Authorization` из существующего helper/interceptor | Тот же helper; не читай storage напрямую в каждом endpoint |
| Header + cookie вместе | Не убирай половину без запроса |
| Нет auth на публичном чтении | Не добавляй token на этот вызов |

Размещение auth/session-модуля — [`frontend-architecture`](../../frontend-architecture/SKILL.md). Этот скил только **подключает** уже существующий контракт к транспорту.

Не делай: перевод cookie-сессии на `localStorage` «как безопаснее/проще»; новый ключ storage рядом со старым; токен в query / fragment / path. Токен в URL — anti-pattern (история, Referer, логи); не копируй, даже если старый код так делает.

Выбор или аудит места хранения — security, не этот файл.

## 2. 401: один refresh, очередь, без цикла

Если зона уже обновляет сессию при 401:

1. **Один** in-flight запрос refresh на все параллельные 401 (single-flight: общий promise/mutex). Не стартуй refresh на каждый вызов.
2. Исходные запросы **ждут** этот refresh и повторяются с обновлённым контрактом.
3. Сам refresh-endpoint при 401 **не** триггерит новый refresh (исключи его из interceptor).
4. Неуспешный refresh → сломанная сессия по уже принятому событию/логауту зоны, не бесконечный цикл 401→refresh→401.
5. 403 не ретраи как 401, если зона их различает.

Нет refresh в зоне — не выдумывай. Повтори соседний 401-handler (редирект на login, событие «сессия сломана»). Не добавляй второй механизм рядом.

Retry исходного **POST** после refresh допустим как повтор той же пользовательской попытки; это не авто-retry из §2 [mutations-errors-and-effects.md](mutations-errors-and-effects.md). Idempotency key, если он уже есть, сохрани на повтор.

Не логируй access/refresh token и тело refresh.

## 3. Upload и download

**Upload.** Тело — `FormData` (или уже принятый binary body зоны). `Content-Type: multipart/form-data` вручную **не** ставь: boundary должен выставить браузер ([http-transport-and-contracts.md](http-transport-and-contracts.md) §6). Прогресс (`onUploadProgress` и аналоги) — только если выбранный client зоны это уже умеет; не подключай второй клиент ради прогресса.

**Download.** Читай как `blob`/`arrayBuffer`. Имя из `Content-Disposition` — недоверенная подсказка: поддержи `filename*`, убери path segments (`/`, `\`), control characters, `.`/`..`, ведущие/замыкающие пробелы и опасные platform names; расширение не должно противоречить ожидаемому media type. Если имя отсутствует или после sanitization пусто — используй нейтральное. Object URL после скачивания **revoke**. Не парси файл как JSON. Не вставляй blob в DOM как HTML — XSS/unsafe preview: [`frontend-security`](../../frontend-security/SKILL.md), не этот скил.

Auth на upload/download — тот же credentials/header, что у JSON-вызовов зоны. Не клади token в query «потому что `<a href>`».

## 4. Пустые ответы

| Сигнал | Клиент |
| --- | --- |
| 204 / 205 | Успех без body; не `json()` |
| 200 + пустое тело | Как у соседа: `void` / `null`, не падение parse |
| `null` JSON | Это тело, не «нет ответа»; тип `Thing \| null` |

Delete и «принять без содержимого» часто 204. Не требуй `{ success: true }`, если контракт пустой.

## 5. 429, сеть, offline

- **429:** если есть `Retry-After`, уважай его; не крути tight-loop retry. Нет политики в зоне — покажи ошибку, не изобретай backoff-библиотеку.
- **Сеть / CORS fail:** транспортная ошибка без HTTP status; не маскируй под «500».
- **Offline:** `navigator.onLine` ненадёжен. Не ставь offline SDK без запроса. Не показывай success, если запрос не ушёл. UI empty/offline — [`frontend-layout`](../../frontend-layout/SKILL.md).

Rate limiting и CORS **сервера** не аудируй здесь.

## 6. Безопасные границы без аудита

Обычная настройка клиента **не** повод открывать security-скил.

| Действие | Этот скил | Не этот скил |
| --- | --- | --- |
| Повесить существующий Bearer/cookie на client | да | — |
| Single-flight refresh по уже принятому endpoint | да | — |
| Решить, cookie vs `localStorage` vs память | нет | [`frontend-security`](../../frontend-security/SKILL.md) |
| HttpOnly / Secure / SameSite, CSRF, API authorization | нет | [`aif-security-checklist`](../../../aif-security-checklist/SKILL.md) |
| Не класть token в URL, не логировать PII | да (гигиена реализации) | аудит существующих утечек — security |

Публичные env (`VITE_*` / `NEXT_PUBLIC_*` и аналоги) в этом скиле — только уже принятый `baseUrl`/origin. Секрет в клиентский бандл не добавляй. Сырые dotenv не открывай.

## 7. Чего не делать

- Канонизировать storage токена из «привычки» или из чужого репо.
- Параллельные refresh без очереди и refresh на сам refresh.
- Token, email-секрет и пароль в query string.
- Ручной multipart `Content-Type`.
- Доверять имени файла из `Content-Disposition` без sanitization.
- `json()` на 204.
- Авто-шторм retry на 429/offline.
- Runtime-логи request/response, token и PII.
- Запуск source-to-sink аудита из задачи «подключи credentials».
