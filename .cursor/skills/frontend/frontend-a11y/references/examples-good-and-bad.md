# Примеры: Плохо / Хорошо

Читай, когда нужен образец платформы, не рецепт проекта. Синтаксис — HTML; в JSX замени `for`/`class` на `htmlFor`/`className`, обработчики — на принятые в зоне. Примитивы (`Button`, `Dialog`) подставь **свои**, если они уже есть. Импорты, алиасы, пути и библиотеки отсюда в репозиторий не копируй.

Каждая пара объясняет **сломанное поведение**, не «поставь атрибут». Не создавай per-framework `examples-*`.

## 1. Кликабельный `div` против кнопки

Плохо: узел выглядит как действие, но для AT и клавиатуры это текст. Tab его не находит, Enter/Space не активируют, роль — `generic`.

```html
<div class="chip" onclick="doAction()">Save</div>
```

Хорошо: нативная кнопка даёт роль, фокус и активацию. В зоне с `Button` — тот примитив, не новый кит.

```html
<button type="button" class="chip" onclick="doAction()">Save</button>
```

`role="button"` на `div` плюс `tabindex="0"` не считается исправлением: легко забыть Space и disabled-семантику.

## 2. Иконка без имени против именованной кнопки

Плохо: единственный контент — декоративная графика. Accessible name пустой: AT скажет «кнопка», не действие.

```html
<button type="button" onclick="closeDialog()">
  <svg viewBox="0 0 16 16" width="16" height="16"><path d="M2 2 L14 14 M14 2 L2 14"></path></svg>
</button>
```

Хорошо: имя у кнопки, иконка скрыта от AT, чтобы не дублировать. Текст — через i18n зоны, если он есть.

```html
<button type="button" onclick="closeDialog()" aria-label="Close">
  <svg viewBox="0 0 16 16" width="16" height="16" aria-hidden="true"><path d="M2 2 L14 14 M14 2 L2 14"></path></svg>
</button>
```

Видимый текст внутри кнопки лучше `aria-label`. Не пиши одно видимое «Закрыть» и другое `aria-label="OK"`.

## 3. Placeholder вместо label против `label` + `id`

Плохо: подсказка исчезает при вводе и часто не становится именем. Поле без постоянного имени; после ввода AT может прочитать только значение.

```html
<input type="email" name="email" placeholder="Email">
```

Хорошо: видимая (или визуально скрытая, но в DOM) подпись связана с контролом. Placeholder, если нужен, — формат, не имя.

```html
<label for="email-field">Email</label>
<input id="email-field" type="email" name="email" placeholder="name@example.com">
```

Как прокинуть `id` через form library и сфокусировать первую ошибку — [`fields-and-accessibility.md`](../../frontend-forms/references/fields-and-accessibility.md), не копируй оттуда рецепты сюда.

## 4. Молчаливое раскрытие против `aria-expanded` + `aria-controls`

Плохо: панель появляется визуально, кнопка не сообщает состояние. AT не знает, открыто ли, и не связывает кнопку с панелью.

```html
<button type="button" onclick="panel.hidden = !panel.hidden">More</button>
<div id="extra-panel">Extra details</div>
```

Хорошо: имя остаётся действием, состояние — в `aria-expanded`, связь — через id панели.

```html
<button type="button"
        aria-expanded="false"
        aria-controls="extra-panel"
        onclick="toggleExtra(this)">
  More
</button>
<div id="extra-panel" hidden>Extra details</div>
```

В обработчике синхронизируй `hidden` и `aria-expanded`. Один `aria-expanded` без смены в коде — ложь, не a11y.

## 5. Модалка без фокуса против dialog

Плохо: оверлей перехватывает клик, клавиатура остаётся на странице под ним. Escape не закрывает, после закрытия фокус не возвращается. AT читает фон как будто диалога нет.

```html
<div class="overlay">
  <div class="sheet">
    <p>Confirm delete?</p>
    <div onclick="confirmDelete()">OK</div>
  </div>
</div>
```

Хорошо: роль диалога, начальный фокус внутрь, Escape, restore на открывшую кнопку. В зоне есть `Dialog` — используй его, свой trap не пиши.

```html
<button type="button" id="open-confirm" onclick="openConfirm()">Delete</button>

<dialog id="confirm-dialog" aria-labelledby="confirm-title">
  <h2 id="confirm-title">Confirm delete?</h2>
  <button type="button" onclick="confirmDelete()">OK</button>
  <button type="button" onclick="closeConfirm()">Cancel</button>
</dialog>
```

