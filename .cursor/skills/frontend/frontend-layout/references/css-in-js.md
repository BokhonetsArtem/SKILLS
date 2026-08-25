# Трек CSS-in-JS

Читай после [pravila.md](pravila.md) **только если** в репозитории уже есть runtime или zero-runtime CSS-in-JS. Это не дефолт скила.

Не добавляй логирование в вёрстку и в примеры. Не ставь `styled-components` / Emotion / Stitches в репо, где их нет.

## Когда этот трек

Смотри `package.json` и ближайшие UI-файлы. Читай **этот** файл, не Modules/Tailwind.

| Сигнал | Библиотека |
| --- | --- |
| `styled-components`, `styled.div`, `.styled.ts` | styled-components |
| `@emotion/styled`, `@emotion/react`, проп `css={}` | Emotion |
| `@stitches/react`, `styled()`, `css()` с `variants` | Stitches |
| `@vanilla-extract/css`, `*.css.ts`, `recipe` / `sprinkles` | vanilla-extract |
| `@stylexjs/stylex`, `stylex.create` | StyleX |
| `@linaria/core`, `@linaria/react` | Linaria |
| `@pandacss/dev`, `styled-system`, `css()` из generated | Panda |

Несколько библиотек в одном репо — как у **соседей той же зоны**, не мешай `styled.div` и `css={}` в одном файле без прецедента.

Код из Figma MCP часто Tailwind. Это reference по иерархии, не копипаста: перенеси на `styled` / `css` / `.css.ts` и тему проекта.

## Общее для всех

- Цвет, шрифт, радиус, spacing — из **темы** (`ThemeProvider`, `theme.ts`, tokens), не hex в шаблонной строке.
- 1D — `display: flex` + `gap` (или layout-примитив проекта). 2D — `display: grid`.
- Не собирай кнопку/инпут рядом с существующим `Button` / `Input`.
- Состав: не внедряй `clsx`/`cn`/`cva`, если в репо их нет. Для вариантов смотри, как уже делают (`attrs`, `css` helper, `variants`, `recipe`).
- Файл стилей — как у соседей: `Component.styled.ts` рядом, или `styled` в том же файле, или `*.css.ts`. Не заводи новый расклад «как принято в туториале».

```tsx
// надо — тема + gap на родителе
const Row = styled.div`
  display: flex;
  align-items: center;
  gap: ${({ theme }) => theme.spacing[4]};
`;

// не надо
const Row = styled.div`
  display: flex;
`;
const Item = styled.span`
  margin-right: 16px;
  color: #400799;
`;
```

## styled-components

- Импорт и API — как у соседей (`styled` default vs `styled` named, v5 vs v6).
- Пропы, которые не должны уезжать в DOM — как в проекте (`$transient`, `shouldForwardProp`). Не выдумывай `$` префикс, если файлы рядом без него.
- Повторяющиеся куски — `css` helper или тема, не копипаста 20 строк в трёх `styled`.
- Не клади бизнес-логику и хуки внутрь `styled` компонента.

## Emotion

- Если соседи на `css={css\`…\`}` — не переводи файл на `styled.div`, и наоборот.
- `sx` у Emotion и `sx` у MUI — разные вещи. MUI — [ui-kit.md](ui-kit.md).
- Тема: `useTheme` / `ThemeProvider` из той же библиотеки, что у соседей (`@emotion/react` vs MUI).

## Stitches / Panda / vanilla-extract / StyleX / Linaria

Zero-runtime и token pipelines: **не** пиши runtime `styled.div` из styled-components «потому что так проще».

- Стили в `*.css.ts` / `stylex.create` / generated `css()`, не в inline `style={{ color }}`.
- Варианты — `recipe` / `variants` / `sprinkles` проекта, не десяток отдельных className.
- Атомарные утилиты Panda (`flex`, `gap="4"`) — шкала темы Panda, не Tailwind-классы и не новый `tailwind.config`.

## Антипаттерны

| Не надо | Надо |
| --- | --- |
| Добавить styled-components в репо без CSS-in-JS | стек, который уже есть |
| `color: #400799` в шаблонной строке | `theme.colors.*` / токен |
| `margin-right` на каждом ребёнке | `gap` на родителе |
| Свой `styled.button` рядом с `Button` | reuse примитива |
| Tailwind-утилиты «на один блок» | этот трек |
| `style={{ display: 'flex' }}` при живом CSS-in-JS | `styled` / `css` / `.css.ts` |

## Чеклист

- [ ] Библиотека уже в репо (иначе этот файл не применяется)
- [ ] Синтаксис как у соседей (styled vs css prop vs `.css.ts`)
- [ ] 1D — flex + `gap`, 2D — grid
- [ ] Цвет и тип из темы
- [ ] Примитивы проекта reused
- [ ] Нет второго стека «для удобства»
