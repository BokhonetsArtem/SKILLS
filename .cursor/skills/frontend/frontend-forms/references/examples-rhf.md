# Примеры: React Hook Form

Читай **только этот файл**, если трек зоны — RHF. `examples-formik.md` и `examples-native.md` не открывай.

Сниппеты — обобщённые рецепты. Не копируй `@/`, имена слайсов и UI-кит как канон чужого репо: импорты, `Input`/`Button` и i18n бери у **соседей этой зоны**. Runtime-логи и значения полей не добавляй.

Правила стека, schema, a11y и submit — в соседних файлах этой папки, не дублируй их сюда целиком.

Как читать пары: сверху **надо**, ниже **не надо**, затем почему. Копируй только приёмы своего стека.

## 1. Простая форма

Нативный `<form>`, `type="submit"`, label связан с `id` контрола, `ref` доходит до input. Form state — в RHF, не в Redux/Zustand.

```tsx
// надо
type ThingFormValues = { title: string };

export function ThingForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<ThingFormValues>({
    defaultValues: { title: '' },
    shouldFocusError: true,
  });

  const generatedId = useId();
  const titleId = `thing-${generatedId}-title`;
  const titleErrorId = `${titleId}-error`;
  const { ref, ...titleField } = register('title');

  return (
    <form onSubmit={handleSubmit(onCreate)}>
      <label htmlFor={titleId}>Title</label>
      <input
        id={titleId}
        {...titleField}
        ref={ref}
        aria-invalid={Boolean(errors.title)}
        aria-describedby={errors.title ? titleErrorId : undefined}
      />
      {errors.title ? <p id={titleErrorId}>{errors.title.message}</p> : null}
      <button type="submit" disabled={isSubmitting}>
        Save
      </button>
    </form>
  );
}
```

`register` можно спреднуть целиком (`{...register('title')}`), если виджет прокидывает `ref` в DOM-input. Не выкидывай `ref`.

Если кнопка визуально в header вне полей — один `<form id={formId}>` на экран **или** `form={formId}` + `type="submit"` на кнопке. `formId` должен быть уникальным (`useId`/prop). Enter и клик — тот же `onSubmit`.

```tsx
// не надо — обход form, потерянный ref, htmlFor={name}, form state в store, смесь с Formik
const title = useAppSelector((s) => s.thingForm.title);

function ThingForm() {
  const { handleSubmit, control } = useForm();
  const { field } = useController({ name: 'title', control });
  const { ref, ...fieldProps } = field; // ref выброшен

  return (
    <>
      <Field name="title" /> {/* Formik в RHF-форме */}
      <label htmlFor="title">Title</label>
      <input {...fieldProps} value={title} />
      <Button onClick={handleSubmit(onCreate)}>Save</Button>
    </>
  );
}
```

**Почему.** Enter не отправит форму без `<form onSubmit>`. `htmlFor={name}` не равен `id` контрола (`title` — имя поля, не DOM id). Без `ref` штатный `shouldFocusError` и явный `setFocus` не попадут в input. Глобальный store дублирует form state. `Field` Formik в RHF-форме — вторая библиотека.

**Как исправить.** Значения — в `useForm`. Разметка — `<form onSubmit={handleSubmit(...)}>` и `type="submit"`. Label — `htmlFor={стабильныйId}` + тот же `id` на input. `{...field}` целиком или `ref` в input. Formik не подмешивай.

Edit: `defaultValues` из сущности в **form values**, не сырой DTO в проп каждого инпута. После догрузки — `reset(mapped)`, не второй `useForm`.

## 2. Schema validation

Одна schema на form values. Resolver — уже стоящий в зоне (`yupResolver` / `zodResolver`). Не дублируй required руками в `onSubmit`.

```tsx
// надо — схема и resolver как у соседей зоны
const thingSchema = yup.object({
  title: yup.string().required(t('validation.required')),
  email: yup.string().email(t('validation.email')).required(t('validation.required')),
});

type ThingFormValues = yup.InferType<typeof thingSchema>;

const form = useForm<ThingFormValues>({
  resolver: yupResolver(thingSchema),
  defaultValues: { title: '', email: '' },
});
```

Если в зоне i18n — ключи как у соседних schema, не хардкод `'Обязательное поле'` и не внедрение i18n «заодно». Нет i18n — строки как у соседей.

В зоне zod — `zodResolver` + `z.infer`, не ставь yup только из-за чужого примера.

```tsx
// не надо — Zod поверх yup-зоны; ручные if рядом с resolver
const form = useForm({ resolver: yupResolver(thingYupSchema) });

async function onSubmit(values: ThingFormValues) {
  if (!values.title.trim()) return; // уже в схеме
  if (!z.string().email().safeParse(values.email).success) return;
  await createThing(values);
}
```

**Почему.** Две схемы расходятся. Ручные `if` в submit дублируют resolver: клиентская проверка уже отработала. Zod в yup-проекте — вторая schema library без запроса.

**Как исправить.** Одна schema, один resolver. Submit маппит form values → DTO, не перепроверяет required. Client validation — не защита сервера.

