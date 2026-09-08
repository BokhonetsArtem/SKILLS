# Стек и обнаружение

Читай, когда трек неясен, в репо несколько способов или нужно решить, какую schema library трогать. Рецепты реализации — только `examples-*` **выбранного** трека, не все сразу. Валидация, поля и submit — соседние файлы этой папки.

Не ставь `react-hook-form`, `formik`, `yup`, `zod`, resolver или другую form/schema library, пока пользователь явно не попросил.

## 1. Это не форма этого скила

Слово «форма» в UI не равно form state. Если нет значений полей, schema, binding, submit/reset или ошибок полей — остановись и отдай другому скилу.

| Что видно | Куда |
| --- | --- |
| Сетка полей, CSS, Flex/Grid, макет Figma, «сверстай форму» | `frontend-layout` |
| Фильтры, поиск и пагинация UI без записи в URL маршрута и без form state | `frontend-layout` |
| Route tree, client redirect, params/search как URL-state маршрута | [`frontend-routing`](../../frontend-routing/SKILL.md) |
| Куда класть слайс, schema-файл, кто владеет state | `frontend-architecture` |
| Endpoint, query/mutation, нормализация ошибок HTTP, без form binding | [`frontend-api`](../../frontend-api/SKILL.md) |
| XSS, unsafe preview, секреты в поле | `frontend-security` |

## 2. Признаки трека

Смотри **зону изменения**, не популярность библиотеки. Достаточно сигналов, не полный аудит репо.

| Доказательство | Трек |
| --- | --- |
| `package.json` / lockfile: `react-hook-form`; импорты `useForm`, `FormProvider`, `useController`, `Controller`, `useFieldArray`, `@hookform/resolvers` | RHF |
| `package.json` / lockfile: `formik`; импорты `useFormik`, `<Formik>`, `Field`, `FastField`, `FieldArray`, `ErrorMessage` | Formik |
| Нет form library в манифесте зоны; `useState` / `useReducer` + нативный `<form>` и обработчики полей | native |
| `action`, server action, `useActionState`, route action или form helper фреймворка владеют submit/state | framework-managed трек этой зоны |
| Импорты другой установленной form library (`@tanstack/react-form`, Final Form, Conform и т.п.) | трек этой библиотеки; изучи локальные формы и API установленной версии |
| В lockfile одна библиотека, в **этой** зоне импорты другой | трек **зоны**, не «главный» пакет репо |

Имя компонента `*Form*` само по себе ничего не доказывает. Смотри хуки, провайдер и submit.

Native — это не «RHF без пакета». Framework-managed форма — не обязательно controlled `useState`. Не подставляй `register` / `FormProvider` / `<Formik>` и не переписывай `action`-поток, если их нет в локальном треке.

## 3. Schema library

Следуй уже стоящей в зоне. Не «улучшай» стек второй схемой.

| Что уже есть у соседей той же зоны | Что делать |
| --- | --- |
| `yup` + `@hookform/resolvers/yup` | `yupResolver`, типы через `yup.InferType` / уже принятый form-values тип |
| `zod` + `@hookform/resolvers/zod` | `zodResolver`, `z.infer` |
| Formik + `yup` (`validationSchema`) | yup-схема Formik, не resolver RHF |
| Formik + `zod` (через адаптер, который уже в репо) | тот же адаптер |
| Схемы нет, проверки в обработчике / `validate` | не добавляй yup/zod без запроса |
| yup в одной фиче, zod в другой | схема **этой** фичи; не унифицируй репо |

Не подставляй Zod в yup-проект и наоборот. Не ставь `@hookform/resolvers`, если RHF резолвит схему иначе или схемы нет.

## 4. Смешанный и неясный стек

| Ситуация | Решение |
| --- | --- |
| В репо несколько библиотек, в зоне один устойчивый прецедент | Трек зоны. Чужой examples-файл не читай |
| В одной форме уже смешаны RHF и Formik / RHF и параллельный `useState` на те же поля | Не усугубляй. Новую логику — в уже выбранный трек формы |
| Библиотека распознана, но для неё нет examples-файла | Следуй локальным формам и API установленной версии; примени тематические правила a11y/lifecycle/validation, не синтаксис чужого трека |
| Framework управляет submit/state | Сохрани route/server action и его pending/error API; не создавай параллельный client state без необходимости |
| Два равноправных способа в зоне или сигналов мало | Спроси. Не ставь пакет и не выбирай «как принято в индустрии» |
| Пользователь явно просит другую библиотеку | Можно ставить только после явного запроса; не мигрируй соседние формы заодно |

Не смешивай RHF с Formik в одной форме. Не дублируй form state в глобальный store, если в зоне форма живёт в `useForm` / Formik / локальном state.

## 5. Reuse примитивов зоны

Если в зоне уже есть оболочка поля, контроллер или leave-guard — **переиспользуй**, не заводи параллельный кит (`FormControlV2`, свой `useLeaveGuard`).

Имена, импорты и пропсы — как у ближайшей формы той же зоны, не из другого репозитория. Чужие пути вроде `@/entities/skill` не переноси туда, где их нет.

Reuse **не** значит копировать дыры примитивов. Локальные anti-patterns (отброшенный `ref`, `htmlFor={name}`, несвязанная подпись, submit только через `onClick`) — в [fields-and-accessibility.md](fields-and-accessibility.md) и [lifecycle-and-submit.md](lifecycle-and-submit.md). При использовании кита не изобретай второй; при правке кита или новой обвязке поля эти дыры не воспроизводи.

## 6. Что читать дальше

| Нужно | Файл |
| --- | --- |
| Schema, типы, `setError` / Formik `errors`+`status`, i18n ошибок | [validation-and-types.md](validation-and-types.md) |
| Label, `aria-*`, focus ошибки, виджеты | [fields-and-accessibility.md](fields-and-accessibility.md) |
| Submit, reset, dirty, arrays, leave-guard | [lifecycle-and-submit.md](lifecycle-and-submit.md) |
| Сниппеты | только `examples-rhf.md` / `examples-formik.md` / `examples-native.md` своего трека |
