# Клавиатура, фокус и динамика

Читай на шаге 3, когда нужны Tab/Enter/Space/Escape, focus trap/restore, roving tabindex, live regions, loading/status. Семантика и имя — [semantics-and-names.md](semantics-and-names.md). Пары — [examples-good-and-bad.md](examples-good-and-bad.md).

Scroll / focus / title **после client navigation** не копируй из [`navigation-state-and-boundaries.md`](../../frontend-routing/references/navigation-state-and-boundaries.md) §7: для route transition — §6 (дельта аудита) и отсылка к routing. Видимый focus-токен и hit-area при вёрстке — [`pravila.md`](../../frontend-layout/references/pravila.md) §6.

## 1. Клавиатурный контракт

Интерактив, который виден и кликабелен, должен быть в Tab-порядке и отвечать на ожидаемые клавиши.

| Контрол | Ожидание |
| --- | --- |
| `<button>`, `role="button"` (если уже есть, не ставить в fix) | Enter и Space активируют |
| `<a href>` | Enter активирует; Space — прокрутка страницы, не «как кнопка» |
| Нативный checkbox / radio / `<select>` | Space / стрелки по платформе |
| Disclosure / menu button | Enter/Space открывают; Escape закрывает, если панель — overlay |
| Dialog / modal | Escape закрывает, если зона так делает для соседей; Tab циклом внутри |
| Tabs / listbox / radio-group как виджет | стрелки + roving tabindex (§4), не все вкладки в Tab |

`tabIndex={0}` / `tabindex="0"` на `div` без роли и клавиатурных обработчиков — ловушка: фокус есть, действия нет. `tabIndex={>0}` не используй: ломает порядок. `tabIndex={-1}` — программный фокус (заголовок экрана, restore target), не замена кнопки.

`pointer-events` + клик при `display` содержимого, которое не в Tab — finding. Disabled нативный контрол из Tab исключён — это платформа, не finding; кастомный «disabled» на `div` с `onClick` — finding.

## 2. Видимый фокус и порядок

Фокус должен быть виден. `outline: none` / `outline: 0` без замены — гипотеза; confirmed, если в коде нет другого видимого индикатора на `:focus` / `:focus-visible`. Какой токен подставить при вёрстке — layout, не этот файл.

Порядок Tab = DOM = визуальный порядок. `order`, `row-reverse`, `column-reverse`, абсолютное переставление интерактивных узлов — finding, если пользователь видит A→B, а Tab идёт B→A. Как не верстать reverse — [`pravila.md`](../../frontend-layout/references/pravila.md) §5; здесь фиксируй факт расхождения. См. examples §9.

Skip link («к содержимому») — проверяй, если в зоне уже принят; не добавляй второй в `fix`, если chrome уже даёт переход к `main`.

## 3. Focus trap и restore

Модальный overlay (диалог, drawer, который блокирует фон) обязан:

1. при открытии перевести фокус внутрь (первый осмысленный контрол или заголовок с `tabIndex={-1}`);
2. удерживать Tab внутри, пока открыт;
3. закрываться по Escape, если соседние модалки зоны так делают;
4. вернуть фокус на элемент, который открыл.

Нет trap / нет restore / фокус улетает в `body` — confirmed, если в коде видно открытие overlay без управления фокусом. Не пиши свой trap, если в зоне есть `Dialog` / `Modal` / FocusLock-примитив: в `fix` переиспользуй. См. examples §5.

Немодальная панель (inline disclosure) trap не требует; нужен `aria-expanded` и фокус на кнопке после закрытия, если кнопка осталась.

Не ставь `autoFocus` на первый input списка при каждом ререндере фильтра — это не открытие модалки.

## 4. Roving tabindex

Для группы, где в Tab должна попадать **одна** «caret»-позиция (вкладки, toolbar, listbox), остальные элементы `tabIndex={-1}`, активный — `0`, стрелки двигают caret. Все вкладки с `tabIndex={0}` — лишний Tab-шум; только мышь без стрелок — finding.

Не навязывай roving наборy независимых кнопок в тулбаре, если соседи зоны табят каждую. Паттерн бери у ближайшего виджета той же зоны.

## 5. Live regions, loading, status

Динамика, которую зрячий видит без нового экрана, не должна быть немой для AT.

| Ситуация | Что проверить |
| --- | --- |
| Короткий статус («сохранено», счётчик, конец загрузки) | Ненавязчивый `role="status"` / `aria-live="polite"` на существующем узле |
| Критичная ошибка, требующая внимания сейчас | `role="alert"` только если зона так делает; не на каждую валидацию поля |
| Спиннер без текста | Есть имя у статуса или связанный текст; крутилка-дивижн без имени — сигнал |
| Бесконечный `aria-live="assertive"` на частый тик | finding: перекрывает чтение |

Не создавай live region на каждый toast, если в зоне уже есть announcer. Не клади unsanitized HTML в live region — [`frontend-security`](../../frontend-security/SKILL.md).

Молча появившийся текст статуса без live region и без переноса фокуса — finding, если пользователь иначе не узнает исход. См. examples §6.

`aria-busy` на регионе загрузки — допустим как дополнение, не замена статуса. Пустой DOM надолго без ориентира — связан с navigation (§6) и layout-skeleton: здесь отметь тишину AT.

## 6. Route transition: только дельта аудита

После **client** navigation (не полной перезагрузки) проверь факты:

- фокус не остался на размонтированной ссылке и не потерялся в `document.body` без ориентира;
- `document.title` (или принятый meta-хелпер зоны) сменился вместе с экраном, не одно имя приложения на всех client routes;
- AT не остаётся в тишине на долгий pending: есть loading-ориентир или route announcer зоны.

Как **сделать** `ScrollRestoration`, `Meta` / `generateMetadata`, hash-scroll и API конкретного роутера — [`navigation-state-and-boundaries.md`](../../frontend-routing/references/navigation-state-and-boundaries.md) §7. При работе над маршрутом ведёт `frontend-routing`. Не копируй эти рецепты сюда и не изобретай второй toast-logger вместо announcer зоны.

Нет announcer и нет фокуса на `main`/`h1` после перехода — a11y finding. Нет `generateMetadata` в Next-треке — routing, не «добавь `aria-live` на весь layout» в обход.
