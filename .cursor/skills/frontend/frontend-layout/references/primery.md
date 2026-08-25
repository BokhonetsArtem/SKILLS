# Примеры: как надо и как не надо

Читай, когда неясно, какой приём выбрать. Пары короткие: слева/сверху — **надо**, справа/снизу — **не надо**.

- Синтаксис в примерах — **иллюстрация**. Копируй только колонки, чей стек совпадает с репозиторием. Нет Tailwind — не копируй `className="flex gap-4"`. Нет `Flex` — не копируй `<Flex gap="16">`. Нет CSS Modules — не заводи `*.module.css` из примера.
- «Репозиторий без Tailwind» ≠ CSS Modules. SCSS, styled-components, Emotion, `sx`, vanilla CSS — свой файл трека ([scss.md](scss.md), [css-in-js.md](css-in-js.md), [ui-kit.md](ui-kit.md)) и соседи, не колонка Modules из этого файла.
- Логи (`console.log`, обвязка логгера) в вёрстку и в эти примеры **не** добавляй.
- Готовые сценарии страниц (таблица, карточки, форма, shell, адаптив) — [patterny.md](patterny.md), и только если в репо есть те же оболочки.

## 1. Ритм: `gap` на родителе, не margin-хак

### CSS Modules (если это стек)

```tsx
// надо
<Flex align="center" gap="16">
  <Button>{t(Translation.SAVE)}</Button>
  <Button variant="outline">{t(Translation.CANCEL)}</Button>
</Flex>
```

```tsx
// не надо — ритм на детях, last-child, сырой flex в module фичи
<div className={styles.row}>
  <Button className={styles.item}>{t(Translation.SAVE)}</Button>
  <Button>{t(Translation.CANCEL)}</Button>
</div>
```

```css
/* не надо */
.row { display: flex; }
.item { margin-right: 16px; }
.item:last-child { margin-right: 0; }
```

### Tailwind (только если это стек репозитория)

```tsx
// надо
<div className="flex items-center gap-4">
  <Button>Save</Button>
  <Button variant="outline">Cancel</Button>
</div>

// не надо
<div className="flex">
  <Button className="mr-4">Save</Button>
  <Button>Cancel</Button>
</div>
```

## 2. Типографика и цвет: `Text` / токены, не hex и `font-size`

### CSS Modules (если это стек)

```tsx
// надо
<Text variant="body5-strong">{t(Skills.CREATE_PAGE_TITLE)}</Text>
<Text variant="body2" color="black-800">{t(Skills.TITLE_LABEL)}</Text>
```

```css
/* надо — цвет из темы */
.panel {
  background-color: var(--background-block);
  color: var(--text-color-bright);
}
```

```tsx
// не надо
<h1 className={styles.title}>Создать навык</h1>
```

```css
/* не надо — упадёт stylelint declaration-strict-value */
.title {
  color: #400799;
  font-size: 20px;
  font-weight: 600;
}
```

### Tailwind

```tsx
// надо — шкала темы
<h1 className="text-xl font-medium text-foreground">{title}</h1>

// не надо — макетный hex и случайный px
<h1 className="text-[20px] font-[600] text-[#400799]">{title}</h1>
```

Если стек — CSS Modules с `Text`, не подменяй его утилитами Tailwind. В репо без `Text` — текстовый примитив или токены соседей, не hex.

## 3. Кнопка: reuse `Button`, не свой `<button>`

```tsx
// надо
import { Button } from '@/shared/ui/Button';

<Button size="large" onClick={onResetFilters}>
  {t(Translation.STUB_FILTER_SUBMIT)}
</Button>

<Button size="large" variant="link" disabled={isLoading} onClick={handleSubmit(onSave)}>
  {t(Translation.SAVE)}
</Button>
```

```tsx
// не надо — свой контрол рядом с существующим примитивом
<button type="button" className={styles.save} onClick={onSave}>
  Сохранить
</button>
```

```tsx
// не надо на треке Tailwind, если в проекте уже есть Button
<button className="rounded-md bg-purple-700 px-4 py-2 text-white">Save</button>
```

Иконка-кнопка — `IconButton` / `Button`, не `div` с `onClick`. Подпись — `t(...)`, не хардкод.

## 4. Сетка карточек: `display: grid` в module, не тройной nested flex

Эталон: [`GurusList.module.css`](../../../../../src/entities/guru/ui/GurusList/GurusList.module.css).

```css
/* надо */
.list {
  display: grid;
  grid-template-columns: repeat(2, auto);
  gap: 20px;
}

@media (width < 480px) {
  .list {
    grid-template-columns: 1fr;
  }
}
```

```tsx
<ul className={classNames(styles.list)}>{items}</ul>
```

```tsx
// не надо — 2D через вложенный Flex
<Flex gap="20">
  <Flex direction="column" gap="20">{col1}</Flex>
  <Flex direction="column" gap="20">{col2}</Flex>
</Flex>
```

```css
/* не надо — сырой flex вместо grid для сетки карточек */
.list {
  display: flex;
  flex-wrap: wrap;
}
.card {
  width: 48%;
  margin: 0 1% 20px 0;
}
```

### Tailwind

```tsx
// надо
<ul className="grid grid-cols-2 gap-5 max-sm:grid-cols-1">{items}</ul>

// не надо
<div className="flex flex-wrap">
  <div className="mb-5 mr-[1%] w-[48%]">{card}</div>
</div>
```

## 5. Tailwind soup vs module для keyframes

Если стек — Tailwind: раскладка утилитами. Keyframes, `clip-path`, глубокие селекторы — в `*.module.css` (или `@keyframes` в CSS), не 15 утилит `animate-[…]`.

```tsx
// надо — hybrid
<div className={cn('relative flex flex-col gap-4', styles.fadeIn)}>
```

```css
@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}

.fade-in {
  animation: fade-in 200ms ease;
}
```

```tsx
// не надо — soup + дублирующий module на том же узле
<div
  className={cn(
    'flex flex-col gap-4 rounded-xl bg-white p-6 shadow-md',
    styles.card,
  )}
>
```

Если стек не Tailwind: анимация сразу в том синтаксисе, что у соседей, без утилит.

## 6. Поиск и фильтры: `SearchSection`, не самодельный бар

Канон админ-таблицы — [patterny.md](patterny.md) (по `UsersTablePage`). `FiltersDrawer` уже рендерится **внутри** `SearchSection` через `renderFilter`.

```tsx
// надо
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
```

```tsx
// не надо — свой тулбар и FiltersDrawer сиблингом Table на странице
<Flex gap="16">
  <input value={search} onChange={onSearch} placeholder="Поиск" />
  <FiltersDrawer hasFilters={hasFilters}>
    <UsersFilters {...filterProps} />
  </FiltersDrawer>
</Flex>
<Card>
  <Table {...tableProps} />
</Card>
```

Wiki / каталог с колонкой фильтров — другой layout, см. [patterny.md](patterny.md) § «Wiki / каталог». Не копируй оттуда `FiltersDrawer` на админ-страницу «таблица + поиск». Не копируй `SearchSection` на wiki.

## Чеклист примеров

- [ ] 1D — `gap` на родителе (примитив или flex стека), не margin-соседи
- [ ] Текст и цвет — примитив / токены проекта, не hex и `font-size` в новом CSS
- [ ] Кнопка — существующий примитив проекта
- [ ] Карточки — Grid синтаксисом стека
- [ ] Keyframes не через Tailwind soup, если стек не просит иное
- [ ] Фильтры: как у ближайшей страницы той же зоны (админ: `SearchSection`; wiki/каталог: `QuestionsPage`)
- [ ] Нет логов в разметке
