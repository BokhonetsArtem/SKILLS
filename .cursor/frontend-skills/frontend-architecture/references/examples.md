# Примеры: как надо и как не надо

Читай, когда неясно, куда класть код или как провести границу. Пары короткие: **надо** / **не надо** / **почему** / **как исправить**.

- Деревья и импорты — **иллюстрация**. Копируй только колонки своего трека. Нет слоёв FSD — не копируй `entities/` / `widgets/`. Нет фиче-папок — не заводи `src/features/`. **Layered ≠ Domain:** не клей `application/` / `domain/` в layered-репо и не размазывай domain-модуль по `components/`+`services/`.
- Пути в примерах (`@/entities/thing`, `@/features/thing/createThing`) — **иллюстрация**. Не переноси их в репозиторий, где другие alias, сегменты или вложенность. Бери соседей **этого** репо.
- FSD-правила (слои, `@x`, query в entity / mutation в feature) **не** универсальны. Читай только раздел своего трека внутри пары.
- Логи (`console.log`, обвязка логгера) в эти примеры **не** добавляй.

Трек ещё не выбран — [principles-and-discovery.md](principles-and-discovery.md), сверка [evaluation.md](evaluation.md), затем спроси. Слои — [architecture-tracks.md](architecture-tracks.md). Фасады — [boundaries-and-imports.md](boundaries-and-imports.md). State/API — [state-api-and-effects.md](state-api-and-effects.md).

## 1. Куда класть новый код

Новый сценарий «создать сущность» живёт рядом с аналогом того же типа, не в page и не в shared.

### FSD

```
# надо
src/features/thing/createThing/
  index.ts
  ui/ThingCreateForm.tsx
  api/createThingApi.ts
```

```
# не надо — сценарий в page или в shared
src/pages/thingCreate/ui/CreateForm.tsx
src/shared/ui/ThingCreateForm/ThingCreateForm.tsx
src/processes/createThing/          ← не создавай слой, если его нет в репо
```

**Почему.** Page собирает экран, shared не знает домен «thing». Слой `processes/` в многих FSD-репо отсутствует.

**Как исправить.** Положи форму и mutation в `features/<domain>/<action>` как у соседа; page только импортирует public API фичи.

### Feature / module

```
# надо
src/features/thing/
  index.ts
  CreateThingForm.tsx
  api.ts
```

```
# не надо — новые FSD-слои «чтобы было правильнее»
src/entities/thing/ui/ThingForm.tsx
src/features/createThing/
src/widgets/ThingCreate/
```

**Почему.** В этом треке единица — фича, не FSD-слой. Новые `entities/` / `widgets/` — второй стиль.

**Как исправить.** Колоцируй UI и API внутри существующей фичи `thing` (или создай `features/thing`, если фичи ещё нет). Слои FSD не заводи.

### Layered

```
# надо
src/screens/ThingCreatePage.tsx   ← wiring
src/components/ThingCreateForm.tsx
src/services/thingApi.ts          ← или hooks/useCreateThing — как у соседа
```

```
# не надо
src/thing/application/createThing.ts   ← это Domain, второй стиль
src/entities/thing/
src/pages/CreateThingPage.tsx          ← HTTP + форма + правила в одном файле, если соседи уже вынесли сервис
```

**Почему.** В технических слоях сценарий живёт в services/hooks, UI — в components. `application/` сюда не переносить.

**Как исправить.** Форму — к соседним компонентам, запрос — в `services`/`hooks`. Страница собирает.

### Domain

```
# надо
src/thing/
  application/createThing.ts
  ui/CreateThingForm.tsx
  data/thingApi.ts
```

```
# не надо
src/features/thing/createThing/
src/pages/CreateThingPage.tsx    ← вся логика создания внутри страницы
```

**Почему.** Сценарий — application, UI — presentation, запрос — data. Страница только wiring.

**Как исправить.** Вынеси use-case и форму в модуль домена; page/container вызывает фасад модуля.

---

## 2. Feature vs entity / domain

