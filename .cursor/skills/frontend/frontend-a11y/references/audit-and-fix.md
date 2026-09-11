# Аудит, evidence и fix

Читай на шаге 4–5 перед статусом. Без полей ниже находку **не подтверждай**. Поля security (`source` / `sink` / exploit / data flow) **не копируй** из [`evidence-and-reporting.md`](../../frontend-security/references/evidence-and-reporting.md).

Файлы в `audit` не меняй. Отчёт только в чат. Не пиши `A11Y.md`, waiver, evaluation-scenarios и любые persistent reports. Runtime-логи и DOM/PII в чат не выводи: `[REDACTED]`.

## 1. Подтверждённая находка (a11y-evidence)

Обязательные поля (все):

| Поле | Что писать |
| --- | --- |
| `file:line` | Узел с проблемой (и зависимость-примитив, если имя/роль читаются оттуда) |
| Роль | Фактическая роль платформы или её отсутствие (`button`, `link`, нет роли, `generic`) |
| Источник имени | Как AT возьмёт имя: текст, `label`, `aria-labelledby`, `aria-label`, `alt`, нет; значение при PII — `[REDACTED]` |
| Клавиатурный путь | Tab / Enter / Space / Escape / стрелки: что доказано кодом, чего нет |
| AT impact | Что не сможет пользователь AT или клавиатуры: не активирует, не узнает имя, не услышит статус, застрянет в overlay |
| Confidence | `high` / `medium` / `low` |
| Severity | По §3, не по имени атрибута |

Нет `file:line`, роли/её отсутствия, impact и хотя бы имени **или** keyboard path — это неподтверждённый сигнал, не finding.

Confidence:

- **high** — узел, поведение и отсутствие имени/клавиатуры/ловушки фокуса видны в scope;
- **medium** — узел в scope, часть контракта в импортированном примитиве за path, но импорт однозначен;
- **low** — только lint-хит или только «подозрительный `div`» без обработчика; в находки не поднимай.

## 2. Lint и инструмент — гипотеза

Предупреждение `eslint-plugin-jsx-a11y` / vue-a11y / axe (если гоняли по контракту [discovery-and-scope.md](discovery-and-scope.md) §6) само по себе **не** confirmed finding.

Подними в находки, только если в коде подтверждены поля §1. Ложные типичные гипотезы:

- `click-events-have-key-events` на контейнере, который делегирует в настоящий `<button>`;
- `no-autofocus` на диалоге, где initial focus — контракт модалки;
- contrast-хит без проверяемого цвета/токена;
- `img-has-alt` на generated/stories (исключения discovery).

Хит есть, контекста нет — секция «неподтверждённые сигналы». Статус аудита меняют только confirmed findings.

## 3. Severity

Ставь по **тяжести барьера × насколько он неизбежен**, не по номеру WCAG и не по confidence.

| Severity | Доказанный барьер |
| --- | --- |
| high | Нельзя выполнить основное действие с клавиатуры / AT: кликабельный `div` без альтернативы, модалка без фокуса и Escape, пустое имя у единственного контрола действия, ловушка фокуса |
| medium | Действие возможно обходным путём, но AT теряет структуру или состояние: нет `h1`, disclosure без `aria-expanded`, статус без live region, таблица из `div`, DOM≠visual у части контролов |
| low | Узкий дефект: слабый `title` вместо имени при видимом тексте рядом, лишний Tab, декоративный `alt="icon"` при скрытой иконке |

Нет `critical` как у XSS: a11y-барьер не приравнивай к script execution. Не поднимай severity из-за «звучит как WCAG 1.4.3», если контраст не измерен.

## 4. `INCOMPLETE` и непроверяемое

Неподтверждённый сигнал ≠ выдуманный `FAIL`. Непроверяемое (контраст / SR / motion / сломанный инструмент / оборванный path) пиши в `Limitations` и секцию `INCOMPLETE` — это не замена итогового `FAIL`/`WARN`/`PASS`. Итоговый `INCOMPLETE` — только если обязательную проверку нельзя закрыть **и** confirmed findings нет.

