# Lifecycle и submit

Читай, когда нужны default values, create/edit, reset, dirty/touched, leave-guard, submit, массивы или защита от повторной отправки. Поля и a11y — [fields-and-accessibility.md](fields-and-accessibility.md). Ошибки сервера на поле — [validation-and-types.md](validation-and-types.md).

## 1. Default values, hydration, reset

Create: `defaultValues` (или Formik `initialValues`) задают пустую форму той же формы, что schema. Не оставляй `undefined` там, где контрол controlled и ждёт `''` / `[]` / `false` — смотри соседей зоны.

Edit: гидрация из сущности в **form values**, не сырой DTO в проп каждого инпута. Маппинг связей и nullable-значений бери у ближайшей edit-формы зоны, не из чужого DTO.

Когда сущность догрузилась после mount — `reset(mappedValues)`, не молчаливый второй `useForm`. После успешного submit — `reset` с новыми значениями, если соседние формы так сбрасывают dirty (иначе leave-guard сработает на уже сохранённое).

`touched` / `dirty` не подменяй самодельным `hasChanged`, если трек уже даёт `formState.isDirty` / Formik `dirty`.

## 2. Leave-guard

Если в зоне есть примитив — переиспользуй, не пиши второй `useBlocker`.

Типичное условие — `isDirty && !isSubmitted && !isSubmitting`, но точные флаги и переходы бери у ближайшей формы зоны.

Не блокируй уход, когда мутация ещё идёт и пользователь уже подтвердил save — отсюда исключение `isSubmitting` / `isSubmitted`. Не ставь guard на экраны без dirty-смысла (логин), если соседи его там нет.

## 3. Настоящий submit: `<form>` + `type="submit"`

Форма отправляется нативным submit: Enter в текстовом поле и клик по кнопке срабатывают одинаково.

- Один (или явно связанный) `<form onSubmit={handleSubmit(onValid)}>` / Formik `<Form>` / native `onSubmit` с `preventDefault`.
- Кнопка сохранения: `type="submit"` (для `<button>` это дефолт; на UI-ките часто дефолт `button` — **ставь явно**).
- Не подменяй это единственным `onClick={handleSubmit(fn)}` на `Button` вне `<form>`.

Если кнопка визуально в header **вне** полей (админ-экраны) — всё равно:

1. оберни header + поля в один `<form>`, или
2. дай форме `id` и кнопке `form={thatId}` + `type="submit"`.

Enter обязан работать. Клик по header-кнопке — тот же `onSubmit`, не второй обход.

Если ближайшая форма отправляется только через `Button.onClick`, не копируй этот фрагмент как good path: стек/виджеты можно переиспользовать, а семантику submit нужно сохранить через `<form>`.

Framework-managed submit (`action`, route/server action, `useActionState`) тоже должен иметь один канонический путь отправки. Используй pending/error API уже выбранного framework-трека; не вызывай ту же action ещё раз из `onClick` и не создавай параллельную client mutation без необходимости.

## 4. Double-submit, async mutation, success/error

На время запроса кнопка `disabled={isSubmitting || isLoading}` (или эквивалент зоны). Для операций, где повтор особенно опасен, опирайся также на синхронный in-flight guard и/или серверную идемпотентность: обновление React-state и `disabled` само по себе не гарантирует блокировку двух событий в одном tick.

Не вызывай мутацию и `handleSubmit` дважды (ещё и из `onClick`, и из `onSubmit`). Саму mutation и нормализацию HTTP-ошибки не описывай здесь — [`frontend-api`](../../frontend-api/SKILL.md). Этот скил вызывает уже существующую mutation на submit, держит pending и применяет field errors: общая ошибка — toast/status формы; ошибка поля — `setError` / `setFieldError` ([validation-and-types.md](validation-and-types.md)).

Успех: редирект, toast success, `reset` — как у соседней мутации той же зоны, не новый глобальный event-bus. Значения полей, пароли и PII не логируй.

## 5. Arrays и nested fields

| Трек | API списка |
| --- | --- |
| RHF | `useFieldArray`; ключ в map — `field.id` (internal id RHF), не индекс |
| Formik | `FieldArray`; ключ — stable id сущности или helper, не индекс |
| native | массив в state; ключ — stable id, при добавлении генерируй id, не `key={index}` |
| Другая library / framework-managed | array API выбранного трека или локальный immutable update; ключ — stable id, не индекс; не подмешивай `useFieldArray`/`FieldArray` |

Индекс допустим в **path** (`taskStructures.${index}.title`), не как React `key` и не как identity строки, которую пользователь переставляет.

Вложенные объекты — те же `name`/`name="a.b"` / Formik `name="a.b"`, без параллельного state на вложенность.

## 6. Производительность `watch` / rerenders

- `watch()` без аргументов подписан на всю форму — не ставь его в крупный экран без нужды.
- Нужно одно поле — `watch('title')` / `useWatch({ name: 'title' })` / Formik `values.title` в узком компоненте.
- Тяжёлые строки списка выноси в компонент, подписанный на свой index/id.
- Не копируй form values в Redux/Zustand «чтобы меньше ререндерить» — это другой владелец state (`frontend-architecture`).

## 7. Файл / картинка в lifecycle

Preview — локальный `useState`. Submit payload — из form state. Схема знает поле файла/строки, как у соседей. После remove — `setValue` в пустое/`null` **и** сброс preview, иначе dirty/схема разъедутся.

Не храни копию всех полей в `useState` рядом с `useForm` / Formik только ради картинки.

## 8. Conditional fields и multi-step

Условно скрытое поле требует явного решения по контракту:

- если скрытое значение не должно отправляться — unregister/remove/очистка через API выбранного трека;
- если значение сохраняется для возврата к условию или предыдущему шагу — не очищай его случайно, но исключи из DTO, если API его не принимает;
- error/touched скрытого поля не должны блокировать submit невидимой ошибкой;
- при смене discriminator пересобери зависимые значения и validation shape согласованно.

Multi-step форма остаётся одним согласованным form state либо использует уже принятый в зоне draft-механизм. Переход «назад» не теряет допустимые значения; переход «дальше» валидирует текущий шаг, а финальный submit — весь обязательный payload. Не добавляй глобальный store только ради wizard, если локальный трек уже умеет хранить шаги.