Карточка и чтение справочника ≠ сценарий «создать / удалить».

### FSD

```
# надо
src/entities/thing/          ← тип, карточка, query списка
  ui/ThingCard.tsx
  api/thingApi.ts            ← getThing, getThingList
src/features/thing/createThing/   ← форма + mutation
```

```
# не надо
src/entities/thing/ui/ThingCreateForm.tsx
src/features/thing/ThingCard.tsx
```

**Почему.** Entity владеет сущностью и чтением. Feature владеет действием пользователя.

**Как исправить.** Карточку и `useGetThingQuery` оставь в entity; форму создания и `useCreateThingMutation` перенеси в feature.

### Feature / module

```
# надо — и карточка, и создание внутри фичи
src/features/thing/
  ThingCard.tsx
  CreateThingForm.tsx
  api.ts
```

```
# не надо — выдуманный слой entities
src/entities/thing/ui/ThingCard.tsx
src/features/thing/CreateThingForm.tsx
```

**Почему.** Без FSD-слоёв «entity vs feature» не из чего соблюдать. Дробление на `entities/` — миграция, её не начинай.

**Как исправить.** Верни карточку в фичу. Общий тупой UI без домена — в `shared`/`ui` **этого** репо, если так у соседей.

### Layered

```
# надо — колокация по техническому слою, не entity vs feature
src/components/ThingCard.tsx
src/components/ThingCreateForm.tsx
src/services/thingApi.ts
```

```
# не надо
src/entities/thing/
src/thing/domain/Thing.ts
```

**Почему.** Нет слоёв FSD и нет внутренних `domain/`/`application/`. Карточка живёт в components, запрос — в services.

**Как исправить.** Оставь оба UI в `components/`, API в `services/`. Не выдумывай entities и не переходи на Domain.

### Domain

```
# надо
src/thing/domain/Thing.ts
src/thing/application/createThing.ts
src/thing/ui/ThingCard.tsx
```

```
# не надо — карточка в components/, правила в pages/
src/components/ThingCard.tsx
src/pages/thing/create.ts      ← доменные правила на странице
```

**Почему.** Тип и правила — domain; сценарий — application; вид — ui модуля.

**Как исправить.** Перенеси тип вниз в domain, сценарий в application, карточку в ui того же модуля.

---

## 3. Shared как свалка

`shared` / `common` / `lib` — инфраструктура без доменного смысла, не «всё переиспользуемое».

### FSD

```
# надо
src/shared/ui/Button/
src/shared/config/
src/features/thing/deleteThing/ui/DeleteThingButton.tsx
```

```
# не надо
src/shared/ui/DeleteThingButton/DeleteThingButton.tsx
src/shared/api/createThing.ts
src/shared/model/thing.ts
```

**Почему.** Кнопка удаления thing знает домен и сценарий. Shared от этого толстеет и тянет верхние слои.

**Как исправить.** Доменный контрол — в feature; типы thing — в entity; в shared оставь кит и base client.

### Feature / module

```
# надо
src/features/thing/DeleteThingButton.tsx
src/shared/ui/Button.tsx          ← только если такой кит уже есть
```

```
# не надо
src/shared/components/DeleteThingButton.tsx
src/lib/thing/createThing.ts
```

**Почему.** Переиспользование не оправдывает свалку домена в shared.

**Как исправить.** Верни в фичу `thing`. В shared — только то, что не знает thing.

### Layered

```
# надо
src/components/Button.tsx
src/services/thingApi.ts
src/utils/formatDate.ts
```

```
# не надо
src/utils/thingHelpers.ts              ← DTO и правила thing
src/thing/ui/DeleteThingButton.tsx     ← Domain в layered-репо
```

**Почему.** `utils` без домена. Кнопка удаления — рядом с формой или в components. Не переезжай в `src/thing/`.

**Как исправить.** Хелперы thing — в сервисе или рядом с формой. В `utils` — format/parse без знания thing.

### Domain

```
# надо
src/thing/ui/DeleteThingButton.tsx
src/lib/http/client.ts
```

