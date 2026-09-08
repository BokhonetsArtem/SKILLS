---
name: frontend-forms
description: Собирает поведение React-форм по стеку репозитория — form state, validation, field binding, submit/reset/errors, dynamic fields и form a11y. Use when form state, validation, валидация полей, field binding, submit, reset, ошибки полей, динамические поля или form-specific accessibility. Не для вёрстки, CSS, Figma (frontend-layout), размещения слайса или schema (frontend-architecture), endpoint/error normalization и вызова mutation (frontend-api), route tree/navigation/URL-state (frontend-routing) и source-to-sink XSS (frontend-security). Не ставит вторую form/schema library и не перехватывает визуальную сетку полей без form state.
disable-model-invocation: false
argument-hint: "[форма | валидация | submit]"
---

# Frontend forms

Собирай **поведение** React-формы по **уже доминирующему стеку этого репозитория**: form state, validation, field binding, submit/reset/errors, dynamic fields, form-specific accessibility. Сначала определи трек (React Hook Form, Formik, native/framework-managed React или другая уже установленная библиотека), затем читай только релевантные references. Не выбирай библиотеку «по умолчанию» и не ставь вторую form/schema library «для удобства».

Не ограничивай себе инструменты. Для этой работы нужны доступные инструменты чтения, поиска, правки файлов и запуска уже существующих в репо проверок. Узкий read-only набор здесь не подходит. Runtime-логи и вывод значений полей не добавляй.

Правила стека, валидации, полей и примеры **не копируй в этот файл**. Читай только нужный файл из `references/` (пути относительно этой папки скила). Примеры — **только выбранного трека**, не все `examples-*` сразу.

**Не этот скил.** Вёрстка, CSS, токены, Flex/Grid, макет Figma, визуальная сетка полей без form state, фильтры и поиск **без** записи в URL маршрута — [`frontend-layout`](../frontend-layout/SKILL.md). Запрос вроде «сверстай форму» / «по макету» без validation/submit/field binding — тоже layout, этот скил дальше не веди. Куда класть файлы формы, schema и кто владеет form state — [`frontend-architecture`](../frontend-architecture/SKILL.md). Endpoint, query/mutation, нормализация ошибок HTTP — [`frontend-api`](../frontend-api/SKILL.md); этот скил применяет уже нормализованные field errors и ведёт form lifecycle. Route tree, client redirect и URL-state маршрута — [`frontend-routing`](../frontend-routing/SKILL.md); leave-guard грязной формы остаётся здесь. XSS, unsafe preview, секреты в клиенте — [`frontend-security`](../frontend-security/SKILL.md), не повод начинать аудит из обычной валидации.

Маршруты (формулировка запроса → скил):

- «сверстай форму / по макету» → [`frontend-layout`](../frontend-layout/SKILL.md)
- «куда положить форму/schema» → [`frontend-architecture`](../frontend-architecture/SKILL.md)
- «endpoint / mutation / нормализация ошибок» → [`frontend-api`](../frontend-api/SKILL.md)
- «валидация / submit / array fields» → этот скил
- «route tree / URL-state / client redirect» → [`frontend-routing`](../frontend-routing/SKILL.md)
- «XSS / секреты / unsafe preview» → [`frontend-security`](../frontend-security/SKILL.md)

Разведение: architecture — размещение и ownership; API — транспорт, mutation и нормализация ошибок; routing — декларация маршрута и URL-state; этот скил — поведение формы и применение field errors; layout — визуальная оболочка; security — отдельный source-to-sink аудит. Если задача сразу про **новый слайс и форму** — сначала размещение (architecture), затем HTTP-контракт ([`frontend-api`](../frontend-api/SKILL.md)), если нужен endpoint/mutation, затем поведение (здесь), затем оболочка (layout), если она нужна. Шаги не смешивай.

**Режим** (не грузи лишнее):

| Задача | Что читать |
| --- | --- |
| Стек неясен / смешанный / какой трек | [references/discovery-and-stack.md](references/discovery-and-stack.md) |
| Schema, типы, sync/async validation, field-level server errors | [references/validation-and-types.md](references/validation-and-types.md) |
| Labels, `htmlFor`/`id`, `aria-*`, focus ошибки, виджеты полей | [references/fields-and-accessibility.md](references/fields-and-accessibility.md) |
| Submit, reset, dirty/touched, leave guard, arrays, double-submit | [references/lifecycle-and-submit.md](references/lifecycle-and-submit.md) |
| Рецепт RHF (только этот трек) | [references/examples-rhf.md](references/examples-rhf.md) |
| Рецепт Formik (только этот трек) | [references/examples-formik.md](references/examples-formik.md) |
| Рецепт native React (только этот трек) | [references/examples-native.md](references/examples-native.md) |
| Другая library / framework-managed / нет examples-файла | [references/discovery-and-stack.md](references/discovery-and-stack.md) + тематический reference + локальные формы/API установленной версии |
| Проверка качества самого скила | [references/evaluation-scenarios.md](references/evaluation-scenarios.md) |