## 3. Async submit

Мутация только из `onSubmit`. На время запроса кнопка `disabled`. Общая ошибка (сеть, 500) — toast/status формы. Ошибка поля — §4, не вместо него.

```tsx
// надо
async function onSubmit(values: ThingFormValues) {
  try {
    await createThing(toDto(values));
    reset(values);
  } catch (error) {
    const fieldResult = applyFieldErrors(error, setError);
    if (!fieldResult.allHandled) showRequestError(error);
    if (fieldResult.handledCount > 0) return;
  }
}

const formId = useId();

<form id={formId} onSubmit={handleSubmit(onSubmit)}>
  {/* поля */}
</form>
<button type="submit" form={formId} disabled={isSubmitting}>
  Save
</button>
```

`isSubmitting` у RHF true, пока `onSubmit` не завершил promise. Не вызывай мутацию ещё и из `onClick`.

```tsx
// не надо — обход handleSubmit; двойной вызов; клик вместо form
function ThingHeader() {
  const { handleSubmit } = useFormContext();
  return <Button onClick={handleSubmit(onCreate)}>Save</Button>;
}

function ThingFields() {
  return (
    <form onSubmit={handleSubmit(onCreate)}>
      <input {...register('title')} />
    </form>
  );
}
```

**Почему.** Кнопка вне `<form>` ломает Enter. Если и header `onClick={handleSubmit}`, и `<form onSubmit>` — мутация может уйти дважды.

**Как исправить.** Один `handleSubmit(onSubmit)` на `<form>`. Кнопка в header — `type="submit"` и тот же `form` id.

## 4. Server field errors — `setError` обязателен

Ошибка конкретного поля («email занят») должна оказаться **на поле**. Соседний toast-only обработчик — не good path, если API явно указал поле.

```tsx
// надо
function isThingField(field: string): field is keyof ThingFormValues {
  return field === 'title' || field === 'email';
}

function applyFieldErrors(
  error: unknown,
  setError: UseFormSetError<ThingFormValues>,
): { handledCount: number; allHandled: boolean } {
  const items = parseFieldErrors(error); // форма тела 400/409 — как у API зоны
  if (!items?.length) return { handledCount: 0, allHandled: false };
  let handledCount = 0;

  for (const item of items) {
    if (isThingField(item.field)) {
      setError(item.field, { type: 'server', message: item.message });
      handledCount += 1;
    }
  }
  return { handledCount, allHandled: handledCount === items.length };
}

async function onSubmit(values: ThingFormValues) {
  try {
    await createThing(toDto(values));
  } catch (error) {
    const fieldResult = applyFieldErrors(error, setError);
    if (!fieldResult.allHandled) showRequestError(error);
    if (fieldResult.handledCount > 0) return;
  }
}
```

Покажи текст ошибки у поля (`errors.email.message` + `aria-describedby`). Сброс — ввод / `clearErrors` / новый submit. Если часть или все поля ответа неизвестны, дополнительно покажи общую ошибку формы.

```tsx
// не надо — только toast; ручной словарь рядом с RHF; нет setError
async function onSubmit(values: ThingFormValues) {
  try {
    await createThing(values);
  } catch (error) {
    toast.error(error.message);
    setLocalError(error.message);
  }
}
```

**Почему.** AT и пользователь не видят, **какое** поле отклонил сервер. Параллельный `useState` для ошибок дублирует `formState.errors`.

**Как исправить.** Разобрать тело ошибки по контракту зоны. Известные поля — `setError('field', { type: 'server', message })`. Toast — для 500/сети.

## 5. Динамический список

`useFieldArray`. React `key` — `field.id` (internal id RHF), не индекс. Индекс допустим только в **path** (`items.${index}.title`).

```tsx
// надо
const { fields, append, remove } = useFieldArray({ control, name: 'items' });

{
  fields.map((field, index) => (
    <div key={field.id}>
      <input
        id={`item-${field.id}-title`}
        {...register(`items.${index}.title`)}
      />
      <button type="button" onClick={() => remove(index)}>
        Remove
      </button>
    </div>
  ));
}

<button type="button" onClick={() => append({ title: '' })}>
  Add
</button>
```

```tsx
// не надо — key={index}; параллельный useState; Formik FieldArray в RHF
const [rows, setRows] = useState([{ title: '' }]);

{
  rows.map((row, index) => (
    <FieldArray name="items" key={index}>
      <input value={row.title} onChange={(e) => updateRow(index, e.target.value)} />
    </FieldArray>
  ));
}
```

**Почему.** Индекс как `key` ломает identity при перестановке/удалении. Параллельный `useState` расходится с RHF. `FieldArray` — API Formik, не этого трека.

**Как исправить.** Один источник — `useFieldArray`. `field.id` — внутренний ключ RHF и используется как React `key`. Если DTO нужен собственный id, назови его отдельно (`entityId`) и не полагайся на внутренний `field.id`; для новой строки доменный id генерируй только когда этого требует контракт.
