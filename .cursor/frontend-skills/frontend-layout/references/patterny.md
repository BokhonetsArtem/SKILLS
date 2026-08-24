# Паттерны страниц и блоков

Готовые сценарии. Сначала прочитай [pravila.md](pravila.md) и файл трека, выбранный на шаге 2 `SKILL.md` (Modules, Tailwind, CSS-in-JS, SCSS или UI-kit). Для иного стека — общие правила и ближайшая страница той же зоны.

Это рецепты для репозитория, где уже есть такие оболочки (`PageWrapper`, `SearchSection`, `Flex`, `FormField`, …). **Нет их в проекте** — найди ближайшую страницу той же зоны в **этом** репо и копируй её состав. Не тащи YEAHUB-оболочки и не копируй Tailwind-сниппеты в репо без Tailwind.

Логи в вёрстку **не** добавляй. Сначала ближайшая страница **той же зоны**. Не тащи админский бар поиска на каталог и наоборот.

| Задача | Опора в репозитории |
| --- | --- |
| Админ-таблица + поиск + фильтры | [`UsersTablePage.tsx`](../../../../src/pages/admin/user/users/ui/UserTablePage/UsersTablePage.tsx) |
| Wiki / каталог (колонка фильтров) | [`QuestionsPage.tsx`](../../../../src/pages/wiki/question/questions/ui/QuestionsPage/QuestionsPage.tsx) |
| Сетка карточек | [`GurusList.module.css`](../../../../src/entities/guru/ui/GurusList/GurusList.module.css) |
| Форма | [`SkillCreateForm.tsx`](../../../../src/features/skill/createSkill/ui/SkillCreateForm/SkillCreateForm.tsx) + `FormField` в [`SkillForm.tsx`](../../../../src/entities/skill/ui/SkillForm/SkillForm.tsx) |
| Адаптив | `useScreenSize` из `@/shared/libs` + `@media` ближайшего module той же фичи |

## 1. Таблица с фильтрами (канон YEAHUB)

Страница **собирает** оболочку. Таблицу и фильтры не реализуй заново на page.

Состав:

1. `PageWrapper` — `stubs` (`error` / `filter-empty` / `empty`), `paginationOptions`, `roles`, флаги загрузки/ошибки/данных, `content` (таблица).
2. Render-prop: `<Flex componentType="main" direction="column" gap="24">`.
3. `SearchSection` — поиск и `renderFilter` (`FiltersDrawer` **уже внутри**). Кнопка создания — опциональный проп `to`: route-строка или render-функция, как в ближайшей admin-странице. Сброс — опциональная пара `onResetFilters` + `showResetFilterButton`; не добавляй её, если соседние страницы сброс не показывают. У `UsersTablePage` нет `to`, но есть reset.
4. `Card` — `{content}` + `{pagination}`.
5. `Table` — внутри `content` (часто отдельный компонент таблицы страницы).

Ключи стабов копируй с эталона, не выдумывай `User.STUB_EMPTY_TITLE` — у `User` это `STUB_EMPTY_USERS_*`.

```tsx
const stubs: PageWrapperStubs = {
  error: { onClick: refetch },
  'filter-empty': { onClick: onResetFilters },
  empty: {
    title: t(User.STUB_EMPTY_USERS_TITLE),
    subtitle: t(User.STUB_EMPTY_USERS_SUBTITLE),
    buttonText: t(User.STUB_EMPTY_USERS_SUBMIT),
    onClick: () => navigate(ROUTES.admin.users.page),
  },
};

return (
  <PageWrapper
    isLoading={isLoading}
    hasError={isError}
    hasFilters={hasFilters}
    hasData={hasData}
    stubs={stubs}
    roles={['admin']}
    paginationOptions={{
      page: filters.page || 1,
      onChangePage,
      limit: users?.limit || 0,
      total: users?.total || 0,
    }}
    content={<UsersTable users={users?.data} />}
  >
    {({ content, pagination }) => (
      <Flex componentType="main" direction="column" gap="24">
        <SearchSection
          renderFilter={() => (
            <UsersFilters
              filters={filters}
              onChangeRoles={onChangeRoles}
              onChangeIsVerified={onChangeIsVerified}
            />
          )}
          showResetFilterButton={hasFilters}
          hasFilters={hasFilters}
          searchValue={filters.search}
          onSearch={onChangeSearch}
          onResetFilters={onResetFilters}
        />
        <Card className={styles.content}>
          <>
            {content}
            {pagination}
          </>
        </Card>
      </Flex>
    )}
  </PageWrapper>
);
```

### Не надо