```
# не надо
src/utils/thingHelpers.ts         ← DTO и правила thing
src/components/ThingCard.tsx      ← доменная карточка «для всех»
```

**Почему.** `utils` без домена; доменные хелперы живут в модуле thing.

**Как исправить.** Хелперы и карточку — в `src/thing/`. `lib` не импортирует thing.

---

## 4. Public API и deep imports

Снаружи модуля — фасад, не файл из `ui/` / `api/` / `internal/`.

### FSD

```ts
// надо
import { ThingCard, useGetThingListQuery } from '@/entities/thing';
import { ThingCreateForm } from '@/features/thing/createThing';

// не надо
import { ThingCard } from '@/entities/thing/ui/ThingCard/ThingCard';
import { useCreateThingMutation } from '@/features/thing/createThing/api/createThingApi';
```

**Почему.** Deep import ломает границу слайса и обходит enforcement (`steiger` `no-public-api-sidestep`, boundaries), если он включён.

**Как исправить.** Экспортируй нужное из корневого `index.ts` слайса и переведи потребителя на фасад. Не копируй alias `@/entities/thing` в репо без этого alias.

### Feature / module

```ts
// надо
import { Cart, useCart } from '@/features/cart';

// не надо
import { CartItem } from '@/features/cart/components/CartItem';
import { addToCart } from '@/features/cart/api/addToCart';
```

**Почему.** Фича B не должна знать внутренности фичи A.

**Как исправить.** Реэкспорт публичного из `features/cart/index.ts` (или того фасада, что уже импортируют соседи). Нет баррелей в репо — импортируй тот файл, который **уже** считается фасадом, не заводи `index.ts` во всём дереве.

### Layered

```ts
// надо — как у соседей: файл слоя или barrel пакета
import { createThing } from '@/services/thingApi';
import { ThingCard } from '@/components/ThingCard';

// не надо
import { thingClient } from '@/services/http/internal/thingClient';
```

**Почему.** Deep import во внутренности сервиса обходит принятый фасад. Не выдумывай `src/thing` barrel в layered-репо.

**Как исправить.** Импортируй публичный файл слоя, который уже импортируют соседи. Несколько `exports` пакета — все описанные, не любой internal.

### Domain

```ts
// надо
import { CreateThingForm, createThing } from '@/thing';

// не надо
import { thingClient } from '@/thing/data/http/thingClient';
import type { ThingDto } from '@/thing/data/mappers/mapThing';
```

**Почему.** Соседний домен зависит от фасада, не от HTTP-адаптера.

**Как исправить.** Экспортируй use-case и UI с корня модуля; мапперы оставь внутри data.

---

## 5. Циклические зависимости

Два модуля одного уровня не импортируют друг друга целиком.

### FSD

```ts
// надо — узкий канал, если он уже есть в репо
import type { Thing } from '@/entities/thing/@x/topic';

// не надо
import type { Thing } from '@/entities/thing';
import type { Topic } from '@/entities/topic'; // из entities/thing
```

```
# не надо чинить цикл так
entities/thing → features/thing/createThing → entities/thing
entities/thing/ui/ThingCard → '@/entities/topic/ui/TopicBadge'  // deep import
```

**Почему.** Слайсы одного слоя + полный public API друг друга = цикл. Deep import цикл не лечит.

**Как исправить.** Если в репо есть `@x/<consumer>` — узкий экспорт **для этого** потребителя. Нет `@x` — не выдумывай: вынеси общий тип вниз в shared/контракт, как у соседей. Не отключай правило цикла.

### Feature / module

```
# надо — общий контракт вниз
src/features/cart/     →  src/shared/types/money.ts
src/features/checkout/ →  src/shared/types/money.ts

# не надо
features/cart/api.ts    import { useCheckout } from '@/features/checkout'
features/checkout/api.ts import { useCart } from '@/features/cart'
```

**Почему.** Фичи не должны замкнуть граф. Общий тип — не повод тащить чужой сценарий.

