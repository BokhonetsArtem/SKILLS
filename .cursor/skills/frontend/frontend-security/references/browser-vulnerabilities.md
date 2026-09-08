# Browser vulnerabilities

Читай на шаге 3. Только разделы стека из [discovery-and-scope.md](discovery-and-scope.md). Имена API не переноси с React на Vue/Svelte/Angular/vanilla.

Ищи цепочку **источник → поток → sink**. Нет источника — нет подтверждённой находки (см. [evidence-and-reporting.md](evidence-and-reporting.md)).

Описывай sink и поток словами репозитория. Не вставляй exploit-строки, payload и shell-инструкции атаки ни в отчёт, ни в чат.

## Источники (любой стек)

Недоверенными считай данные, которые код **читает** извне, не константу разработчика:

- URL: query, hash, path-сегмент, `searchParams`;
- `window.name`, `postMessage` / `MessagePort` payload, `BroadcastChannel`, WebSocket/EventSource messages;
- `IndexedDB`/storage/cookie, если значение записывает другой origin, third-party или иной недоверенный код;
- поля формы, upload, clipboard/drag-and-drop, `document.referrer`, UTM;
- server-rendered DOM/атрибуты и ответ API, если содержимое контролирует пользователь/внешний контур и без подходящей защиты идёт в sink (сам API-сервер здесь не аудируй).

Текст, который фреймворк **экранирует по умолчанию**, не флагь. Статический литерал в JSX/шаблоне без внешнего входа — не XSS.

## DOM / HTML sinks

Читай **одну** строку стека.

| Стек | Unsafe API / директива | Штатный escape (не находка сам по себе) |
| --- | --- | --- |
| React / Next | `dangerouslySetInnerHTML`; прямой DOM через `ref` (`innerHTML`, `insertAdjacentHTML`, `outerHTML`, `document.write`) | Текстовые children и обычные атрибуты React экранирует |
| Vue / Nuxt | `v-html`; прямой DOM в `el` / `useTemplateRef` теми же HTML-setter | Интерполяция `{{ }}` и обычный `v-bind` текста экранирует |
| Svelte / SvelteKit | `{@html ...}`; присвоение `innerHTML` в `onMount` / action | Текстовые узлы шаблона экранирует |
| Angular | `bypassSecurityTrustHtml` / `TrustScript` / `TrustUrl` / `TrustResourceUrl` / `TrustStyle` с недоверенной строкой; обход sanitizer | `[innerHTML]` без bypass идёт через sanitizer Angular — не флагь без bypass или другого raw sink |
| Vanilla / jQuery | `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`; jQuery `$()` с HTML-строкой, `.html()` / `.append()` / `.prepend()` / `.before()` / `.after()` со строкой | `textContent` / `.text()` |

Дополнительно, если пакет есть в манифесте: Lit `unsafeHTML` / `unsafeSVG`; Solid — присвоение `innerHTML`.

Дополнительные browser sinks:

- `iframe.srcdoc`, `Element`/`ShadowRoot.setHTMLUnsafe`, `Document.parseHTMLUnsafe`;
- `Range.createContextualFragment` и `DOMParser.parseFromString(..., "text/html")`, если созданные nodes затем вставляются в живой документ;
- `setAttribute("on…", value)` и другие inline event-handler content attributes;
- `script.text` / `textContent` / текстовый node, если script затем подключается к документу;
- `eval` / `new Function` / `setTimeout` / `setInterval` со **строкой** из источника.

Парсинг во временный/inert document сам по себе не исполнение: подтверди перенос в активный DOM или другой executable consumer.

Trusted Types object не доказывает безопасность сам по себе. Проверь реализацию `createHTML` / `createScript` / `createScriptURL`, допустимые policy names, отсутствие passthrough/default policy и, если заявлена защита всех sinks, фактическое enforcement. Описывай динамическое выполнение без payload.

## URL и навигация

Сначала определи **контекст URL**, затем эффект:

| Контекст | Что подтверждать |
| --- | --- |
| Навигация: `location`, `assign` / `replace`, `window.open`, `a href`, `form action`, navigation router | Исполняемая `javascript:`/опасная `data:`-схема либо внешний/protocol-relative origin, который реально принимает API |
| Вложенный документ: `iframe src` / `srcdoc`, `object data`, `embed src` | Active document с недоверенным HTML/script, опасная схема или чужой origin с доказанным impact |
| Script/worker resource: `script.src`, dynamic `import()`, Worker/SharedWorker/importScripts | Неконстантный URL без protocol+origin allowlist; это executable resource |
| Изображение/media: `img src` / `srcset`, video/audio/source | Не считай `javascript:` XSS: resource context его не исполняет. Флагь только доказанный privacy/network leak или подмену доверенного ресурса |
| Framework binding: React `href={...}`, Vue `:href`, Svelte `href={}`, Angular `[href]` / `[src]` | Учитывай sanitizer/normalization конкретного стека и фактический DOM context |

