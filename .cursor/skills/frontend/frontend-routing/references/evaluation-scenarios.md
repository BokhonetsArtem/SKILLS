# Eval-сценарии скила

Используй после изменения `SKILL.md`, references или examples и на финальной проверке пакета. Для каждого сценария зафиксируй: выбранный трек (и mode, если React Router), прочитанные references, запрещённые действия и краткий handoff. Реализовывать экран целиком не нужно — достаточно dry-run решения агента.

Детерминированные проверки ниже запускай **существующими** командами (`python3`, `wc`, чтение файлов). Отдельный исполняемый валидатор, test runner и runtime-логи в пакет не добавляй.

## Матрица discovery и handoff

| Сценарий | Ожидаемый маршрут | Нельзя |
| --- | --- | --- |
| Vite plugin / route modules / `Route` typegen в зоне, задача про nested layout | React Router **framework**; discovery (mode) + react-router-rules + examples-react-router + navigation-state | переводить на data/declarative «заодно»; читать Next examples |
| `createBrowserRouter` + `RouterProvider` + `loader`/`action`, задача про 404 | React Router **data**; rules + examples-react-router + navigation-state | подменять на `<BrowserRouter>`+`<Routes>`; ставить второй роутер |
| `createBrowserRouter` **без** loader в дереве, задача про params | всё равно React Router **data**; тот же набор | считать declarative и добавлять `BrowserRouter` |
| `BrowserRouter` + `<Routes>` без data router, задача про search в URL | React Router **declarative**; rules + examples-react-router-declarative + navigation-state | читать `examples-react-router.md`; добавлять loaders/`useBlocker` без прецедента; подмешивать `next/navigation` |
| `react-router-dom` v5 (`Switch`, `useHistory`), задача про redirect | React Router **v5**; discovery + navigation-state + docs установленной v5 | читать `examples-react-router*.md`; подставлять `element` / `useNavigate` / `<Routes>` |
| `@tanstack/react-router` / `wouter` / Remix route modules уже в зоне | существующий трек зоны; discovery + navigation-state + локальный tree + docs **установленной версии** | читать React Router / Next examples как синтаксический шаблон; ставить `react-router` |
| `next` + `app/**/page.tsx` / `layout.tsx`, задача про redirect | Next **App Router**; next-routing-rules (App) + examples-next-app-router + navigation-state | смешивать `next/router`; класть тот же экран в `pages/`; ставить React Router |
| `pages/_app` + `getServerSideProps` / `next/router`, без App в зоне | Next **Pages Router** (legacy); next-routing-rules (Pages) + examples-next-pages-router | переносить в `app/` «заодно»; копировать `loading.tsx` / `notFound()` / `generateMetadata` |
| Монорепо: App Router в одной оболочке, React Router в другой; правка только SPA | трек **зоны изменения**; examples только SPA | тащить `next/navigation` в SPA; мигрировать соседнюю оболочку |
| В зоне равноправны `app/` и `pages/`, или два SPA-router | **спросить**; discovery | выбирать React Router / App Router по популярности; мигрировать стек |
| Сигналов мало: React без router-пакета и без файлового Next | **спросить**; discovery не разворачивать в examples | ставить `react-router` «по умолчанию» |
| HTTP 3xx / `Location` / params **endpoint** без client route | [`frontend-api`](../../frontend-api/SKILL.md), остановиться | грузить routing references |
| Поиск / фильтр / пагинация UI без записи в URL маршрута | [`frontend-layout`](../../frontend-layout/SKILL.md), остановиться | начинать этот скил из «обычного поиска» |
| Простой `<a href>` на статику, якорь, download, внешний origin без смены route behavior | не этот скил | подменять router `Link` или наоборот без evidence |
| «Куда положить page / public API» без декларации маршрута | [`frontend-architecture`](../../frontend-architecture/SKILL.md), затем routing только для tree | решать placement этим скилом |
| Dirty form / confirm leave / `useBlocker` как form lifecycle | [`frontend-forms`](../../frontend-forms/SKILL.md) | перехватывать leave-guard здесь |
| «Проверь XSS / open redirect / client-only auth» | [`frontend-security`](../../frontend-security/SKILL.md), остановиться | подменять аудит обычной навигацией |
| В репо несколько роутеров, в **этой** зоне один устойчивый прецедент | трек зоны; examples только его | подмешивать чужой examples-файл |