**Как исправить.** Общий кусок вниз в shared/lib. Сценарий вызывает соседа через его public API в одну сторону (checkout → cart), не наоборот. `@x` не изобретай.

### Layered

```
# надо
screens → components → hooks → services
utils не импортирует screens и components

# не надо
services/thingApi import from screens/ThingPage
utils/format import from components/ThingCard
```

**Почему.** Нижний технический слой не знает верхний. Цикл screens↔services ломает направление.

**Как исправить.** Общий тип — в `types/` или `lib` без UI. Сервис не импортирует страницу. Не чини цикл переездом в Domain.

### Domain

```
# надо — оба зависят от domain-типа
src/catalog/domain/Sku.ts
src/billing/application/charge.ts  →  import type { Sku } from '@/catalog'  // фасад, если репо так делает
```

```
# не надо
catalog/ui → billing/ui → catalog/data
```

**Почему.** UI/data не должны ходить по кругу. Домен не знает HTTP соседнего модуля.

**Как исправить.** Общий тип в domain/фасаде; инфраструктура реализует порт. Не импортируй страницы из data.

---

## 6. Ответственность page / container

Страница собирает, не реализует таблицу, форму и API заново.

### FSD

```
# надо
pages/thing/list/ui/ThingTablePage.tsx   →  widgets + features + entity query hook
features/thing/createThing/              →  форма
entities/thing/                          →  useGetThingListQuery
```

```tsx
// надо — композиция (схема, не копипаста путей из примеров)
const { data } = useGetThingListQuery(params);
return (
  <PageShell>
    <ThingFilters />
    <ThingTable rows={data} />
  </PageShell>
);
```

```tsx
// не надо — page как god-container
export const ThingTablePage = () => {
  const [rows, setRows] = useState([]);
  useEffect(() => { fetch('/api/things').then(setRows); }, []);
  return <table>{/* разметка колонок, фильтры, мутации */}</table>;
};
```

**Почему.** В FSD-репо, где page уже композирует, новая логика на странице размазывает границы. Query сущности со страницы **допустим**, если так делают соседние pages — не запрещай «по чистоте».

**Как исправить.** Фильтры — feature, таблица — widget или ui страницы как у аналога, список — entity query. Не копируй `PageWrapper` / `SearchSection` в репо, где их нет.

### Feature / module

```
# надо
pages/CartPage.tsx  →  import { Cart } from '@/features/cart'
features/cart/      →  список, хуки, API
```

```
# не надо
pages/CartPage.tsx  →  fetch, форма, расчёт скидки, разметка таблицы
```

**Почему.** Фича владеет сценарием; страница — точка маршрута.

**Как исправить.** Перенеси состояние и API в `features/cart`, page оставь обёрткой маршрута.

### Layered

```
# надо
screens/ThingList.tsx  →  useThingList() / services
components/ThingTable.tsx
```

```
# не надо
screens/ThingList.tsx  →  HTTP + таблица + все правила, если соседи уже вынесли сервис
thing/application/list.ts  ← Domain в layered-репо
```

**Почему.** Экран собирает. Fetch и правила — в hooks/services, как у соседа. Не требуй domain/application.

**Как исправить.** Вынеси запрос в сервис/хук; таблицу — в components. Страница wiring.

### Domain

```
# надо
screens/ThingList.tsx     →  useThingList() из модуля
thing/application/list.ts →  сценарий
thing/ui/ThingTable.tsx
```

```
# не надо
screens/ThingList.tsx     →  HTTP + таблица + правила фильтрации
```

**Почему.** Container wiring, не место доменных правил.

**Как исправить.** Правила — в domain/application; экран вызывает хук фасада.

---

## 7. Доменный UI в shared

Тот же антипаттерн, что свалка, но про виджеты: селект сущности — не кит.

### FSD

```
# надо
src/entities/thing/ui/ThingSelect/ThingSelect.tsx
src/shared/ui/Select/Select.tsx
```

```
# не надо
src/shared/ui/ThingSelect/ThingSelect.tsx
src/shared/ui/ThingCard/ThingCard.tsx
```