Открытие: `showModal()` (или примитив зоны) + фокус на заголовок/`OK`. Закрытие: Escape и кнопки возвращают фокус на `#open-confirm`. Без restore пользователь теряет место на странице.

## 6. Тихий async-статус против `role="status"`

Плохо: после загрузки текст появляется только визуально. Фокус не двигается, live region нет — пользователь клавиатуры/AT не узнает исход.

```html
<button type="button" onclick="save()">Save</button>
<p id="save-result" hidden></p>
```

```js
save().then(function () {
  var node = document.getElementById("save-result");
  node.hidden = false;
  node.textContent = "Saved";
});
```

Хорошо: вежливый статус, без перехвата чтения (`assertive` не нужен). Текст — i18n зоны. Live region уже в DOM; тот же текст, что в «Плохо», пишется в него.

```html
<button type="button" onclick="save()">Save</button>
<p id="save-result" role="status" aria-live="polite"></p>
```

```js
save().then(function () {
  document.getElementById("save-result").textContent = "Saved";
});
```

Не вешай `role="alert"` на каждый успешный тик. Не вставляй в статус сырой HTML ответа сервера — [`frontend-security`](../../frontend-security/SKILL.md).

## 7. Вредный `alt` против декоративного или содержательного

Плохо: пустой смысл (`image`, имя файла) или `alt` у декора, который дублирует соседний текст. AT слышит шум или ничего полезного.

```html
<img src="hero.jpg" alt="image">
<p>
  <img src="star.svg" alt="star icon"> Featured
</p>
```

Хорошо: содержательное изображение описывает роль на экране; декоративное — пустой `alt` или `aria-hidden`.

```html
<img src="hero.jpg" alt="Empty inbox illustration">
<p>
  <img src="star.svg" alt="" aria-hidden="true"> Featured
</p>
```

Иконка — единственный контент ссылки: имя у ссылки/`alt`, не оба с разным смыслом.

## 8. «Таблица» из `div` против семантической таблицы

Плохо: сетка стилями. AT не даёт заголовки колонок, навигацию по ячейкам и сводку «таблица, N столбцов».

```html
<div class="grid">
  <div class="row">
    <div>Name</div>
    <div>Status</div>
  </div>
  <div class="row">
    <div>Alpha</div>
    <div>Open</div>
  </div>
</div>
```

Хорошо: заголовки связаны с данными. Подпись таблицы — `<caption>`, если без неё набор неоднозначен.

```html
<table>
  <caption>Items</caption>
  <thead>
    <tr>
      <th scope="col">Name</th>
      <th scope="col">Status</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Alpha</td>
      <td>Open</td>
    </tr>
  </tbody>
</table>
```

Карточный список сущностей — это список, не таблица. Не ставь `role="table"` на сетку `div`, если можно `<table>`.

## 9. CSS-перестановка против совпадения DOM и visual order

Плохо: зритель видит «Назад, Дальше», Tab идёт «Дальше, Назад». Клавиатурный путь врёт.

```html
<div class="actions" style="display: flex; flex-direction: row-reverse;">
  <button type="button">Next</button>
  <button type="button">Back</button>
</div>
```

Хорошо: в разметке тот же порядок, что на экране. Выравнивание — `margin`/`justify-content`/`margin-inline-start`, не reverse интерактивного ряда.

```html
<div class="actions" style="display: flex; justify-content: flex-end; gap: 8px;">
  <button type="button">Back</button>
  <button type="button">Next</button>
</div>
```

Токены и `gap` при вёрстке — [`pravila.md`](../../frontend-layout/references/pravila.md) §5; здесь важен факт: Tab ≠ взгляд.

## 10. Анимация без альтернативы против `prefers-reduced-motion`

Плохо: крупное или бесконечное движение всегда. Пользователь с просьбой ОС уменьшить motion не получает альтернативы; восприятие «на глаз» без media query не доказывай как `FAIL` — см. [audit-and-fix.md](audit-and-fix.md).

```css
.banner {
  animation: shake 0.4s infinite;
}
```

Хорошо: явные keyframes глушатся запросом среды. Обёртка motion зоны — если уже есть.

```css
.banner {
  animation: shake 0.4s infinite;
}

@media (prefers-reduced-motion: reduce) {
  .banner {
    animation: none;
  }
}
```

Не удаляй все `transition` проекта. Короткое затухание opacity обычно допустимо; бесконечный shake / крупный parallax — нет.