## Шаг 1. Классифицируй задачу

Опора: сообщение пользователя, `$ARGUMENTS` (если скил вызван с аргументом), открытые файлы. В обычном чате `$ARGUMENTS` может быть пустым — это не стоп.

| Сигнал | Режим |
| --- | --- |
| Form state, schema, field binding, submit/reset, ошибки полей, array/nested fields, form a11y | Этот скил |
| Только сетка/оболочка, CSS, Figma, «сверстай форму», фильтры/поиск без form state | [`frontend-layout`](../frontend-layout/SKILL.md) — остановись |
| Куда положить форму, schema, слайс; кто владеет state | [`frontend-architecture`](../frontend-architecture/SKILL.md) — сначала размещение |
| Endpoint, query/mutation, нормализация ошибок HTTP без form binding | [`frontend-api`](../frontend-api/SKILL.md) — не этот скил |
| Route tree, navigation, URL-state без form state | [`frontend-routing`](../frontend-routing/SKILL.md) — остановись |
| XSS / unsafe preview / секреты в поле | [`frontend-security`](../frontend-security/SKILL.md) — не этот скил |

Несколько сигналов сразу — веди по самому широкому: сначала граница и владелец (architecture), потом HTTP-контракт (API), потом поведение формы (здесь), потом визуал (layout).

## Шаг 2. Обнаружь стек формы

Смотри репозиторий, не привычку модели. Не классифицируй трек только по «популярности» RHF.

Что проверить (достаточно сигналов, не полный аудит репо):

- манифест и lockfile: `package.json` — `react-hook-form`, `formik`, `@hookform/resolvers`, `yup`, `zod`;
- импорты в **зоне изменения** (`useForm`, `FormProvider`, `useFormik`, `<Formik>`, `useFieldArray`, `FieldArray`);
- ближайшая форма той же зоны — локальный прецедент важнее «главного» стека репо;
- schema library, которой уже пользуются соседи (yup / zod / resolver), не та, которую «обычно ставят».

| Что видно в зоне | Трек | Что читать |
| --- | --- | --- |
| Доминируют `react-hook-form`, `useForm`, `FormProvider`, `@hookform/resolvers` | RHF | [examples-rhf.md](references/examples-rhf.md); discovery — только если сигналы противоречат друг другу |
| Доминируют `formik`, `useFormik`, `<Formik>`, `Field` / `FieldArray` | Formik | [examples-formik.md](references/examples-formik.md); discovery — только если сигналы противоречат друг другу |
| Нет form library; `useState` / `useReducer` + нативный `<form>` | native | [examples-native.md](references/examples-native.md); discovery — только если сигналы противоречат друг другу |
| Framework управляет submit/state (`action`, server action, `useActionState`, route action) | framework-managed | [discovery-and-stack.md](references/discovery-and-stack.md), затем локальные формы и документация уже используемого framework API; не подменяй native-примером |
| Есть другая form library (`@tanstack/react-form`, Final Form, Conform и т.п.) | существующий трек зоны | [discovery-and-stack.md](references/discovery-and-stack.md) + локальные формы/типы этой библиотеки; RHF/Formik examples не читай |
| В репо несколько библиотек, в этой зоне один прецедент | трек зоны | examples **только** этого трека; чужой не подмешивай |
| Сигналов мало или в зоне два равноправных способа | спроси | не ставь пакет и не смешивай RHF с Formik в одной форме |

Не устанавливай `react-hook-form`, `formik`, `yup`, `zod`, resolver или другую form/schema library, пока пользователь явно не попросил. Schema — следуй уже стоящей: не подставляй Zod в yup-проект и наоборот.

Если библиотека распознана, но отдельного examples-файла для неё нет, это **не** повод переводить форму на RHF/Formik/native. Сохрани её модель state/validation/submit, возьми API из установленной версии и локальных форм, а универсальные требования к семантике, a11y, server errors и lifecycle — из тематических references этого скила.

## Шаг 3. Выбери references и аналог

По типу задачи читай только нужное (progressive disclosure — не грузи все references сразу):

| Задача | Тематический reference сверх уже выбранных файлов режима |
| --- | --- |
| Schema, типы значений, sync/async validation, `setError` / Formik `status`+`errors` | [validation-and-types.md](references/validation-and-types.md) |
| Label, `aria-invalid`, `aria-describedby`, focus первой ошибки, checkbox/radio/select/file | [fields-and-accessibility.md](references/fields-and-accessibility.md) |
| Default values, create/edit hydration, reset, dirty, leave guard, arrays, double-submit | [lifecycle-and-submit.md](references/lifecycle-and-submit.md) |

Паттерн бери у **ближайшей формы той же зоны**, не у первого попавшегося Grep и не из другого репозитория. Если в зоне уже есть локальные form-примитивы поля, ошибок или leave-guard — переиспользуй их, не изобретай параллельные. Имена и импорты — как **здесь**.

Чужие пути вроде `@/entities/skill` не переноси в проект, где их нет. Нет близкого аналога — читай examples выбранного трека. Всё ещё пусто или спорно — спроси, не выдумывай канон.

