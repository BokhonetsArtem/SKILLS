# Примеры: native React

Читай **только этот файл**, если в зоне **нет** form library: `useState` / `useReducer` + нативный `<form>`. `examples-rhf.md` и `examples-formik.md` не открывай и пакеты из них не ставь.

Сниппеты — обобщённые рецепты. Не копируй `@/`, имена слайсов и UI-кит как канон. Импорты, виджеты и i18n — как у **соседей этой зоны**. Runtime-логи и значения полей не добавляй.

Native — это не «RHF без пакета». Не подставляй `register` / `FormProvider` / `<Formik>` / `useFieldArray`.

Правила стека, schema, a11y и submit — в соседних файлах этой папки.

Как читать пары: сверху **надо**, ниже **не надо**, затем почему.

## 1. Простая форма

Один объект значений в локальном state. Нативный `<form onSubmit>` с `preventDefault`. Label — `htmlFor` + `id` контрола, не `htmlFor={name}`. Ошибки — словарь с теми же ключами, что поля.

```tsx
// надо
type ThingFormValues = { title: string };
type ThingFormErrors = Partial<Record<keyof ThingFormValues, string>>;

export function ThingForm() {
  const [values, setValues] = useState<ThingFormValues>({ title: '' });
  const [errors, setErrors] = useState<ThingFormErrors>({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  const titleRef = useRef<HTMLInputElement>(null);
  const inFlightRef = useRef(false);
  const generatedId = useId();
  const titleId = `thing-${generatedId}-title`;
  const titleErrorId = `${titleId}-error`;

  function onSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    if (inFlightRef.current) return;
    const nextErrors = validateThing(values);
    setErrors(nextErrors);
    if (nextErrors.title) {
      titleRef.current?.focus();
      return;
    }
    inFlightRef.current = true;
    void submitThing(values, setErrors, setIsSubmitting).finally(() => {
      inFlightRef.current = false;
    });
  }

  return (
    <form onSubmit={onSubmit}>
      <label htmlFor={titleId}>Title</label>
      <input
        id={titleId}
        ref={titleRef}
        value={values.title}
        onChange={(event) => {
          setValues((current) => ({ ...current, title: event.target.value }));
          setErrors((current) => ({ ...current, title: undefined }));
        }}
        aria-invalid={Boolean(errors.title)}
        aria-describedby={errors.title ? titleErrorId : undefined}
      />
      {errors.title ? <p id={titleErrorId}>{errors.title}</p> : null}
      <button type="submit" disabled={isSubmitting}>
        Save
      </button>
    </form>
  );
}
```

Если кнопка в header вне полей — один `<form id={formId}>` **или** `form={formId}` + `type="submit"` на кнопке. `formId` должен быть уникальным (`useId`/prop). Enter и клик — тот же `onSubmit`.

Form state не клади в глобальный store без прецедента зоны.

```tsx
// не надо — RHF/Formik «заодно»; store; htmlFor={name}; submit кликом; потерянный ref
const title = useAppSelector((s) => s.thingForm.title);
const { register, handleSubmit } = useForm();

function ThingForm() {
  return (
    <>
      <label htmlFor="title">Title</label>
      <input {...register('title')} value={title} />
      <Formik initialValues={{ title }} onSubmit={onCreate}>
        <Field name="title" />
      </Formik>
      <Button onClick={() => onCreate({ title })}>Save</Button>
    </>
  );
}
```

**Почему.** `useForm` / `<Formik>` — установка второй (и третьей) библиотеки в native-зону. Store дублирует локальные значения. Без `<form>` Enter не работает. `onClick` обходит валидацию и double-submit. `htmlFor={name}` не связывает label с input.

**Как исправить.** Только native state + `<form onSubmit>`. Не ставь `react-hook-form` / `formik`, пока пользователь явно не попросил. Прокинь `ref` на input. Label — стабильный `id`.

