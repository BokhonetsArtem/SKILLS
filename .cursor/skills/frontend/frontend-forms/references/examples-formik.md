# Примеры: Formik

Читай **только этот файл**, если трек зоны — Formik. `examples-rhf.md` и `examples-native.md` не открывай.

Сниппеты — обобщённые рецепты. Не копируй `@/`, имена слайсов и UI-кит как канон. Импорты, виджеты и i18n — как у **соседей этой зоны**. Runtime-логи и значения полей не добавляй.

Правила стека, schema, a11y и submit — в соседних файлах этой папки.

Как читать пары: сверху **надо**, ниже **не надо**, затем почему.

## 1. Простая форма

Нативный submit через Formik `<Form>` (это `<form onSubmit>`). Кнопка `type="submit"`. Label связан с `id` контрола, не с `name` поля. Form state — в Formik, не в Redux/Zustand и не в RHF.

```tsx
// надо
type ThingFormValues = { title: string };

function FocusFirstError({
  titleRef,
}: {
  titleRef: RefObject<HTMLInputElement | null>;
}) {
  const { errors, isSubmitting, submitCount } = useFormikContext<ThingFormValues>();

  useEffect(() => {
    if (submitCount > 0 && !isSubmitting && errors.title) {
      titleRef.current?.focus();
    }
  }, [errors.title, isSubmitting, submitCount, titleRef]);

  return null;
}

export function ThingForm() {
  const titleRef = useRef<HTMLInputElement>(null);
  const generatedId = useId();
  const titleId = `thing-${generatedId}-title`;
  const titleErrorId = `${titleId}-error`;

  return (
    <Formik
      initialValues={{ title: '' } satisfies ThingFormValues}
      onSubmit={onCreate}
    >
      {({ errors, isSubmitting }) => (
        <Form>
          <FocusFirstError titleRef={titleRef} />
          <label htmlFor={titleId}>Title</label>
          <Field
            id={titleId}
            name="title"
            innerRef={titleRef}
            aria-invalid={Boolean(errors.title)}
            aria-describedby={errors.title ? titleErrorId : undefined}
          />
          {errors.title ? <p id={titleErrorId}>{errors.title}</p> : null}
          <button type="submit" disabled={isSubmitting}>
            Save
          </button>
        </Form>
      )}
    </Formik>
  );
}
```

`useFormik` + свой `<form onSubmit={formik.handleSubmit}>` — тот же контракт: `preventDefault` уже внутри `handleSubmit`. Не подменяй это `onClick={formik.submitForm}` без `<form>`.

Если кнопка в header вне полей — оберни header + поля в один `<Form>` **или** дай форме уникальный `id` (`useId`/prop) и кнопке тот же `form={id}` + `type="submit"`.

Focus первой ошибки: сохрани `ref`/`innerRef` на поля и после завершения неуспешной validation вызови `focus()` для первого поля в явном DOM-порядке. Для nested errors не полагайся на сырой `Object.keys(errors)`: используй принятый в зоне ordered field-path list. `innerRef` не выкидывай.

```tsx
// не надо — RHF в Formik-форме; store; htmlFor={name}; submit только кликом
const title = useAppSelector((s) => s.thingForm.title);

function ThingForm() {
  const formik = useFormik({ initialValues: { title }, onSubmit: onCreate });
  const { register } = useForm();

  return (
    <>
      <label htmlFor="title">Title</label>
      <input {...register('title')} value={formik.values.title} />
      <Button onClick={() => formik.submitForm()}>Save</Button>
    </>
  );
}
```

**Почему.** `register` — API RHF, вторая библиотека. Глобальный store дублирует `values`. `htmlFor="title"` не есть `id` контрола. Без `<form>` Enter не отправит форму. `submitForm()` с кнопки обходит нативный submit.

**Как исправить.** Один Formik на форму. `<Form>` или `<form onSubmit={formik.handleSubmit}>`. Label — стабильный `id`. `innerRef` доходит до input. RHF не подмешивай.

Edit: `initialValues` из сущности в **form values**. Когда сущность догрузилась — `enableReinitialize` (если так делают соседи) или `resetForm({ values: mapped })`, не второй `useFormik`.

## 2. Schema validation

Одна schema на form values. Formik: `validationSchema` (yup) **или** уже стоящий в зоне адаптер zod. Не ставь yup в zod-зону и наоборот. Не дублируй required руками в `onSubmit`.

```tsx
// надо — yup, если его уже резолвят соседи зоны
const thingSchema = yup.object({
  title: yup.string().required(t('validation.required')),
  email: yup.string().email(t('validation.email')).required(t('validation.required')),
});

<Formik
  initialValues={{ title: '', email: '' }}
  validationSchema={thingSchema}
  onSubmit={onCreate}
>
```

Если в зоне zod — тот же адаптер, что у соседей (`validate` с `schema.parse`, готовый `toFormikValidationSchema` и т.п.). **Не** добавляй yup «потому что Formik так привыкли».

Если в зоне i18n — сообщения schema через него. Нет i18n — не внедряй.

```tsx
// не надо — схема + ручные if; Zod и yup сразу; yupResolver от RHF
<Formik
  validationSchema={thingYupSchema}
  validate={(values) => {
    const parsed = thingZodSchema.safeParse(values);
    if (!values.title) return { title: 'required' };
    return parsed.success ? {} : { email: 'invalid' };
  }}
  onSubmit={async (values) => {
    if (!values.title.trim()) return;
    await createThing(values);
  }}
/>

const rhf = useForm({ resolver: yupResolver(thingYupSchema) });
```

