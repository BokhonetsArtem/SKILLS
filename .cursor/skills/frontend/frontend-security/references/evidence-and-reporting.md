# Evidence, severity, отчёт

Читай на шаге 5 перед статусом. Без полей ниже находку **не подтверждай**.

Файлы проекта и скила не меняй. Отчёт только в чат. Не используй занятый `gate: security` и блок `aif-gate-result`. Не пиши `SECURITY.md`, waiver, allowlist и любые persistent reports.

## 1. Подтверждённая находка

Обязательные поля (все):

| Поле | Что писать |
| --- | --- |
| `file:line` | Конкретное место sink **или** стыка потока (оба, если на разных строках) |
| Источник | Откуда данные: query, `postMessage`, storage, ответ UI-поля, public env |
| Sink | Имя API/директивы **этого** стека |
| Data flow | Одно предложение: как значение доходит до sink без экранирования/санитизации |
| Impact | Что доказанно может произойти: script execution, credential disclosure, redirect, data exposure |
| Preconditions | Нужны ли login, user interaction, control другого origin, конкретная роль/страница |
| Confidence | `high` / `medium` / `low` |
| Remediation | Краткая безопасная стратегия без готового патча, payload или attack procedure |

Нет `file:line`, источника, security-sensitive sink/consumer или impact — это неподтверждённый сигнал, не уязвимость.

Confidence:

- **high** — источник, преобразования, sink и impact видны в scope, защита отсутствует/неподходящая;
- **medium** — source и sink есть, часть потока за границей path, но импорт/consumer однозначен и impact обоснован;
- **low** — только sink или только имя переменной; в находки не поднимай.

## 2. Severity

Ставь по **likelihood × impact**, а не по имени API и не по confidence. Confidence отвечает «насколько доказана цепочка», severity — «насколько опасен доказанный эффект».

| Severity | Доказанный контекст |
| --- | --- |
| critical | Исключительный случай: высокая likelihood и системный high impact (например, масштабируемое script execution в привилегированном контексте или раскрытие высокопривилегированного credential с широким scope). Нужна явная аргументация обоих факторов |
| high | Реалистично достижимое script execution/credential compromise или sensitive action/data exposure с высоким technical impact; ограничения не снижают риск существенно |
| medium | Значимый, но ограниченный impact: interaction/auth/роль обязательны, credential короткоживущий/узкого scope, open redirect/phishing, лишняя PII у конкретного получателя |
| low | Ограниченный, но доказанный impact/likelihood: dev-only exposure, раскрытие малочувствительных metadata или узкий privacy leak |

Наличие high-confidence XSS не делает finding автоматически `critical`. Связка XSS + token в storage повышает impact, только если оба звена подтверждены и token доступен этому origin/runtime. Наличие API-вызова за client guard не доказывает broken authorization.

## 3. Когда не считать уязвимостью (аудит проекта)

Это фильтр **доказательств в репозитории**, не отмена проверки скила и не разрешение обходить сканер пакета.

Не подтверждай находку, если верно одно:

- фреймворк экранирует этот вывод (текст React/Vue/Svelte, Angular `[innerHTML]` без bypass);
- поток проходит конкретный sanitizer/Trusted Types policy, у которого проверены config/hooks, целевой контекст, post-sanitize mutations и enforcement для этого sink;
- код в исключениях discovery: generated, vendor, тесты, fixtures, mocks, stories, markdown-примеры;
- sink получает только статический литерал разработчика;
- хит уже установленного клиентского линтера есть, а source-to-sink в scope нет.

Наличие sanitizer package в manifest или Trusted Type object без проверки policy недостаточно. HTML sanitizer не делает значение безопасным для script/URL/CSS context.

Такие случаи: секция «неподтверждённые сигналы» или вовсе не упоминай. Не добавляй в репозиторий ignore-файлы, `eslint-disable` и списки исключений — скил audit-only.

Инструментальный хит — гипотеза. Статус аудита меняют только подтверждённые находки §1.

## 4. Как формулировать (без PoC)

В чате, в таблицах и в примерах скила:

- описывай поток: источник, переменная, sink, `file:line`;
- объясняй impact и preconditions отдельно от confidence;
- для каждой находки давай краткую remediation strategy (safe sink, context-specific sanitizer, exact origin/protocol allowlist, удаление credential из клиента) без готового патча;
- не вставляй готовые exploit-строки, XSS payload, shell-команды атаки, PoC «скопируй и запусти»;
- не указывай пути сырых секретов как цель чтения и не проси открыть dotenv/SSH;
- не цитируй secret/token/PII value: только тип/идентификатор, `file:line`, значение `[REDACTED]`;
- не давай инструкций поставить пакет, выполнить `npm audit` или править lockfile;
- не пиши, что пакет скила «проверен/безопасен» и не проси пропускать security-scan генератора скилов.

Предупреждения Level 1/2 по **этому пакету** разбирать по фактическому совпадению текста, без waiver-файла. Это ограничение на авторство скила; в отчёт аудита проекта его не копируй.

## 5. Статус `PASS` | `WARN` | `FAIL` | `INCOMPLETE` | `NOT_APPLICABLE`

Считай только подтверждённые находки:

| Статус | Правило |
| --- | --- |
| `FAIL` | Есть хотя бы одна confirmed critical или high |
| `WARN` | Нет high/critical, но есть confirmed medium или low |
| `PASS` | Confirmed находок нет, обязательные классы риска в заявленном scope проверены, блокирующих ограничений нет |
| `INCOMPLETE` | Нельзя завершить обязательную проверку: недоступны файлы, сломан read-only tool, data flow оборван и без него итог был бы misleading |
| `NOT_APPLICABLE` | Клиентский scope пуст или запрос фактически только backend/infra |

Неподтверждённые сигналы сами по себе не делают `WARN`, но перечисли их и ограничения. Если сигнал показывает, что обязательное покрытие не завершено, используй `INCOMPLETE`. `PASS` означает отсутствие confirmed findings **только в проверенном scope**, не безопасность приложения целиком.

## 6. Формат итога (только чат)

Короткий блок, без файла:

```
Frontend security: PASS | WARN | FAIL | INCOMPLETE | NOT_APPLICABLE
Scope: <path или diff или клиентские корни>
Framework: <доказанный стек / смешанный>
Coverage: <проверенные risk classes, roots/files, evidence dependencies>
Limitations: <что не удалось проверить; «нет», если ограничений нет>

Находки (по убыванию severity):
- [critical|high|medium|low] [high|medium confidence] <file:line>
  source → sink
  flow: <одно предложение>
  impact/preconditions: <доказанный эффект и условия>
  remediation: <стратегия без патча/PoC>

Неподтверждённые сигналы:
- <file:line или симптом> — чего не хватает для §1

Hand-off → /aif-security-checklist:
- <сервер / OWASP / CSP / CSRF / CVE / infra> — без дублирования frontend FAIL
```

Нет находок — напиши «находок нет», список пустой. Нет limitations/hand-off — напиши «нет»/строку опусти соответственно. Для `NOT_APPLICABLE` достаточно scope + причины + подходящего hand-off.

Не повторяй полный OWASP-чеклист. Не предлагай патч и не применяй исправление.