```ts
// надо
import { ThingSelect } from '@/entities/thing';
import { Select } from '@/shared/ui/Select';

// не надо
import { ThingSelect } from '@/shared/ui/ThingSelect';
```

**Почему.** `ThingSelect` знает сущность. Кит — бездоменный `Select`.

**Как исправить.** Составной селект — в entity; примитив остаётся в shared. Образец — ближайшая entity той же зоны, не кит.

### Feature / module

```
# надо
src/features/thing/ThingSelect.tsx
src/components/Select.tsx   ← если кит уже там
```

```
# не надо
src/components/ThingSelect.tsx
```

**Почему.** Доменный селект принадлежит фиче, не общему каталогу компонентов.

**Как исправить.** Перенеси в фичу; общий Select оставь в ките репо.

### Layered

```
# надо
src/components/ThingSelect.tsx   ← рядом с другими селектами сущностей
src/components/Select.tsx        ← примитив
```

```
# не надо
src/utils/ThingSelect.tsx
src/thing/ui/ThingSelect.tsx     ← если доменных модулей нет
```

**Почему.** В layered доменный селект живёт в components, примитив — тоже. Не переноси в `src/thing/ui`.

**Как исправить.** Оставь ThingSelect в components; не клади его в utils.

### Domain

```
# надо
src/thing/ui/ThingSelect.tsx
src/ui/Select.tsx
```

```
# не надо
src/components/ThingSelect.tsx
```

**Почему.** Presentation домена живёт в модуле домена.

**Как исправить.** Верни селект в `thing/ui`.

---

## 8. Владелец state

Сначала вид state, потом место. Не всё в глобальный store.

### FSD

```tsx
// надо — список с сервера: entity query; drawer: локально
const { data } = useGetThingListQuery(params);
const [open, setOpen] = useState(false);

// не надо — server state в Redux slice «на всякий случай»
dispatch(setThings(await fetchThings()));
const things = useSelector((s) => s.things.items);
```

**Почему.** В репо с query-клиентом список уже в кэше. Дубль в slice расходится с сервером. UI-флаг не обязан быть глобальным.

**Как исправить.** Чтение — хук entity (если так у соседей). Флаг UI — в компоненте/widget. Форма — в feature, не в `shared`.

### Feature / module

```tsx
// надо
// features/thing/useThingList.ts — query фичи
// CreateThingForm.tsx — form state локально

// не надо
src/store/thingSlice.ts     ← весь продукт пишет сюда
src/store/isDrawerOpen.ts   ← одно поле — один файл стора
```

**Почему.** Фича владеет своим серверным и форм-состоянием. God-store и дробление store на поле — оба края.

**Как исправить.** Хуки и form state верни в фичу. Глобально — только то, что уже глобально у соседей (session), тем же клиентом.

### Layered

```
# надо
services/thingApi.ts или hooks/useThingList.ts   ← server
components/ThingFilters.tsx                      ← UI state
```

```
# не надо
store/index.ts  →  things, filters, drawer, form, session
thing/application/submitThing.ts  ← Domain в layered-репо
```

**Почему.** Server — в сервисе/хуке. UI-флаг — в компоненте. Не заводи application-слой.

**Как исправить.** Верни список в существующий data-fetching слой. Не добавляй второй state manager.

### Domain

```
# надо
thing/data/thingRepository.ts     ← server
thing/ui/ThingFilters.tsx         ← UI state рядом с UI
thing/application/submitThing.ts  ← form submit
```

```
# не надо
store/index.ts  →  things, filters, drawer, form, session
```

**Почему.** Один store на всё скрывает владельца.

**Как исправить.** Server — data; сценарий записи — application; UI-флаги — ui. Не добавляй второй state manager.

---

## 9. Владелец API: query vs mutation

**Отдельная пара.** Сплит «query в entity / mutation в feature» — правило **FSD-репозиториев, где так уже сделано**. На feature-module, layered и domain его **не** внедряй.