```tsx
// FiltersDrawer сиблингом Table на странице — ломает канон SearchSection
<Flex componentType="main" direction="column" gap="24">
  <input value={search} onChange={onSearch} />
  <FiltersDrawer hasFilters={hasFilters}>
    <UsersFilters {...filterProps} />
  </FiltersDrawer>
  <UsersTable users={users} />
</Flex>
```

- Не дублируй empty/error stubs руками, если уже есть `PageWrapper`.
- Не клади пагинацию вне `Card`, если канон страницы — таблица в карточке.
- На запрос «админ-таблица с фильтрами» бери `UsersTablePage`, не wiki. Wiki / каталог — раздел ниже.
- `TableMobile` **не канон**. Компонент есть в `shared/ui`, страницы его **не** импортируют. Не подключай его «для мобилки».

### Другой стек (не YEAHUB)

Тот же состав по смыслу: page shell → колонка `main` с `gap` → search/filter bar проекта → карточка → таблица. Фильтры не выноси сиблингом таблицы, если в проекте бар уже умеет drawer. Синтаксис — стек репозитория, не сниппет ниже «на всякий случай».

## 1.1 Wiki / каталог (колонка фильтров)

Эталон: [`QuestionsPage.tsx`](../../../../src/pages/wiki/question/questions/ui/QuestionsPage/QuestionsPage.tsx). Похожие layouts есть у wiki `CollectionsPage` / `ResourcesPage` и у отдельной зоны `TasksPage`, но их брейкпоинты различаются. Это **не** админ-таблица: `SearchSection` сюда не тащи.

Состав:

1. `PageWrapper` + `stubs` + `paginationOptions` + `content` (список, не `Table`).
2. `skeleton` в `PageWrapper` — только если ближайшая страница передаёт его туда (у `QuestionsPage` — да). Не путай с route-level skeleton: `UsersTablePage.skeleton.tsx` существует и используется `SkeletonGenerator`, но сама страница не передаёт его в `PageWrapper`.
3. Ряд: основная `Card` + боковая `Card` с фильтрами.
4. На узком экране боковую колонку прячет `@media`, фильтры открывает `FiltersDrawer` на странице. Порог бери из этой же зоны: у `QuestionsPage` / `TasksPage` это 1280px, у `CollectionsPage` / `ResourcesPage` — 1024px.

```tsx
{({ content, pagination }) => (
  <Flex gap="20" align="start">
    <Card className={styles.main}>
      <div className={styles['questions-list-header']}>
        <Text variant={isMobileS ? 'body5-accent' : 'body6'} isMainTitle maxRows={1}>
          {t(Questions.TITLE_SHORT)}
        </Text>
        {isSmallScreen && <FiltersDrawer>{renderFilters()}</FiltersDrawer>}
      </div>
      <>
        {content}
        {pagination}
      </>
    </Card>
    <Card className={styles.filters}>{renderFilters()}</Card>
  </Flex>
)}
```

```css
.filters {
  position: sticky;
  top: 0;
  min-width: 360px;
  max-width: 360px;
}

@media (width < 1280px) {
  .filters {
    display: none;
  }
}
```

Не копируй этот `FiltersDrawer` на админ-страницу «таблица + поиск» — там слот `SearchSection.renderFilter`.

## 2. Карточки: grid в module

Эталон — `GurusList`: список = `display: grid` в `*.module.css`. Одномерный ряд внутри карточки — `Flex`.

```css
.list {
  display: grid;
  grid-template-columns: repeat(2, auto);
  gap: 20px;
}

.list-with-borders {
  grid-template-columns: repeat(auto-fill, minmax(289px, 1fr));
  width: 100%;
}

@media (width < 480px) {
  .list {
    grid-template-columns: 1fr;
  }
}
```

```tsx
<ul
  className={classNames(styles.list, {
    [styles['list-with-borders']]: variant === 'list-with-borders',
  })}
>
  {gurus.map((guru) => (
    <GurusItem key={guru.id} guru={guru} />
  ))}
</ul>
```

Не собирай сетку тройным вложенным `Flex` и не пиши `display: flex` + процентные ширины карточек в module фичи.

### Другой стек

Сетка — `display: grid` в том синтаксисе, который уже есть (module, утилиты, `sx`, styled). В репозиторий без Tailwind className с `grid grid-cols-*` **не** вставляй.

## 3. Форма: `Flex` + `Card` + `FormField`

Эталон — `SkillCreateForm` (оболочка) + `SkillForm` (поля).

