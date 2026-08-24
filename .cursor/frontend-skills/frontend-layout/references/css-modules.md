# Трек CSS Modules

Читай после [pravila.md](pravila.md) **только если** в репозитории доминируют CSS Modules (`*.module.css` или `*.module.scss`). Это не дефолт скила и не запасной вариант «если нет Tailwind».

Не добавляй `console.log` и прочее логирование в компоненты вёрстки. Не вставляй Tailwind в репозиторий без Tailwind.

Конвенции ниже сначала сверяй с ближайшими файлами: `classnames` vs `clsx`, kebab vs camel, есть ли `Flex`/`Text`, FSD, i18n, `.css` vs `.scss`. Где в репо иначе — повтори репо, не этот файл.

## Когда этот трек

Сигналы: `import styles from './Name.module.css'` или `./Name.module.scss`, нет доминирующего Tailwind.

- Только `*.module.css` (YEAHUB) — этот файл. Sass **не** добавляй.
- `*.module.scss` / `*.module.sass` — этот файл **и** [scss.md](scss.md). Scoping, colocation, `classNames` — отсюда; `$`, `@use`, nest, mixins — из SCSS. Не переименовывай в `.module.css` «чтобы было как в каноне».

В YEAHUB этот трек обязателен (FSD, `var(--*)`, stylelint `declaration-strict-value`). В другом Modules-проекте — те же идеи, но имена примитивов и линтер — как там.

## Файлы рядом с компонентом

Если в репозитории module лежит иначе (не рядом, другой суффикс) — повтори раскладку проекта.

```
SkillCreateForm/
  SkillCreateForm.tsx
  SkillCreateForm.module.css   /* или .module.scss — как у соседей */
```

- Стили через CSS Modules, не глобальные классы и не inline `style` для цвета/типа (исключение: динамическая ширина вроде `style={{ width }}` у `Text`).
- Импорт: `import styles from './Name.module.css'` или `from './Name.module.scss'` — тот суффикс, что у соседей. Не смешивай `.module.css` и `.module.scss` в одном компоненте.
- Состав классов: как у соседей. В YEAHUB — пакет **`classnames`**, импорт **`classNames`**. **Не** подменяй на `clsx` / `cva` / `cn` из shadcn, если в репо их нет. Не копируй редкий локальный алиас `import cn from 'classnames'` из старых файлов YEAHUB.

```tsx
import classNames from 'classnames';
import styles from './FiltersBar.module.css';

className={classNames(styles['filter-button'], isActive && styles.active, className)}
```

## Имена классов: как в репозитории

В YEAHUB в CSS — kebab-case. В TSX к классу с дефисом — скобки: `styles['filter-button']`. Одно слово — `styles.list`. Не пиши `.filterButton` в **новых** модулях YEAHUB.

В другом Modules-проекте — как у соседей (camelCase тоже бывает). Не навязывай kebab, если файлы рядом в другом стиле.

## Без `composes`

Если в репозитории нет `composes:` — не вводи CSS Modules composition. Общие куски — компонент или повтор токена, не `@extend` / `composes`. Если соседи уже используют `composes` — повтори их, не этот абзац.

## 1D = layout-примитив с `gap`, когда он подходит

Если в репозитории есть ряд-примитив вроде [`Flex`](../../../../src/shared/ui/Flex/Flex.tsx), предпочитай его для самостоятельных layout-обёрток страниц и фич. Но не добавляй лишний DOM-узел только ради `Flex`: внутренний layout уже существующего элемента, сложные селекторы и случаи, где так делают ближайшие компоненты, можно оформить через `display: flex` в module. Нет такого примитива — пиши flex так, как соседние modules.

```tsx
import { Flex } from '@/shared/ui/Flex';

<Flex componentType="main" direction="column" gap="24">
  <Header />
  <Card>{children}</Card>
</Flex>
```

На YEAHUB проп `gap` — только шкала `FlexGap`: `"4"` | `"6"` | `"8"` | `"10"` | `"12"` | `"14"` | `"16"` | `"20"` | `"24"` | `"26"` | `"28"` | `"30"` | `"32"` | `"40"` | `"48"` | `"52"` | `"60"` | `"100"` | `"120"`. Не выдумывай `"18"`.

Другие пропы YEAHUB: `direction`, `justify`, `align`, `wrap`, `maxWidth`, `maxHeight`, `flex={1}` (в `flexClasses` только `1`), `componentType` (`"main"` для оболочки админ-страницы).

