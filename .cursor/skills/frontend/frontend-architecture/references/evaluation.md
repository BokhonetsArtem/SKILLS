# Evaluation: классификация трека

Читай, когда шаг 2 **спорный** или правишь этот скил. На обычное размещение не грузи.

Каждый кейс: вход → ожидаемый трек → что **запрещено** рекомендовать.

## 1. FSD без enforcement

**Вход.** `src/app`, `pages`, `widgets`, `features`, `entities`, `shared`; слайсы с `index.ts`; импорты вниз по слоям. Нет `steiger` и нет `@feature-sliced/*`.

**Трек.** FSD.

**Запрещено.** Называть проект «неясным» только из-за отсутствия линтера. Спрашивать «перейти на FSD?». Добавлять steiger этим скилом.

## 2. Next.js `app/` — не FSD

**Вход.** Корневой `app/(routes)/page.tsx`, `src/components/`, `src/lib/`. Нет `entities/` / `widgets/`. Нет FSD-enforcement.

**Трек.** Layered (или feature-module, если рядом вертикальные фичи). `app/` — framework-owned.

**Запрещено.** Считать `app/` слоем FSD. Создавать `src/entities/` «для правильности». Переносить роуты Next в FSD-`app/`.

## 3. Feature folders

**Вход.** `src/features/cart/`, `src/features/auth/` с колокацией UI+API. Нет `entities/` / `widgets/`. Может быть `src/shared/ui`.

**Трек.** Feature / module.

**Запрещено.** Заводить FSD-слои. Вынимать query в `entities/`. Внедрять `@x`.

## 4. Технические слои

**Вход.** `src/pages` или `screens` + `components` + `hooks`/`services`/`store`. Нет вертикальных `domain/`+`application/` внутри модулей.

**Трек.** Layered.

**Запрещено.** Создавать `src/<domain>/application`. Наклеивать `entities/` / `features/`. Резать `services/api.ts` миграцией на DDD без запроса.

## 5. Доменные модули

**Вход.** `src/catalog/domain`, `src/catalog/application`, `src/catalog/ui`, `src/catalog/data`. Сосед `src/billing/` того же вида.

**Трек.** Domain.

**Запрещено.** Размазывать модуль по глобальным `components/`+`services/`. Внедрять FSD-сплит query/mutation. Добавлять гексагональные порты, если их нет.

## 6. Колокация ≠ Domain

**Вход.** `src/cart/Cart.tsx`, `src/cart/api.ts`, `src/cart/useCart.ts`. Внутри нет `domain/` / `application/`.

**Трек.** Feature / module.

**Запрещено.** Классифицировать как Domain. Добавлять `application/` «как в Clean Architecture».

## 7. Смешанный legacy

**Вход.** В одной зоне и `src/features/cart/`, и `src/components/CartTable` + `src/services/cartApi` без явного победителя.

**Трек.** Смешанный → **спросить**, в каком стиле вести новое место. Класть как соседи выбранной зоны.

**Запрещено.** Угадывать FSD. Создавать третий стиль (`src/entities/` рядом). Мигрировать весь `src` заодно.

## 8. Сигналов мало

**Вход.** Плоский `src/components`, пара файлов, нет слоёв, фич, доменов, enforcement.

**Трек.** Неясный → спросить → копировать ближайших соседей.

**Запрещено.** Выбирать FSD/layered/domain по привычке модели. Добавлять архитектурный линтер или второй state manager.

## 9. Пакет в monorepo

**Вход.** `packages/ui` с `package.json` `exports` (несколько subpath). Внутри — кит, не продуктовые слайсы.

**Трек.** По внутренностям пакета (часто layered или feature-module). Public API = описанные `exports`, не любой файл пакета.

**Запрещено.** Считать пакет FSD-слоем `shared`. Игнорировать `exports` и импортировать `src/internal`. Навязывать один barrel, если соседи используют несколько entrypoints.
