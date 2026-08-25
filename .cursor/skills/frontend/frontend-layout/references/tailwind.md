# Трек Tailwind

Читай после [pravila.md](pravila.md) **только если** Tailwind **уже** стек репозитория. Это не дефолт скила.

## Стоп

Нет Tailwind в репо — закрой этот файл. Не добавляй `tailwind.config`, `@tailwind`, пакет `tailwindcss`, утилиты `className="flex gap-4"` и `cn`/`cva` «для удобства». Дальше — стек, который реально найден (Modules, SCSS, CSS-in-JS, UI-kit), а не CSS Modules «на всякий случай».

Не добавляй логирование в вёрстку и в примеры.

## Когда этот трек

1. Есть `tailwind.config.js` / `tailwind.config.ts` / `tailwind.css` с `@import "tailwindcss"` / `@tailwind`.
2. В `package.json` есть `tailwindcss`, и компоненты уже пишут утилиты.
3. Hybrid: Tailwind есть **и** много `*.module.css` — layout/spacing утилитами, сложные селекторы и анимации в module (см. ниже).

Код из Figma MCP часто приходит как React + Tailwind. Это **reference**, не копипаста: перенеси ритм и иерархию, классы адаптируй под токены и примитивы проекта.

## Utility-first

Пиши раскладку утилитами на разметке, не заводи module «на каждый отступ».

```tsx
// надо
<div className="flex flex-col gap-6">
  <header className="flex items-center justify-between gap-4">
    <h1 className="text-xl font-medium text-foreground">{title}</h1>
    <Button>{action}</Button>
  </header>
</div>
```

Макро-сетка — `grid`:

```tsx
<ul className="grid grid-cols-2 gap-5 max-sm:grid-cols-1">
```

Микро-ряд — `flex` + `gap-*`, не `space-x-*` + `last:mr-0`.

Соблюдай [pravila.md](pravila.md): `gap` вместо margin-соседей, `min-w-0` на flex-детях с текстом, без `flex-row-reverse` для интерактива.

## Токены: `@theme` / `tailwind.config`, не произвольный hex

Цвета, радиусы, шрифты, spacing — из конфигурации проекта.

```tsx
// надо — шкала темы
className="bg-background text-foreground border-border gap-4"

// не надо — макетный hex в утилите
className="bg-[#FFFFFF] text-[#18003E] gap-[13px]"
```

- Tailwind v4: токены в `@theme` (CSS).
- Tailwind v3: `theme.extend` в `tailwind.config`.
- Не копируй палитру другого проекта в новый `tailwind.config`.
- `arbitrary values` (`w-[332px]`, `top-[76px]`) — только если в теме нет шкалы и значение разовое (ширина сайдбара из макета). Цвет и тип через arbitrary — нет.

Reuse примитивов проекта (`Button`, `Input`, `Card`), если они есть. Не собирай кнопку из `bg-purple-700 px-4 py-2 rounded-md` рядом с существующим `Button`.

## Состав классов: `cn` / `cva` — как в проекте

Смотри репозиторий, не привычку:

| Если в проекте | Используй |
| --- | --- |
| `cn` из `@/lib/utils` (часто `clsx` + `tailwind-merge`) | `cn('flex gap-4', className)` |
| `class-variance-authority` / `cva` | варианты кнопки/бейджа через `cva` |
| ничего из этого, только `clsx` | `clsx` |
| нет Tailwind | **не** внедряй `cn`/`cva`; состав классов — как в репо |

```tsx
import { cn } from '@/lib/utils';

className={cn('flex min-w-0 items-center gap-2', isOpen && 'bg-muted', className)}
```

Конфликт утилит (`p-4` + `p-2`) решает `tailwind-merge` внутри `cn`. Не конкатенируй строки через `+` / шаблон, если в проекте есть `cn`.

## Когда уходить в CSS Module

Оставь Tailwind, пока классы читаются. Вынеси в `*.module.css` (или `@layer` + обычный CSS), если:

- keyframes, сложная анимация, `clip-path`, градиентные маски;
- селекторы вроде `> * + *`, `thead th:not(:last-child)`, состояние родителя на глубоком потомке, которое не закрывается `group`/`peer`;
- class-soup: десятки утилит, повтор в трёх компонентах, не спасает `cva`.

Hybrid-правило: **не** мешай в одном узле длинную Tailwind-строку и module без причины. Либо утилиты на этом узле, либо один осмысленный класс из module.

```tsx
// надо — анимация в module, раскладка утилитами
<div className={cn('relative flex flex-col gap-4', styles.fadeIn)}>

// не надо — soup + дублирующий module
<div className={cn('flex flex-col gap-4 p-6 rounded-xl bg-white', styles.card)}>
```

## Адаптив

Mobile-first утилиты проекта: `sm:` / `md:` / `lg:` по брейкпоинтам **конфига**, не свои пиксели в каждом `min-[487px]:`.

Если в репозитории кастомные брейкпоинты — повтори их, не дефолт Tailwind 640/768/1024.

Container queries (`@container`, `@md:`) — только если проект уже их включает.

## i18n и семантика

Как в общих правилах: пользовательские строки не хардкодь, если в проекте есть i18n. Семантика (`main`, `button`, подписи) важнее набора `flex` классов.

## UnoCSS / Windi

Если в репо `unocss` / `windicss`, а не `tailwindcss` — этот файл читай как **атомарный** трек, но синтаксис не подменяй на Tailwind.

- Классы, attributify, `class` vs `className`, пресеты — как у соседей.
- Токены из конфига Uno/Windi (`theme`, shortcuts), не `text-[#400799]` и не новый `tailwind.config`.
- Правила [pravila.md](pravila.md) те же: `gap` на родителе, grid для 2D, `min-w-0` у текстовых flex-детей.

Не добавляй UnoCSS в Tailwind-репо и наоборот.

## Антипаттерны трека

| Не надо | Надо |
| --- | --- |
| Вставить Tailwind в репо без Tailwind | стек, который уже есть |
| `text-[#400799] bg-[#fff]` | токены `@theme` / `theme.extend` |
| `mr-4` на каждом ребёнке | `flex gap-4` на родителе |
| 25 утилит + копипаста в трёх файлах | `cva` или module |
| Keyframes классами `animate-[…]` на 15 шагов | CSS Module / `@keyframes` |
| Свой `<button className="…">` рядом с `Button` | reuse примитива |
| `flex-row-reverse` чтобы «кнопка справа» | порядок в DOM |

## Чеклист трека

- [ ] Tailwind или Uno/Windi уже есть в репозитории (иначе этот файл не применяется)
- [ ] Layout 1D — `flex` + `gap`, 2D — `grid`
- [ ] Цвет и тип из темы, не hex в arbitrary
- [ ] Состав классов как в проекте (`cn` / `cva` / `clsx`)
- [ ] Сложные селекторы и анимации — не soup, а module
- [ ] Брейкпоинты конфига
- [ ] Примитивы проекта reused