**Почему.** Две схемы и ручной `if` расходятся с `validationSchema`. `yupResolver` — API RHF. Zod+yup в одной форме — вторая schema library.

**Как исправить.** Один канал: `validationSchema` **или** `validate`, как у соседей. Submit маппит values → DTO. Client validation — не защита сервера.

## 3. Async submit

Мутация только из `onSubmit`. `isSubmitting` дизейблит кнопку. Общая ошибка (сеть, 500) — toast / `setStatus`. Ошибка поля — §4.

```tsx
// надо
async function onCreate(values: ThingFormValues, helpers: FormikHelpers<ThingFormValues>) {
  try {
    await createThing(toDto(values));
    helpers.resetForm({ values });
  } catch (error) {
    const fieldResult = applyFieldErrors(error, helpers);
    if (!fieldResult.allHandled) {
      helpers.setStatus({ form: messageFrom(error) });
      showRequestError(error);
    }
    if (fieldResult.handledCount > 0) return;
  }
}

const formId = useId();

<Form id={formId}>
  {/* поля */}
</Form>
<button type="submit" form={formId} disabled={isSubmitting}>
  Save
</button>
```

Не вызывай `submitForm()` ещё и из `onClick` на той же кнопке, что `type="submit"`.

```tsx
// не надо — обход Form; двойной submit; мутация из onClick
function ThingHeader({ formik }: { formik: FormikProps<ThingFormValues> }) {
  return (
    <Button
      onClick={() => {
        void formik.submitForm();
        void createThing(formik.values);
      }}
    >
      Save
    </Button>
  );
}
```

**Почему.** Кнопка вне `<form>` ломает Enter. Прямой `createThing(formik.values)` обходит валидацию (`handleSubmit` / `<Form>`). Два вызова — двойная мутация.

**Как исправить.** Один `onSubmit`. Кнопка в header — `type="submit"` и `form` id. Значения на сервер — только из Formik `onSubmit`.

## 4. Server field errors — `setFieldError` обязателен

Ошибка конкретного поля должна оказаться **на поле**, не только в toast. Эквивалент RHF `setError` здесь — `setFieldError` (и `setStatus` для общей ошибки). Не подменяй поле одним `setStatus`, если бэкенд указал поле.

```tsx
// надо
function isThingField(field: string): field is keyof ThingFormValues {
  return field === 'title' || field === 'email';
}

function applyFieldErrors(
  error: unknown,
  helpers: FormikHelpers<ThingFormValues>,
): { handledCount: number; allHandled: boolean } {
  const items = parseFieldErrors(error); // форма тела 400/409 — как у API зоны
  if (!items?.length) return { handledCount: 0, allHandled: false };
  let handledCount = 0;

  for (const item of items) {
    if (isThingField(item.field)) {
      helpers.setFieldError(item.field, item.message);
      handledCount += 1;
    }
  }
  helpers.setStatus({ form: undefined });
  return { handledCount, allHandled: handledCount === items.length };
}
```

Текст — `errors.email` + `aria-describedby`. Если часть или все поля ответа неизвестны, дополнительно покажи `setStatus` / общую ошибку.

```tsx
// не надо — только toast; ручной словарь; setStatus вместо поля
async function onCreate(values: ThingFormValues) {
  try {
    await createThing(values);
  } catch (error) {
    toast.error(error.message);
    setLocalError(error.message);
  }
}

// тоже не надо, если ошибка про email:
helpers.setStatus({ form: 'Email already taken' });
```

**Почему.** Пользователь и AT не видят поле. Параллельный `useState` дублирует `errors`. `setStatus` без `setFieldError` прячет field-level ответ.

**Как исправить.** Известные поля — `setFieldError`. Toast / `setStatus` — для 500/сети и неизвестных полей.

## 5. Динамический список

`FieldArray`. React `key` — stable id сущности или helper, не индекс. Индекс допустим в **path** (`items.${index}.title`).

```tsx
// надо
<FieldArray name="items">
  {({ push, remove, form }) => (
    <>
      {form.values.items.map((item, index) => (
        <div key={item.id}>
          <label htmlFor={`item-${item.id}-title`}>Title</label>
          <Field id={`item-${item.id}-title`} name={`items.${index}.title`} />
          <button type="button" onClick={() => remove(index)}>
            Remove
          </button>
        </div>
      ))}
      <button type="button" onClick={() => push({ id: createId(), title: '' })}>
        Add
      </button>
    </>
  )}
</FieldArray>
```

При add генерируй `id`, если серверного ещё нет. Не `key={index}`.

```tsx
// не надо — key={index}; useFieldArray RHF; параллельный useState
const [rows, setRows] = useState([{ title: '' }]);
const { fields } = useFieldArray({ name: 'items' });

{
  rows.map((row, index) => (
    <input key={index} value={row.title} onChange={(e) => updateRow(index, e.target.value)} />
  ));
}
```

**Почему.** Индекс как `key` ломает identity при перестановке. `useFieldArray` — API RHF. Параллельный `useState` расходится с `values.items`.

**Как исправить.** Один источник — `values.items` через `FieldArray`. `key={item.id}`. Добавление — `push` со stable id.