Edit: начальные значения — маппинг сущности → form values в `useState` / `useReducer`. После догрузки — одна гидрация (`setValues(mapped)`), не второй источник правды в пропах инпутов.

## 2. Schema validation

Проверки — одна функция `validate` (или уже стоящая в зоне yup/zod **без** resolver RHF/Formik). Не дублируй required в `onSubmit` ещё раз. Не ставь yup/zod «заодно», если соседи валидируют руками.

```tsx
// надо — как у соседей зоны; i18n только если он уже есть
type ContactFormValues = { title: string; email: string };
type ContactFormErrors = Partial<Record<keyof ContactFormValues, string>>;
const CONTACT_FIELD_ORDER = ['title', 'email'] as const;

function validateContact(values: ContactFormValues): ContactFormErrors {
  const errors: ContactFormErrors = {};
  if (!values.title.trim()) errors.title = t('validation.required');
  if (!values.email.trim()) errors.email = t('validation.required');
  else if (!isEmail(values.email)) errors.email = t('validation.email');
  return errors;
}

function ContactForm() {
  const [values, setValues] = useState<ContactFormValues>({ title: '', email: '' });
  const [errors, setErrors] = useState<ContactFormErrors>({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  const refs = {
    title: useRef<HTMLInputElement>(null),
    email: useRef<HTMLInputElement>(null),
  };

  function onSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    const nextErrors = validateContact(values);
    setErrors(nextErrors);
    const firstInvalidField = CONTACT_FIELD_ORDER.find(
      (field) => nextErrors[field],
    );
    if (firstInvalidField) {
      refs[firstInvalidField].current?.focus();
      return;
    }
    void submitContact(values, setErrors, setIsSubmitting);
  }

  return (
    <form onSubmit={onSubmit}>
      {/* title/email inputs используют values, errors и соответствующие refs */}
      <button type="submit" disabled={isSubmitting}>Save</button>
    </form>
  );
}
```

Если в native-зоне уже есть yup/zod как чистые функции — `schema.validateSync` / `schema.safeParse` в той же `validateThing`. Не оборачивай это в `yupResolver` / Formik `validationSchema`.

```tsx
// не надо — схема и ручные if; resolver RHF; Zod «потому что TypeScript»
const form = useForm({ resolver: yupResolver(thingSchema) });

function onSubmit(event: FormEvent<HTMLFormElement>) {
  event.preventDefault();
  if (!values.title.trim()) return;
  const parsed = thingZodSchema.safeParse(values);
  if (!parsed.success) return;
  if (!validateThing(values).title) void createThing(values);
}
```

**Почему.** Несколько каналов валидации расходятся. `yupResolver` тянет RHF. Новая schema library без запроса — смена стека.

**Как исправить.** Один `validate` / одна уже стоящая schema-функция. Submit маппит values → DTO. Client validation — не защита сервера.

## 3. Async submit

Мутация только из `onSubmit`. Флаг `isSubmitting` (или `useRef` для in-flight) дизейблит кнопку и игнорирует повторный Enter. Общая ошибка — toast/status формы. Ошибка поля — §4.

```tsx
// надо
async function submitThing(
  values: ThingFormValues,
  setErrors: Dispatch<SetStateAction<ThingFormErrors>>,
  setIsSubmitting: Dispatch<SetStateAction<boolean>>,
) {
  setIsSubmitting(true);
  try {
    await createThing(toDto(values));
  } catch (error) {
    const fieldResult = applyFieldErrors(error, setErrors);
    if (!fieldResult.allHandled) showRequestError(error);
    if (fieldResult.handledCount > 0) return;
  } finally {
    setIsSubmitting(false);
  }
}

const formId = useId();

<form id={formId} onSubmit={onSubmit}>
  {/* поля */}
</form>
<button type="submit" form={formId} disabled={isSubmitting}>
  Save
</button>
```

Не вызывай `createThing` из `onClick` кнопки, если тот же обработчик уже на `onSubmit`.

