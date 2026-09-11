# Границы forms / routing / layout и восприятие

Читай на шаге 3, когда системный a11y-запрос пересекается с формой, SPA-переходом, контрастом, motion или медиа. Это **границы и кросс-срез**, не рецепты соседей.

Не копируй:

- поля, `htmlFor`, `aria-invalid`, focus первой ошибки — [`fields-and-accessibility.md`](../../frontend-forms/references/fields-and-accessibility.md);
- hit-area, токены, DOM-порядок как вёрстку — [`pravila.md`](../../frontend-layout/references/pravila.md) §5–§6;
- scroll / focus / title API роутера — [`navigation-state-and-boundaries.md`](../../frontend-routing/references/navigation-state-and-boundaries.md) §7.

Дельта аудита полей — [semantics-and-names.md](semantics-and-names.md) §6. Дельта SPA-перехода — [keyboard-focus-and-dynamics.md](keyboard-focus-and-dynamics.md) §6. Непроверяемое восприятие — `INCOMPLETE`, правила статуса в [audit-and-fix.md](audit-and-fix.md).

## 1. Кто ведёт

Совпадает с матрицей [`SKILL.md`](../SKILL.md). Здесь — когда не забирать чужую реализацию.

| Тема | При реализации фичи | При системном a11y/WCAG-запросе |
| --- | --- | --- |
| label / ошибки поля / focus первой ошибки / submit | `frontend-forms` | этот скил: факты имени и объявления; рецепт — forms |
| сетка поля, токены, hit-area, видимый focus-стиль | `frontend-layout` | этот скил: факт отсутствия индикатора / hit-target; токен — layout |
| route tree, URL-state, scroll-restoration, title helper | `frontend-routing` | этот скил: факт тишины / потерянного фокуса после перехода |
| XSS в preview / live region / имени | `frontend-security` | всегда security; a11y только hand-off |

Запрос «сделай форму» без аудита доступности — не этот файл. Запрос «проверь доступность экрана с формой» — здесь плюс дельта, без переписывания form state.

## 2. Формы: что проверять, чего не чинить

В scope экрана с формой подтверди:

- у контролов есть имя не из одного `placeholder`;
- ошибка не только цветом;
- submit с клавиатуры (Enter в поле / кнопка `type="submit"`), не единственный `onClick` на `div`.

Не подтверждай как a11y-finding и не чини в этом скиле:

- выбор RHF vs Formik vs native, schema, `register`, dirty/leave-guard;
- параллельный `useState` на то же значение;
- визуальную колонку `FormField` без смены поведения.

Это `frontend-forms`. Если форма есть, а системного a11y-запроса нет — остановись и отдай forms.

## 3. Навигация и route announcer

Кросс-срез, не дерево маршрутов.

- Skip link / «к содержимому» — если chrome зоны его даёт, проверь цель (`main` / skip-target). Второй не добавляй.
- Смена client route без смены title и без фокуса на содержимом — finding; **как** выставить title в треке — routing §7.
- Route announcer / live region layout — переиспользуй. Нет в зоне — не изобретай глобальный `aria-live` на весь outlet «на всякий случай»; минимум фиксируется как отсутствие ориентира (фокус на `h1`/`main`), не как новый пакет.
- Фильтр того же экрана (search-param) ≠ новый экран: не требуй полный route-announce на каждый чип, если соседи объявляют только смену страницы.

Открытие модалки роутом — фокус как у dialog ([keyboard-focus-and-dynamics.md](keyboard-focus-and-dynamics.md) §3), не `autoFocus` на список фильтра.

## 4. Контраст

Смысл не только цветом: ошибка, выбранный таб, disabled vs текущий — нужен нецветовой признак (текст, иконка, форма, `aria-current`).

Числовой контраст текста/фона:

- есть измеримый токен зоны или уже стоящий инструмент, который считает contrast — можно confirmed, если значение/токен доказанно ниже принятого в зоне порога;
- «на глаз» из CSS `color` / `background` без расчёта и без токена — **не** `FAIL` и не перебивает уже поставленный `FAIL`/`WARN`. Пиши в `Limitations` / секцию `INCOMPLETE` или как неподтверждённый сигнал.

Не подбирай новую палитру и не меняй design tokens в `fix`, если запрос — a11y-аудит: либо укажи несоответствие существующему токену/компоненту зоны, либо `INCOMPLETE`. Новые цвета — `frontend-layout`.

## 5. Reduced motion

Необязательная анимация (декор, бесконечный pulse, parallax, крупный motion при входе):

- в зоне уже есть обёртка / токен motion — переиспользуй;
- в коде есть `@media (prefers-reduced-motion: reduce)` (или эквивалент) — можно проверить, что крупная анимация глушится;
- анимация есть, альтернативы и media query нет — finding, если движение задано в коде явно (keyframes, transition на transform/крупный layout);
- «пользователю будет плохо от этого easing» без измеримого токена/запроса среды — `INCOMPLETE`, не выдуманный `FAIL`.

См. examples §10. Как вписать motion в токены при вёрстке — layout §6; здесь — факт наличия/отсутствия альтернативы.

Не выключай все transition проекта. Не ставь библиотеку motion.

## 6. Медиа

`<video>` / `<audio>` / встроенный плеер в scope:

- есть управляемые контролы (play/pause с клавиатуры), не только автоплей без остановки;
- автоплей звука без явного жеста — finding;
- содержательное видео: текстовая альтернатива (субтитры/caption track) или связанная расшифровка, если зона уже так делает для соседей; нет ни дорожки, ни текста — finding, если ролик несёт речь/смысл;
- декоративный короткий loop без речи — не требуй caption.

Не вшивай сторонний caption-сервис. Не проверяй «как слышит screen reader ролик» вручную — это `INCOMPLETE` (реальный AT).

Иконка-кнопка плеера без имени — [semantics-and-names.md](semantics-and-names.md), не «медиа-рецепт».
