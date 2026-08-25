# Трек UI-kit (MUI, Ant Design, Chakra, Mantine)

Читай после [pravila.md](pravila.md) **только если** основной UI — компонентный кит с темой. Это не дефолт скила.

Не добавляй MUI / Ant / Chakra / Mantine в репо, где их нет. Если в репо свой `shared/ui` без кита — этот файл не применяется, пока пользователь явно не просил кит.

Не добавляй логирование в вёрстку и в примеры.

## Когда этот трек

| Сигнал | Кит |
| --- | --- |
| `@mui/material`, `@mui/system`, `sx=`, `styled` из `@mui` | MUI |
| `antd`, `@ant-design`, `ConfigProvider`, `token` | Ant Design |
| `@chakra-ui/*` | Chakra |
| `@mantine/core` | Mantine |
| другой кит с темой (`ThemeProvider`, `sx`, `style` API) | тот же принцип: примитивы кита + тема |

Кит **есть**, но зона вокруг пишет CSS Modules / Tailwind — как у **соседей этой зоны**, не переводи всё на `sx`.

Код из Figma MCP (Tailwind) — reference по иерархии. Сложи в `Stack` / `Grid` / `Flex` кита и токены темы, не в утилиты Tailwind.

## Общее

- Сначала компонент кита: `Button`, `TextField`/`Input`, `Card`/`Paper`, `Table`, `Stack`/`Space`/`Flex`, `Grid`. Не свой `<button>` и не сырой `div`+CSS рядом.
- Цвет, тип, радиус, тень — из темы кита (`palette`, `token`, `theme.colors`), не `sx={{ color: '#400799' }}` и не новый hex в `styled`.
- 1D — `Stack` / `Space` / `Flex` кита с `spacing`/`gap`. 2D — `Grid` кита или CSS `display: grid`, как у соседей. Не тройной вложенный `Stack` вместо сетки.
- Кастомные стили — тем же API, что у соседей (`sx` vs `styled` vs `className` + less/css). Не мешай три способа на одном узле.
- Не подключай второй кит и не тащи shadcn/`cn` «для удобства».

```tsx
// надо
<Stack component="main" gap={3}>
  <Typography variant="h6">{title}</Typography>
  <Button variant="contained">{action}</Button>
</Stack>

// не надо
<div style={{ display: 'flex', flexDirection: 'column', gap: 24 }}>
  <h1 style={{ color: '#400799' }}>{title}</h1>
  <button className="rounded-md bg-purple-700 px-4 py-2">{action}</button>
</div>
```

## MUI

- Тема: `theme.palette` / `theme.typography` / `theme.spacing`. Новые цвета — в `createTheme`, не в компонент.
- `sx` для одноразового отступа и выравнивания. Повторяющийся блок — `styled` или слот темы, не копипаста `sx` на 15 строк в трёх файлах.
- Grid: смотри версию в репо (`Grid2` vs `Grid`). Не подменяй API v5 на v6 наугад.
- Emotion под капотом MUI — не импортируй `@emotion/styled` отдельно, если соседи берут `styled` из `@mui/material/styles`.

## Ant Design

- Токены: `theme.useToken()` / `ConfigProvider`, не less-переменные в новом файле, если проект уже на token API.
- Layout: `Space` / `Flex` / `Row`+`Col` как у соседей. Не смешивай `Row/Col` и CSS grid в одном экране без прецедента.
- Форма: `Form.Item` + китовый инпут, не placeholder вместо label.
- Кастом: `styles` / `classNames` пропы компонента (v5) или токен, не глобальный `.ant-btn` override «на глаз».

## Chakra / Mantine

- Spacing и цвет — шкалы кита (`4`, `md`, `gray.700`), не px/hex.
- `as` / polymorphic — для семантики (`as="main"`), не ради красивого тега ценой a11y.
- Responsive: массив/объект брейкпоинтов **кита**, не дефолты Tailwind.

## Когда свой CSS всё же нужен

Оставь кит, пока хватает. Вынеси в CSS / `styled` / module, если:

- keyframes, `clip-path`, сложная сетка, которой нет в ките;
- селекторы, которые `sx` не закрывает.

Не дублируй `sx={{ display: 'flex', gap: 2 }}` классом `.row { display: flex; gap: 16px }` на том же узле.

## Антипаттерны

| Не надо | Надо |
| --- | --- |
| Добавить MUI/Ant в репо без кита | `shared/ui` проекта |
| Hex в `sx` / `style` | токен темы |
| Свой flex-div рядом со `Stack`/`Space` | примитив кита |
| Tailwind-утилиты в MUI-экране | `sx` / тема |
| Глобальный override `.ant-*` / `.Mui*` без нужды | token / `styled` слота |
| Второй кит «только для таблицы» | то, что уже в репо |

## Чеклист

- [ ] Кит уже в репо (иначе этот файл не применяется)
- [ ] Компоненты кита reused, не сырой HTML+CSS
- [ ] 1D — Stack/Space/Flex кита + gap, 2D — Grid
- [ ] Цвет и тип из темы / token
- [ ] Кастом тем же API, что у соседей
- [ ] Нет второго стека «для удобства»
