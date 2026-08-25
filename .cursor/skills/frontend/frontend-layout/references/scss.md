# Трек SCSS / Sass / Less / обычный CSS

Читай после [pravila.md](pravila.md) **только если** в репозитории уже есть Sass/Less или обычный CSS без Tailwind. Это не дефолт скила.

Два разных случая — не путай:

1. **SCSS Modules** (`*.module.scss`) — CSS Modules, написанные на Sass. Читай ещё [css-modules.md](css-modules.md). Не переводи на `.module.css`.
2. **Глобальный SCSS** (без `.module.`) — только этот файл. Если рядом доминируют `*.module.css` — **не** применяй этот файл и **не** добавляй Sass.

Не добавляй логирование в вёрстку и в примеры.

## Когда этот трек

| Сигнал | Что это |
| --- | --- |
| `*.scss` / `*.sass`, зависимость `sass` / `sass-embedded` | Sass/SCSS |
| `*.less`, `less` | Less |
| `*.module.scss` | SCSS Modules — этот файл + [css-modules.md](css-modules.md). Не подменяй на Tailwind и не сплющивай в `.module.css` |
| Много `*.css` без `.module.`, импорт глобальных таблиц, нет Tailwind | обычный CSS |
| BEM-классы (`.block__elem--mod`) в разметке | BEM — повтори соглашение соседей |

Есть и `*.module.css`, и редкий `.scss` — не переводи всё на SCSS. Пиши как **в этой зоне**.

Код из Figma MCP (часто Tailwind) — reference по иерархии. Перенеси на классы, переменные и вложенность **этого** репо.

## Файлы и имена

- Colocation как у соседей: `Component.scss` рядом с компонентом, или `styles/`, или partials `_mixins.scss`.
- Имена классов — как в репо (BEM, kebab, camel). Не внедряй BEM, если вокруг его нет.
- Импорт: `@use` / `@forward` (Dart Sass) vs `@import` — как уже написано. Не мигрируй `@import` → `@use` «заодно».
- Вложенность: 1–2 уровня. Не строй `.page { .card { .header { .title }}}`.

```scss
// надо
.filters {
  display: grid;
  grid-template-columns: 1fr 360px;
  gap: $spacing-20;
}

// не надо — ритм на детях + глубокий nest
.filters {
  display: flex;
  .item {
    margin-right: 16px;
    .label {
      color: #400799;
      font-size: 20px;
    }
  }
}
```

## Токены

Цвет и типографика — из переменных проекта, не сырой hex/px в новом правиле.

Откуда брать — смотри соседей:

- Sass: `$color-text`, `variables.scss`, maps
- CSS custom properties: `var(--text-color-primary)` (часто вместе с SCSS)
- Less: `@color-text`

Не дублируй палитру: новый `$purple-700` не заводи, если уже есть `$color-brand` / `--color-purple-700`.

Шкала spacing — из тех же переменных. `gap` в layout может быть из шкалы (`20px`, `$spacing-20`), цвет и шрифт — всегда токен.

## Mixins и медиа

Брейкпоинты — mixins / переменные **репозитория** (`@include mobile`, `@media (width < $bp-s)`), не дефолты Bootstrap/Tailwind наугад.

```scss
.list {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: $spacing-20;

  @include below($bp-s) {
    grid-template-columns: 1fr;
  }
}
```

Не пиши mixins «Flex row», если в проекте уже есть layout-примитив или утилитарный класс.

## Обычный CSS (без препроцессора)

Те же правила: `gap`, Grid для 2D, токены `var(--*)`, порядок свойств как в stylelint если он есть. Не заводи SCSS «чтобы было удобнее вкладывать».

## Антипаттерны

| Не надо | Надо |
| --- | --- |
| Добавить Sass в репо без Sass | стек, который уже есть |
| `color: #400799` | `$token` / `var(--*)` |
| `margin-right` + `:last-child` | `gap` на родителе |
| Четверной nest ради удобства | плоские классы или BEM как у соседей |
| Tailwind-классы «на один блок» | этот трек |
| Новый `.btn` рядом с `Button` | reuse примитива |

## Чеклист

- [ ] Препроцессор или обычный CSS уже в репо
- [ ] Имена классов и colocation как у соседей
- [ ] 1D — flex + `gap`, 2D — grid
- [ ] Цвет и тип из переменных
- [ ] Медиа из брейкпоинтов проекта
- [ ] Нет второго стека «для удобства»