## Шаг 4. Реализуй поведение формы

По выбранному треку и аналогу:

- значения полей живут в form state выбранной библиотеки (или native state), не дублируй всю форму в параллельный `useState` и не клади form state в глобальный store без прецедента;
- валидация — schema ↔ form values ↔ API DTO; client validation не считать защитой сервера;
- ошибки конкретного поля — через API трека (`setError` / Formik `errors`+`status`), не только toast; нормализацию HTTP-ошибки не делай здесь — [`frontend-api`](../frontend-api/SKILL.md);
- разметка — настоящий `<form onSubmit>` и кнопка `type="submit"` (Enter-submit); не заменяй это одним `Button.onClick={handleSubmit(...)}`, даже если так сделано в соседнем экране;
- динамические списки — API трека (`useFieldArray` / `FieldArray`); в RHF React key — внутренний `field.id`, доменный id хранится отдельно, в остальных треках — stable id/helper, не индекс массива;
- file/image: локальный preview в `useState` допустим, отправляемое значение — в form state через API выбранного трека (`setValue`, `Field`, локальный setter и т.п.).

Если в проекте есть i18n — тексты ошибок через него, не хардкод и не внедрение i18n «заодно». Нет i18n — не внедряй.

Не копируй локальные anti-patterns даже из этого репо: отброшенный controller `ref`, `htmlFor={name}` вместо `id` контрола, подпись не связанная с полем, submit только через `onClick`. Как надо / как не надо — в examples выбранного трека.

После поведения, если нужна визуальная оболочка — передай её в [`frontend-layout`](../frontend-layout/SKILL.md). Куда лежат файлы — уже решено architecture; здесь файлы не переезжают.

## Шаг 5. Чеклист перед сдачей

Если пункт не выполнен — почини, не сдавай. Пункты про конкретный инструмент — только если он **есть** в репозитории.

- **Трек.** Выбран уже доминирующий в зоне; новая form/schema library не установлена без явного запроса. Прочитан только examples своего трека.
- **Семантика.** Нативный `<form onSubmit>`, кнопка `type="submit"`, Enter отправляет форму.
- **a11y.** Label связан с контролом (`htmlFor`+`id`, не `htmlFor={name}`); `aria-invalid` / `aria-describedby`; ошибка объявляется; focus первой ошибки идёт в input (рабочий `ref`); placeholder не заменяет label.
- **Ошибки.** Field-level server errors через form-API трека (`setError` / `setFieldError`), не только toast. Нормализация HTTP-ошибки — [`frontend-api`](../frontend-api/SKILL.md). Тексты — через i18n, если он есть.
- **Типы.** Form values и submit payload типизированы; schema не дублируется ручными проверками без причины.
- **Lifecycle.** Default values / hydration для create и edit; reset и dirty/touched учтены; защита от повторного submit на время async mutation.
- **Списки.** Ключ элемента — stable id, не индекс.
- **Проверки репозитория.** Запусти уже существующие lint / typecheck по затронутой области. Если в репо уже есть тестовый стек — покрой поведение формы существующим способом; тестовый раннер не добавляй.
- **Данные.** Значения полей, пароли и PII не логируй.
- **Условные/многошаговые поля.** Скрытые поля не отправляют stale values без явного контракта; переходы между шагами не теряют допустимые значения и ошибки.

Перед сдачей — коротко, не эссе:

- **Трек:** RHF / Formik / native / framework-managed / другая library (и что спросили у пользователя).
- **Доказательства:** 2–3 факта из репо (зависимость, импорт зоны, ближайшая форма).
- **Schema:** yup / zod / нет / как у соседей.
- **Что не копировали:** если в зоне были anti-patterns submit/a11y/`setError` — что сделали иначе.

## Если что-то пошло не так

- Стек неясен или в зоне два способа → спроси. Не подставляй RHF, Formik или native наугад и не смешивай библиотеки в одной форме.
- Нет аналога → ищи соседние формы **этого** репо; чужие импорты и слайсы не копируй.
- Пользователь не просил новую библиотеку → не добавляй `react-hook-form`, `formik`, `yup`, `zod` или resolver.
- Нужна только визуальная сетка / макет / CSS → это [`frontend-layout`](../frontend-layout/SKILL.md), не этот скил.
- Нужно только размещение слайса или schema → это [`frontend-architecture`](../frontend-architecture/SKILL.md), не этот скил.
- Нужны endpoint, нормализация ошибок HTTP или вызов mutation без form binding → это [`frontend-api`](../frontend-api/SKILL.md), не этот скил.
- Нужен route tree / URL-state / client redirect без form lifecycle → это [`frontend-routing`](../frontend-routing/SKILL.md), не этот скил.
- Нужен XSS / unsafe preview / секреты → это [`frontend-security`](../frontend-security/SKILL.md), не этот скил.
- В зоне submit через `onClick`, отброшенный `ref` или несвязанный label → не воспроизводи; good path в examples трека.