### FSD (только если аналоги так делают)

```
# надо
entities/thing/api/thingApi.ts
  build.query + providesTags

features/thing/createThing/api/createThingApi.ts
  build.mutation + invalidatesTags + onQueryStarted
```

```ts
// надо — чтение у сущности
getThingList: build.query({ query: ..., providesTags: [Tags.THINGS] });

// надо — запись у сценария
createThing: build.mutation({
  query: ...,
  invalidatesTags: [Tags.THINGS],
  async onQueryStarted(_, { queryFulfilled, extra }) { /* навигация, toast — как у соседа */ },
});
```

```
# не надо
entities/thing/api/thingApi.ts     →  createThing mutation
features/thing/createThing/        →  getThingList query
shared/api/thingApi.ts             →  и query, и mutation
```

**Почему.** Чтение и теги кэша принадлежат сущности; изменение и side effect — сценарию. Смешение в entity раздувает её; query в feature дублирует справочник.

**Как исправить.** Перенеси query в entity API, mutation в feature API, оба через существующий `baseApi`/клиент репо. Не копируй `@/shared/config` и `ApiTags` в чужой проект.

### Feature / module — не вынимай entity API

```
# надо
src/features/thing/api.ts    ← getThingList и createThing вместе, как у соседей
```

```
# не надо — «правильный FSD»
src/entities/thing/api.ts
src/features/thing/createThing/api.ts
```

**Почему.** Нет слоя entities — некуда вынимать query. Сплит создаст второй стиль.

**Как исправить.** Оставь query и mutation в фиче. Общий HTTP-клиент — там, где он уже лежит (`shared/api`, `lib/http`).

### Layered

```
# надо
src/services/thingApi.ts    ← get и create вместе, как у соседей
```

```
# не надо
src/entities/thing/api.ts
src/features/thing/createThing/api.ts
thing/data/thingApi.ts + thing/application/createThing.ts
```

**Почему.** В layered нет слоя entities и нет application. Сплит FSD или переезд в Domain — второй стиль.

**Как исправить.** Оставь query и mutation в сервисе/хуке. Общий HTTP-клиент — там, где он уже лежит.

### Domain — data + application, не FSD-сплит

```
# надо
thing/data/thingApi.ts              ← HTTP: get и create
thing/application/listThings.ts
thing/application/createThing.ts
```

```
# не надо
src/entities/thing/api/thingApi.ts
src/features/thing/createThing/api.ts
```

**Почему.** Запросы — infrastructure/data; оркестрация — application. Имена FSD здесь чужие.

**Как исправить.** Не создавай `entities/` / `features/`. UI вызывает application, application — порт data.

---

## 10. God-module

Один модуль не смешивает несвязанные сущности и сценарии.

### FSD

```
# надо
entities/thing/
entities/topic/
features/thing/createThing/
features/topic/createTopic/
```

```
# не надо
entities/catalog/     ← thing + topic + user + cart
  api/everythingApi.ts
  ui/ThingCard.tsx
  ui/UserBadge.tsx
```

**Почему.** Правка карточки thing ломает user. Импорты становятся «из catalog», граница пустая.

**Как исправить.** Разрежь по сущностям и сценариям как у соседних слайсов. Не в один diff весь catalog — [refactoring.md](refactoring.md).

### Feature / module

```
# надо
features/cart/
features/checkout/
features/account/
```

```
# не надо
features/shop/   ← корзина, оплата, профиль, админка, блог
```

**Почему.** Фича «всё приложение» = папка без границы.

**Как исправить.** Выдели сценарии, которые меняются отдельно. Общий кит — вниз, не внутрь shop.

### Layered

```
# надо
src/services/thingApi.ts
src/services/userApi.ts
src/components/ThingCard.tsx
```

```
# не надо
src/services/api.ts          ← все эндпоинты продукта
src/catalog/ + src/billing/  ← DDD в layered-репо без запроса
```

**Почему.** God-service скрывает владельца. Нарезка на domain-модули — другой трек, её не начинай «заодно».