Не дублируй простую самостоятельную обёртку CSS-классом, если её без изменения DOM и поведения выражает `Flex`:

```css
.row {
  display: flex;
  gap: 16px;
}
```

`display: flex` в module допустим для внутреннего устройства компонента или существующего семантического элемента, когда замена на `Flex` потребует лишней обёртки, потеряет нужный селектор или расходится с паттерном ближайшей зоны.

## 2D = `display: grid` в module

Отдельного Grid-компонента нет. Сетка карточек, колонки, дашборд — в `*.module.css`:

Эталон: [`GurusList.module.css`](../../../../src/entities/guru/ui/GurusList/GurusList.module.css).

```css
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

`gap` / padding / width в layout **могут** быть px, если попадают в шкалу проекта (20, 24, 16…). Stylelint не требует `var()` для `gap`. Цвет и шрифт — всегда токены.

## Текст: примитив проекта, не `font-size` в новом module

Если есть [`Text`](../../../../src/shared/ui/Text/Text.tsx): обязательный `variant`, цвет из `Pallete` (`color="black-800"`, по умолчанию `black-900`). `variant` только из [`Text/types.ts`](../../../../src/shared/ui/Text/types.ts). Не комбинируй суффиксы наугад: `body4-accent` и `body6-strong` в типе нет.

Нет `Text` — токены/классы как у соседей, всё равно не сырой `font-size` в новом module «подогнать заголовок».

```tsx
<Text variant="body5-strong">{t(Skills.CREATE_PAGE_TITLE)}</Text>
<Text variant="body2" color="black-800">{t(Skills.TITLE_LABEL)}</Text>
```

**Не** задавай `font-size` / `font-weight` / `font-family` в новом module, чтобы «подогнать заголовок». Нужен другой вид — другой `variant` или согласованное расширение `Text`, не локальный px.

Обрезка строк: проп `maxRows` (1–4), не свой `-webkit-line-clamp` в каждом модуле.

## Токены

На YEAHUB — `src/app/styles/variables/`. В другом Modules-проекте — откуда соседи уже берут `var(--*)`.

| Файл | Зачем |
| --- | --- |
| `pallete.css` | сырая палитра `--color-purple-700`, `--color-black-800`, … |
| `themes.css` | семантические `--background-block`, `--text-color-primary`, `--border-color`, … |
| `main-variables.css` | контейнер, шрифтовые размеры/веса, тени, z-index |
| `font-families.css` | `--font-family-code` |
| `button-colors.css` | цвета кнопок |

В module:

```css
.panel {
  background-color: var(--background-block);
  color: var(--text-color-bright);
}
```

Для текста в TSX предпочти `Text color="…"`, не дублируй цвет в CSS. Семантические токены темы (`--background-block`) лучше сырого `--color-white-900`, если смысл — «фон карточки».

## Stylelint

Если в репозитории есть stylelint — соблюдай его. На YEAHUB файл: [`.stylelintrc.json`](../../../../.stylelintrc.json).

### `declaration-strict-value`

Только `var(--*)` (или ignore-значения) для:

`color`, `background-color`, `font-size`, `font-weight`, `font-family`, `filter`

Ignore: `currentColor`, `unset`, `inherit`, `initial`, `transparent`.

```css
/* надо */
color: var(--text-color-primary);
font-size: var(--font-size-p-m);
background-color: var(--background-block);