```tsx
// не надо — клик вместо form; нет защиты от повторной отправки; обход validate
function ThingHeader() {
  return <Button onClick={() => createThing(values)}>Save</Button>;
}
```

**Почему.** Enter не сработает. Клик шлёт невалидные данные. Двойной клик — две мутации.

**Как исправить.** Один `onSubmit` с `preventDefault` → `validate` → мутация. Кнопка в header — `type="submit"` + `form` id. `disabled={isSubmitting}`.

## 4. Server field errors — словарь поля обязателен

Эквивалент `setError` — запись в **тот же** словарь ошибок, что и client validation. Ключ совпадает с контролом. Не ограничивайся toast.

```tsx
// надо
function isThingField(field: string): field is keyof ThingFormValues {
  return field === 'title' || field === 'email';
}

function applyFieldErrors(
  error: unknown,
  setErrors: Dispatch<SetStateAction<ThingFormErrors>>,
): { handledCount: number; allHandled: boolean } {
  const items = parseFieldErrors(error); // форма тела 400/409 — как у API зоны
  if (!items?.length) return { handledCount: 0, allHandled: false };
  const knownItems = items.filter(
    (item): item is typeof item & { field: keyof ThingFormValues } =>
      isThingField(item.field),
  );

  setErrors((current) => {
    const next = { ...current };
    for (const item of knownItems) next[item.field] = item.message;
    return next;
  });
  return {
    handledCount: knownItems.length,
    allHandled: knownItems.length === items.length,
  };
}
```

Покажи `errors.email` рядом с полем и в `aria-describedby`. Если часть или все поля ответа неизвестны, дополнительно покажи общую ошибку формы. Сброс ключа — при вводе / новом submit.

```tsx
// не надо — только toast; отдельный serverError не связан с input
async function submitThing(values: ThingFormValues) {
  try {
    await createThing(values);
  } catch (error) {
    toast.error(error.message);
    setServerBanner(error.message);
  }
}
```

**Почему.** Баннер/toast не ставят `aria-invalid` на поле и не ведут focus. Пользователь не видит, какое поле отклонил сервер.

**Как исправить.** Известные поля — ключи в `errors`. Toast — для 500/сети.

## 5. Динамический список

Массив в form state. React `key` — stable id, сгенерированный при add (или серверный id). Индекс — только чтобы взять элемент, не как identity.

```tsx
// надо
type Item = { id: string; title: string };

function addItem() {
  setValues((current) => ({
    ...current,
    items: [...current.items, { id: createId(), title: '' }],
  }));
}

function removeItem(id: string) {
  setValues((current) => ({
    ...current,
    items: current.items.filter((item) => item.id !== id),
  }));
}

{
  values.items.map((item) => (
    <div key={item.id}>
      <label htmlFor={`item-${item.id}-title`}>Title</label>
      <input
        id={`item-${item.id}-title`}
        value={item.title}
        onChange={(event) => updateItem(item.id, event.target.value)}
      />
      <button type="button" onClick={() => removeItem(item.id)}>
        Remove
      </button>
    </div>
  ));
}
```

Не подключай `useFieldArray` / Formik `FieldArray` в native-зоне.

```tsx
// не надо — key={index}; useFieldArray; Formik FieldArray; список вне form state
const { fields } = useFieldArray({ name: 'items' });
const [rows, setRows] = useState([{ title: '' }]);

{
  rows.map((row, index) => (
    <FieldArray name="items" key={index}>
      <input value={row.title} onChange={(e) => updateRow(index, e.target.value)} />
    </FieldArray>
  ));
}
```

**Почему.** Индекс как `key` ломает строки при удалении/перестановке. `useFieldArray` / `FieldArray` — чужой трек и новые зависимости. Список в отдельном `useState` рядом с `values` расходится при submit.

**Как исправить.** `items` внутри того же объекта form values. `key={item.id}`. Add — новый id, не индекс.