**Как исправить.** Разнеси эндпоинты по файлам сервисов той же технической сетки. Фасады сохрани. FSD-имена не наклеивай.

### Domain

```
# надо
src/catalog/
src/billing/
src/identity/
```

```
# не надо
src/services/api.ts     ← все эндпоинты продукта
src/app/Domain.ts       ← все типы
```

**Почему.** Один модуль «все типы и все эндпоинты» скрывает bounded context.

**Как исправить.** Разнеси по доменам, сохрани фасады. Не наклеивай FSD-имена слоёв и не возвращай всё в глобальный `services/api.ts`.

---

## 11. Чрезмерное дробление

Не каждый заголовок формы — новый слайс/фича/модуль.

### FSD

```
# надо — шапка формы внутри того же createThing
src/features/thing/createThing/
  ui/ThingCreateForm.tsx
  ui/ThingCreateFormHeader.tsx
  lib/validation/thingCreateSchema.ts
```

```
# не надо
src/features/thing/createThing/
src/features/thing/createThingHeader/
src/features/thing/createThingValidation/
src/widgets/thing/ThingCreateFormHeader/
```

**Почему.** Куски одного сценария, которые всегда меняются вместе, не заслуживают своей границы. Лишние слайсы плодят циклы и `insignificant-slice`.

**Как исправить.** Верни файлы внутрь существующего слайса. Новый слайс — только новый сценарий или новая сущность (см. [principles-and-discovery.md](principles-and-discovery.md) § 4).

### Feature / module

```
# надо
src/features/cart/
  CartBadge.tsx
  CartPage.tsx
  useCart.ts
```

```
# не надо
src/features/cart/
src/features/cartBadge/
src/features/useCart/
```

**Почему.** Badge без корзины бессмыслен. Это не отдельная фича.

**Как исправить.** Собери обратно в `cart`. Дроби, только если появился отдельный пользовательский сценарий.

### Layered

```
# надо
src/services/thingApi.ts          ← create и update вместе
src/components/ThingForm.tsx
src/components/ThingFormHeader.tsx
```

```
# не надо
src/services/thingCreateApi.ts
src/services/thingUpdateApi.ts
src/thing-header/
```

**Почему.** Файл на функцию дробит cohesiveness. Шапка формы — не отдельный сервис.

**Как исправить.** Собери обратно в `thingApi` и соседние components. Не нарезай `src/thing-create/`.

### Domain

```
# надо
src/thing/application/createThing.ts
src/thing/application/updateThing.ts
```

```
# не надо
src/thing-create/
src/thing-update/
src/thing-header/
src/thing-schema/
```

**Почему.** Модуль на функцию дробит cohesiveness и заставляет модули импортировать друг друга.

**Как исправить.** Несколько use-case одного домена — в одном модуле `thing`.

---

## Чеклист примеров

Читай пункты своего трека, не все сразу.

- [ ] Новый сценарий не лежит в page/shared и не создаёт слой, которого нет в репо (`processes/`, `entities/` на feature-module, `application/` на layered)
- [ ] Карточка/чтение и «создать/удалить» разведены **так, как в этом треке**, не «как в статье про FSD»
- [ ] В shared/common/lib нет доменного UI и доменного API
- [ ] Потребитель импортирует описанный public API (barrel / exports / subpath), не `ui/` / `api/` / `internal` вглубь
- [ ] Нет нового цикла; `@x` только если канал уже есть
- [ ] Page/container композирует, не реализует таблицу+fetch+форму с нуля (если аналоги уже композируют)
- [ ] Server state не дублируется в клиентский store без образца; UI-флаг не глобален без нужды
- [ ] Query/mutation сплит entity/feature — только на FSD с таким аналогом
- [ ] Нет god-module и нет слайса на каждый подфайл формы
- [ ] Layered не получил `domain/`+`application/`; Domain не размазан по глобальным `components/`+`services/`
- [ ] В репозиторий не уехали чужие alias/пути из примеров (`@/entities/thing`), если в репо другие