/* не надо — упадёт lint */
color: #400799;
font-size: 16px;
background-color: white;
```

`border-color` и `box-shadow` в strict-списке нет, но новые значения всё равно бери из токенов (`--border-color`, `--main-shadow`, `--focus-shadow`), не новый hex.

### `order/properties-order`

Порядок свойств в правиле (сжато, как в конфиге):

1. `position` / inset / `z-index`
2. `display`, grid-*, flex-*, `justify-*`, `align-*`, `gap`
3. `order`, float
4. `margin`, `padding`
5. `box-sizing`, размеры (`width` / `height` / min / max)
6. `overflow`, `clip`
7. `box-shadow`, `outline`, `border*`
8. `background*`, `color`
9. текст / `font*`
10. `transform`, `transition`, `animation`

Не сортируй «как привычнее» (`width` перед `display`). Отступ — 2 пробела (`indentation: 2`).

## Куда класть блок

Ниже — если в репозитории FSD как на YEAHUB. Иная структура (`app/`, `components/`, feature folders) — клади блок как соседи, не навязывай FSD.

```
src/app        — стили, лейауты, провайдеры (токены уже здесь)
src/pages      — страница: собирает widgets/features, мало своей вёрстки
src/widgets    — составной блок экрана (таблица+обвязка, шапка секции)
src/features   — действие пользователя (создать скилл, фильтр)
src/entities   — карточка/форма сущности
src/shared/ui  — примитивы (Flex, Text, Card, Button, FormField, Table, …)
```

- Страница **собирает**, не реализует заново таблицу и фильтры.
- Новый блок — в слой по смыслу. Не клади бизнес-UI в `shared/ui`. Не тащи виджет в `shared`.
- Импорты вниз по слоям: `pages` → `widgets` / `features` / `entities` / `shared`. Не импортируй `pages` из `features`.
- Импорт примитива — публичный API слайса (`@/shared/ui/Button`, `@/widgets/SearchSection`), не глубокий путь к файлу внутри слайса.
- Сначала `Grep`/`Glob` по `Flex`, `Text`, `Card`, `Button`, `FormField`, `PageWrapper`, `SearchSection`, `Table`. Нет примитива — не копируй UI-кит из другого проекта.
- Иконки: существующий SVG из `@/shared/assets/icons/` или компонент из `@/shared/ui/Icons`. Не рисуй SVG «на глаз» и не клади новую иконку в `shared/ui/Icons`, если достаточно файла в `assets`.

Канон формы: `Flex` column `gap="24"` + header + `Card` + поля на `FormField` (см. `SkillCreateForm` + `SkillForm`). Паттерны страниц — [patterny.md](patterny.md).

## i18n: пользовательские строки

Если в проекте нет i18n — не внедряй. Если есть — как у соседей. Ниже канон YEAHUB (`react-i18next` + enum + ru/en JSON).

```tsx
import { useTranslation } from 'react-i18next';
import { i18Namespace, Skills } from '@/shared/config';

const { t } = useTranslation(i18Namespace.skill);
<Text variant="body4">{t(Skills.TITLE_FULL)}</Text>
```

- Namespace из `i18Namespace` (`src/shared/config/i18n/i18nNamespaces.ts`).
- Ключи — константы неймспейса в `src/shared/config/i18n/i18nTranslations.ts` (`Skills.*`, `Translation.*`), не сырая строка `'Create skill'` в JSX.
- Новый ключ — в enum **и** в `public/locales/ru/<ns>.json` и `public/locales/en/<ns>.json`. Enum `'stub.empty.users.title'` в JSON — вложенно: `{ "stub": { "empty": { "users": { "title": "…" } } } }`.
- `aria-label`, `placeholder`, `title`, текст кнопки — тоже `t(...)`.
- Технические `data-testid` и имена CSS-классов переводить не нужно.

## Антипаттерны трека

| Не надо | Надо |
| --- | --- |
| CSS-класс только ради простой layout-обёртки | `<Flex gap="16">`, если он не добавляет лишний DOM |
| Три вложенных flex вместо сетки | `display: grid` в module |
| `font-size: 20px` / `color: #400799` | `<Text variant color>` + `var(--*)` |
| `clsx` / `cn` / `cva`, если в репо `classnames` | `classnames` |
| `.cardHeader` в CSS | `.card-header` + `styles['card-header']` |
| `composes: card from '…'` | компонент или токен |
| `'Сохранить'` в JSX | `t(...)` |
| Tailwind-классы «на один блок» | этот трек, без Tailwind |
| Новый `Button` с нуля | `@/shared/ui/Button` |

## Чеклист трека

- [ ] `*.module.css` как у соседей (на YEAHUB — рядом, kebab-case, `classNames`)
- [ ] Нет чужого стека (Tailwind, `cn`/`cva`, если их нет в репо)
- [ ] 1D — layout-примитив с `gap`, если он есть; иначе flex как у соседей
- [ ] 2D — `display: grid` в module
- [ ] Текст и цвет — примитив / `var(--*)`, не hex/px
- [ ] Линтер стилей соблюдён, если он есть
- [ ] Слой/папка как в этом репо, примитивы reused
- [ ] Пользовательские строки через i18n, если i18n уже есть