Всегда секция ограничений / неподтверждённый сигнал, не `FAIL` из этого одного факта:

- контраст «на глаз» из CSS без токена и без измеримого инструмента зоны;
- «как это слышит настоящий screen reader» — реальный VoiceOver/NVDA не гоняй как доказательство;
- восприятие motion без `prefers-reduced-motion` / токена / инструмента зоны и без явных keyframes в коде;
- сломанный или не-read-only инструмент; недоступные файлы;
- оборванный обязательный keyboard path (нельзя проследить Tab/Enter до действия и нельзя честно сказать, что барьера нет).

`PASS` — нет confirmed findings **в проверенном scope**, не «весь продукт доступен». `FAIL` — есть confirmed high или несколько medium, которые закрывают сценарий. `WARN` — confirmed только medium/low. `NOT_APPLICABLE` — нет клиентского UI / нет scope.

## 5. Формат итога (`audit`, только чат)

```
Frontend a11y: PASS | WARN | FAIL | INCOMPLETE | NOT_APPLICABLE
Режим: audit
Scope: <path или diff или экран>
Framework / UI-kit: <доказанный стек>
Tooling: <что гоняли read-only / вручную и почему>
Coverage: <категории, корни, evidence dependencies>
Limitations: <непроверяемый контраст/SR/motion; «нет», если ограничений нет>

Находки (по убыванию severity):
- [high|medium|low] [high|medium confidence] <file:line>
  роль: <...>
  имя: <источник или «нет»; значение [REDACTED] при PII>
  клавиатура: <путь или «нет»>
  AT impact: <барьер>
  рекомендация: <стратегия без патча>

Неподтверждённые сигналы:
- <file:line или lint-хит> — чего не хватает для §1

INCOMPLETE:
- <что нельзя проверить>

Hand-off → frontend-security:
- <unsanitized HTML в имени / live region / preview>

Hand-off → forms / layout / routing:
- <реализация поля / токена / route API, если нужна не a11y-дельта>
```

Нет находок — напиши «находок нет». Не предлагай патч в `audit` и не применяй его.

## 6. Безопасный порядок `fix`

Править только confirmed findings в заявленном scope. После правок **повторно проверь тот же scope** (категории, которые трогал).

Порядок, не наоборот:

1. **Семантика** — `<button>` / `<a href>` / `<main>` / `<table>` вместо ложной роли.
2. **Имя** — видимый текст, `label`, `aria-labelledby`; `aria-label` если текста быть не должно (иконка). Через i18n, если i18n уже есть.
3. **Клавиатура и фокус** — Tab/Enter/Space/Escape, trap/restore, порядок = DOM.
4. **ARIA** — только дырки, которые нативное не закрыло (`aria-expanded`, `aria-controls`, `aria-current`).
5. **Анонсы** — `role="status"` / announcer зоны, не assertive-спам.

Переиспользуй `Button`, `FormField`, `Dialog` зоны. Не заводи параллельный кит. Не ставь `role="button"` на `div`. XSS-находку не «закрывай» `innerHTML` — [`frontend-security`](../../frontend-security/SKILL.md).

В `fix` не выходи за scope, не рефакторь соседние экраны, не ставь axe/pa11y/eslint-плагин. Существующий lint/typecheck по затронутой области — без `--fix`, если цель только проверка.

Итог `fix` в чат: что подтвердили, что поменяли (`file`), результат повторной проверки, оставшиеся `INCOMPLETE`. Persistent отчёт не создавай.

## 7. PII и запрет полей security

В evidence и в чате:

- цитируй тип контрола и `file:line`, не значение email/имени/токена;
- не копируй DOM-текст с PII;
- не пиши source → sink, payload, exploit, PoC.

Предупреждение инструмента без §1 не меняет статус на `FAIL`.