## Поведенческие регрессии

Проверь, что рецепт выбранного трека гарантирует:

- deep link на вложенный path открывает тот же экран, что и client navigation; второй роутер / вложенный `BrowserRouter` это ломает;
- login success, канонизация URL и «уже здесь» идут через **replace**, а не копируют history entry;
- search и hash сохраняются при соседней навигации, если так делают ближайшие маршруты зоны; фильтр в одном `useState` не выдаётся за URL-state;
- route params читаются API трека (`useParams` / `params` / `router.query`), а не парсингом `pathname` в обход tree;
- 404 / error / loading — route-level convention трека, не `return null` и не вечный blank gate, который снимает chrome и фокус;
- client guard — UX-ограничение навигации, **не** server authorization и не замена session/endpoint checks;
- после client navigation: scroll по контракту зоны, focus не на `document.body`, document title совпадает с сегментом;
- return URL / redirect target — same-origin relative path после decode (`/` да, `//` / `\` / `http:` / `javascript:` нет); хелпер в examples — минимум, не allowlist; произвольный абсолютный URL из query — open redirect, не «гибкий next»;
- navigation, params, return URL и PII не логируются;
- новый test runner и второй router-пакет не появляются.

## Проверка пакета

Корень пакета: `.cursor/skills/frontend/frontend-routing/`. Команды ниже — копируемые рецепты, не новый файл в репозитории.

Нейтральные учебные пути **разрешены**: `src/routes.tsx`, `src/app/...`, `src/pages/...`, `app/...`, `pages/...`, `src/catalog/...`. Проверка **не** имеет права завалить пакет только потому, что в тексте есть `src/` или `src/...`.

### 1. Frontmatter основного `SKILL.md`

- Файл начинается с YAML `---`.
- `name` равен `frontend-routing` и совпадает с именем директории пакета.
- Есть `description` и `disable-model-invocation`.
- `argument-hint` заключён в кавычки (`"..."` или `'...'`): голые `[...]` ломают YAML.

```bash
python3 - <<'PY'
from pathlib import Path
p = Path(".cursor/skills/frontend/frontend-routing/SKILL.md")
text = p.read_text(encoding="utf-8")
assert text.startswith("---\n"), "missing frontmatter"
end = text.find("\n---", 4)
assert end != -1, "unclosed frontmatter"
fm = text[4:end]
assert "name: frontend-routing" in fm
assert p.parent.name == "frontend-routing"
assert "description:" in fm
assert "disable-model-invocation:" in fm
hint = next(line for line in fm.splitlines() if line.startswith("argument-hint:"))
assert hint.split(":", 1)[1].strip()[:1] in {"'", '"'}, "argument-hint must be quoted"
print("frontmatter ok")
PY
```

### 2. Лимит строк `SKILL.md`

Основной файл короче 500 строк; подробности живут в `references/`.

```bash
python3 - <<'PY'
from pathlib import Path
n = len(Path(".cursor/skills/frontend/frontend-routing/SKILL.md").read_text(encoding="utf-8").splitlines())
assert n < 500, n
print(f"SKILL.md lines: {n}")
PY
```

### 3. Относительные ссылки существуют

Все markdown-ссылки внутри пакета относительные (не обязательный абсолютный URL наружу) и указывают на существующий файл. Hash-якорь отбрасывается. Ссылки только на соседние frontend-скилы (`../frontend-*/SKILL.md` и их references) тоже должны существовать.

```bash
python3 - <<'PY'
import re
from pathlib import Path
root = Path(".cursor/skills/frontend/frontend-routing")
link_re = re.compile(r"\[[^\]]*\]\(([^)]+)\)")
missing = []
for md in root.rglob("*.md"):
    for raw in link_re.findall(md.read_text(encoding="utf-8")):
        target = raw.strip()
        if target.startswith(("http://", "https://", "mailto:")):
            continue
        if target.startswith("#"):
            continue
        path = target.split("#", 1)[0]
        resolved = (md.parent / path).resolve()
        if not resolved.is_file():
            missing.append(f"{md}: {target}")
assert not missing, "\n".join(missing)
print("links ok")
PY
```

### 4. GOOD/BAD в каждом examples-файле

Файлы: `examples-react-router.md`, `examples-react-router-declarative.md`, `examples-next-app-router.md`, `examples-next-pages-router.md`.

В каждом:

- во вступлении есть метки `(GOOD)` и `(BAD)` рядом с «надо» / «не надо» (жирный markdown вокруг слов допустим: `**надо** (GOOD)`);
- есть девять разделов `## 1.` … `## 9.` с темами: декларация маршрута, nested layouts, history push/replace, params/search, redirect/return URL, guards, loading/error/404, scroll/focus, тестовый harness;
- в каждом таком разделе есть и `// надо`, и `// не надо`.

```bash
python3 - <<'PY'
from pathlib import Path
import re
root = Path(".cursor/skills/frontend/frontend-routing/references")
files = [
    "examples-react-router.md",
    "examples-react-router-declarative.md",
    "examples-next-app-router.md",
    "examples-next-pages-router.md",
]
needles = [
    "Декларация маршрута",
    "Nested layouts",
    "History: push vs replace",
    "Params и search params",
    "Redirect и return URL",
    "Guards",
    "Loading, error, 404",
    "Scroll и focus после client navigation",
    "Тестовый harness",
]
for name in files:
    text = (root / name).read_text(encoding="utf-8")
    assert "(GOOD)" in text and "(BAD)" in text, name
    assert "надо" in text and "не надо" in text, name
    sections = re.split(r"\n(?=## \d+\. )", text)
    headed = [s for s in sections if re.match(r"## \d+\. ", s)]
    assert len(headed) == 9, (name, len(headed))
    for i, (section, title) in enumerate(zip(headed, needles), 1):
        first = section.splitlines()[0]
        assert title in first, (name, first, title)
        assert "// надо" in section, (name, i, "missing GOOD")
        assert "// не надо" in section, (name, i, "missing BAD")
print("GOOD/BAD ok")
PY
```

### 5. Нет product fingerprints текущего репозитория

Составляй список **во время проверки** из хост-репозитория, в котором лежит пакет. Сам список в файлы скила **не копируй** — иначе универсальный пакет станет картой чужого продукта.

Откуда брать токены:

1. `package.json` хоста: `name`; последний сегмент `repository.url` / `homepage`; org в GitHub-пути; host из `homepage` / `bugs.url` / email в `authors`.
2. Alias из `compilerOptions.paths` (`tsconfig.json` / `jsconfig.json`): префикс до `/*`. Сам префикс `@/` как предупреждение «не копируй alias» **разрешён**. Запрещён только alias **плюс** first-level каталог хоста (`src/` children), например импорт вида alias+`entities` / alias+`shared`, если такие каталоги есть у хоста.
3. Реальные module/path identifiers: относительные пути хостовых файлов, в имени которых есть `route`/`router` (любой регистр), длиной ≥ 3 сегментов после корня приложения (`src/`, `app/`, `pages/`). Плюс basename этих файлов, кроме нейтральных `index`, `routes`, `router`, `route`, `layout`, `page`, `loading`, `error`.

Что **не** является fingerprint:

- учебные пути `src/routes.tsx`, `src/app/catalog/...`, `src/pages/catalog/...`, `app/catalog/...`, `pages/catalog/...`;
- любое вхождение `src/` без совпадения с составленным host-specific путём;
- имена библиотек (`react-router`, `next`, `@tanstack/react-router`) и нейтральные сущности примеров (`catalog`, `sign-in`, `itemId`).

Поиск — по всем `*.md` пакета, без учёта регистра для identity-токенов из п.1.

```bash
python3 - <<'PY'
import json, re
from pathlib import Path
skill = Path(".cursor/skills/frontend/frontend-routing")
host = Path(".")
pkg = json.loads((host / "package.json").read_text(encoding="utf-8"))
identity = set()
def add_ident(value):
    if not value or not isinstance(value, str):
        return
    identity.add(value)
    skip = {
        "http", "https", "github.com", "www", "com", "org", "html",
        "issues", "blob", "tree", "pull", "pulls", "commit", "wiki",
    }
    for part in re.split(r"[/:@\s<>]+", value):
        part = part.strip(" .")
        if len(part) >= 4 and part.lower() not in skip:
            identity.add(part)
add_ident(pkg.get("name"))
for key in ("homepage", "repository"):
    raw = pkg.get(key)
    if isinstance(raw, dict):
        add_ident(raw.get("url"))
    else:
        add_ident(raw)
bugs = pkg.get("bugs")
add_ident(bugs.get("url") if isinstance(bugs, dict) else bugs)
add_ident(pkg.get("authors") if isinstance(pkg.get("authors"), str) else None)
GENERIC_BASE = {"index", "routes", "router", "route", "layout", "page", "loading", "error"}
host_paths = []
host_bases = []
for folder in (host / "src", host / "app", host / "pages"):
    if not folder.is_dir():
        continue
    for file in folder.rglob("*"):
        if not file.is_file() or not re.search(r"route", file.name, re.I):
            continue
        rel = file.relative_to(host).as_posix()
        if rel.count("/") >= 3:
            host_paths.append(rel)
        stem = file.stem
        if stem.lower() not in GENERIC_BASE and len(stem) >= 8:
            host_bases.append(stem)
alias_hits = []
for cfg_name in ("tsconfig.json", "jsconfig.json"):
    cfg_path = host / cfg_name
    if not cfg_path.is_file():
        continue
    cfg = json.loads(cfg_path.read_text(encoding="utf-8"))
    paths = (cfg.get("compilerOptions") or {}).get("paths") or {}
    src_children = [p.name for p in (host / "src").iterdir() if p.is_dir()] if (host / "src").is_dir() else []
    for alias in paths:
        prefix = alias.split("/", 1)[0]
        if not prefix:
            continue
        for child in src_children:
            alias_hits.append(f"{prefix}/{child}")
blob = "\n".join(p.read_text(encoding="utf-8") for p in skill.rglob("*.md"))
blob_l = blob.lower()
hits = []
for token in sorted(identity, key=len, reverse=True):
    if token.lower() in blob_l:
        hits.append(f"identity:{token}")
for rel in host_paths:
    if rel in blob:
        hits.append(f"path:{rel}")
for stem in host_bases:
    if re.search(r"\b" + re.escape(stem) + r"\b", blob):
        hits.append(f"module:{stem}")
for item in alias_hits:
    if item in blob:
        hits.append(f"alias:{item}")
assert not hits, "product fingerprints:\n" + "\n".join(hits)
print("fingerprints ok; educational src/ paths are allowed")
PY
```

### 6. После регистрации скила

Эти пункты проверяй, когда появятся rule и каталог frontend-скилов (не раньше):

- `.cursor/rules/frontend-routing.mdc` существует и ведёт на полный `SKILL.md`, а не копирует контракт;
- trigger правила требует routing evidence (route tree / navigation / redirect / params / search / framework middleware или proxy) и **не** срабатывает только из-за открытого `ts`/`tsx`, HTTP redirect/endpoint params или обычного поиска без URL-state;
- `.cursor/skills/frontend/README.md` перечисляет скил и отдельное правило вызова.

## Как читать результат dry-run

Коротко, не эссе:

- **Трек:** React Router (framework / data / declarative / v5) / другой SPA-router / Next App / Next Pages / спросили пользователя / остановились (handoff).
- **References:** какие файлы прочитаны; examples — только выбранного трека.
- **Запрещено:** второй роутер, смена mode/стека, runtime-логи, новый runner, выдача client guard за authorization.
- **Handoff:** architecture / api / forms / layout / security / не этот скил.