```tsx
<FormProvider {...methods}>
  <LeavingPageBlocker isBlocked={isDirty && !isSubmitted && !isSubmitting}>
    <Flex componentType="main" direction="column" gap="24">
      <SkillCreateFormHeader />
      <Card className={styles.content}>
        <SkillForm />
      </Card>
    </Flex>
  </LeavingPageBlocker>
</FormProvider>
```

Поле — `FormField` (лейбл + описание) и контрол внутри, не «голый» `Input` без подписи:

```tsx
<Flex direction="column" gap="60">
  <FormField label={t(Skills.TITLE_FULL)} description={t(Skills.TITLE_LABEL)}>
    <FormControl name="title" control={control} className={styles['input-form']}>
      {(register, hasError) => <Input {...register} error={hasError} />}
    </FormControl>
  </FormField>
</Flex>
```

- Header (назад / сохранить) — отдельный `Flex`, не внутри `Card`.
- Строки — `t(...)`.
- Не заводи свою сетку лейбл+инпут через `display: flex` в module, если есть `FormField`.

### Другой стек

Колонка с `gap`, поля через Form-примитив проекта, не `placeholder` вместо label. Синтаксис колонки — как у соседей, не обязательно Tailwind.

## 4. Page shell

`PageWrapper` закрывает loading, error, filter-empty, empty, роли, пагинацию.

Передавай:

- `stubs` — колбэки и тексты (`t(...)`) для нужных типов;
- `roles`, если страница ограниченная;
- `paginationOptions`, если список с страницами;
- `content` — основное тело (таблица, детали);
- `skeleton` — передавай в `PageWrapper`, только если так делает соседняя страница той же зоны. Отдельный route-level `*.skeleton.tsx` не означает, что его нужно дублировать в `PageWrapper`.

Дочерний `Flex` с `componentType="main"` — лендмарк страницы. Не плоди второй `<main>`.

Не надо:

```tsx
if (isLoading) return <Loader />;
if (!data.length) return <div>Пусто</div>;
return <Table items={data} />;
```

если задача — обычная админ-страница со стабами.

## 5. Адаптив

1. Хуки проекта: `useScreenSize` (`@/shared/libs`). Флаги: `isMobileS` (≤480), `isMobileM` (≤567), `isMobile` (≤767), `isTablet` (768–1023), `isLaptop` (1024–1279), `isDesktopS` (1280–1439), `isDesktop` (≥1440), `isLargeScreen` (≥1280), `isSmallScreen` (<1280).
2. Сетка и контейнеры — `@media` как в ближайшем module той же фичи (`width < 480px`, `768px`, `1024px`, `1280px`, `1440px` — что уже есть рядом). Даже похожие страницы расходятся: wiki filters скрываются на 1024px или 1280px в зависимости от зоны. Не канонизируй значение из другого экрана и редкие значения вроде 1198, если в этой фиче их нет.
3. Не вводи дефолты Tailwind 640 / 768 / 1024 «как в доке Tailwind», если в модулях уже другие числа.

```tsx
import { useScreenSize } from '@/shared/libs';

const { isMobile, isTablet, isMobileS } = useScreenSize();

<Text variant={isMobileS ? 'body3-accent' : 'body5-accent'}>
  {t(Skills.TITLE_FULL)}
</Text>
```

```css
@media (width < 480px) {
  .list {
    grid-template-columns: 1fr;
  }
}
```

- JS-флаги — смена состава (скрыть блок, другой `variant` у `Text`, другой `direction` у `Flex`).
- CSS — колонки, отступы, перенос сетки.
- Container queries — только если в репозитории уже есть или пользователь явно просил.
- **`TableMobile` не используй** как мобильную таблицу страницы. Тот же `Table` + `@media` / `useScreenSize` для обвязки.

На треке Tailwind (только если он есть): префиксы `sm:` / `md:` / `lg:` из **конфига проекта**. Если брейкпоинты кастомные — повтори их, не дефолт 640/768/1024. В остальных стеках — хуки и `@media` соседей.

## Чеклист паттерна

- [ ] Админ-таблица: `PageWrapper` + `Flex` `main` `gap="24"` + `SearchSection` (`renderFilter`) + `Card` + `Table`
- [ ] Wiki / каталог: колонка фильтров как `QuestionsPage`, не `SearchSection`
- [ ] Админ-таблица: `FiltersDrawer` не сиблинг `Table` на странице
- [ ] Карточки: grid в module
- [ ] Форма: `Flex` + `Card` + `FormField`
- [ ] Стабы и роли — через `PageWrapper`, не самодельные `if`
- [ ] Адаптив: `useScreenSize` + `@media` ближайшего модуля
- [ ] `TableMobile` не подключён «для канона»
- [ ] Нет логов