Роутер (`navigate`, `router.push`, `<Navigate to>`, `goto`) — open redirect только если attacker-controlled absolute/protocol-relative URL действительно уходит на внешний origin. Пользовательский внутренний path без allowlist сам по себе не находка. Серверный 302 — чеклист. Обычная декларация маршрута и same-origin navigation без недоверенного URL — [`frontend-routing`](../../frontend-routing/SKILL.md), не этот файл.

Allowlist проверяй через `URL`: exact `protocol` и `origin` (scheme + hostname + port), без `includes`/наивного suffix/prefix; отдельно проверяй credentials, protocol-relative URL и path после нормализации.

## postMessage

Подтверждённая находка, если в scope есть security impact:

- обработчик `message` без exact allowlist `event.origin` **и** использование `event.data` в HTML/script/URL/navigation/auth или другом security-sensitive consumer;
- проверка origin через `includes`, `startsWith`, неправильный `endsWith`, regex без границ или принятие `"null"` **вместе** с security-sensitive consumer;
- `postMessage(..., "*")` с секретом, токеном или PII в payload.

После origin проверь ожидаемый `event.source`/channel и schema/type payload. `event.source` без origin недостаточен. Запись в storage без доказанного downstream consumer — неподтверждённый сигнал, не конечный sink. Сообщение между окнами этого origin без security-sensitive consumer — не находка.

## iframe, worker

| Паттерн | Когда флагь |
| --- | --- |
| `iframe` / `embed` `src` из источника | Пользовательский origin/схема приводит к active content или другому доказанному security impact |
| `sandbox` с `allow-scripts` + `allow-same-origin` | Недоверенный **same-origin** документ может обратиться к parent/снять sandbox; для обычного cross-origin документа комбинации недостаточно |
| `iframe.srcdoc` из источника | Недоверенный active HTML без проверенного sanitizer/TrustedHTML и подходящего sandbox |
| `new Worker` / `SharedWorker` / `importScripts` с URL из источника | URL не константа приложения |
| Blob/object URL из недоверенного текста как worker script или iframe document | Есть поток из источника |

Пустой iframe под своё UI без пользовательского `src` — не находка.

## Upload и preview

- превью файла через HTML-sink (`innerHTML` / `{@html}` / `v-html` / `dangerouslySetInnerHTML`) из `FileReader` или текста файла;
- SVG/HTML-файл в iframe/`srcdoc` без ограничения типа;
- `object URL` active content как `iframe src` без sandbox/ограничений.

`File.type`/extension контролируются клиентом и не являются самостоятельной проверкой содержимого. Превью raster через `img` + `blob:` обычно не XSS благодаря image context; подтверждай active content и его browsing context. Не считай уязвимостью сам `<input type="file">`.

## Service worker и cache

В scope: `service-worker*.js`, `sw.js`, `navigator.serviceWorker.register`, сообщения worker и Cache API.

Флагь при доказанном потоке:

- registration/import URL из источника или scope шире ожидаемого origin-path, если это даёт контроль над страницами;
- `importScripts` с неконстантным URL;
- подмена HTML-документа из Cache API данными пользователя;
- кэширование authenticated/private response под общим cache key с доказанным межпользовательским раскрытием;
- обработка worker message из недоверенного клиента, ведущая к cache/navigation/script sink.

Query/header в cache key, `skipWaiting` / `clients.claim` и cache-first сами по себе — не находка. Нужен persistent execution, подмена или раскрытие данных.

## Third-party scripts

- `script src` из переменной, query или public env, которую задаёт не деплой-конфиг;
- загрузка скрипта в runtime (`createElement('script')`) с URL из источника;
- JSONP-коллбек из query.

Константный remote script — trust boundary с полномочиями origin. Не объявляй CVE: проверь, получает ли он токены/PII/DOM, pinned ли URL, применимы ли SRI+`crossorigin` или self-hosting. Отсутствие SRI само по себе — неподтверждённый supply-chain сигнал, не FAIL.

## Косвенные browser gadgets

Если встречаются untrusted object merge/deep assignment или attacker-controlled HTML, проверь **доказанный** downstream gadget:

- prototype pollution меняет security-sensitive config/URL/sink;
- DOM clobbering подменяет ожидаемое свойство/form/global и ведёт в HTML/script/navigation sink.

Сам generic merge или element с `id`/`name` без downstream impact не флагь.

## Не флагь без доп. доказательств

- экранированный текст фреймворка;
- markdown, stories, fixtures, тесты (исключения discovery);
- HTML после конкретного вызова sanitizer, только если проверены config/hooks, версия/режим, соответствие HTML-контексту и отсутствие опасной мутации после sanitization;
- Trusted Type только при проверенной policy и фактическом enforcement для этого sink;
- sink только со статическим литералом разработчика.
