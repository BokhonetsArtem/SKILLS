# Frontend skills

Сюда кладём **фронтовые** скилы: вёрстка, UI, стили, макеты, архитектура frontend-кода, клиентский HTTP/REST/JSON, клиентская маршрутизация, поведение форм, аудит клиентской безопасности.

Процессные скилы AI Factory (`aif-*`) остаются в `.cursor/skills/` на верхнем уровне и сюда **не** переносятся.

Cursor рекурсивно обходит `.cursor/skills/` и подхватывает любой `SKILL.md`. Папка `frontend/` — только группировка; имя скила берётся из папки, где лежит `SKILL.md` (`frontend-layout`, не `frontend`).

## Конвенции

- Один скил = одна папка `kebab-case/` с `SKILL.md` и при необходимости `references/`
- Язык скилов пока **русский**; YAML `name` — латиница и совпадает с именем папки
- Новые фронтовые скилы добавляем сюда, не в корень `.cursor/skills/`
- Правила вызова раздельные, не одно общее на любой TypeScript:
  - layout (`tsx`/`css`): `.cursor/rules/frontend-skills.mdc`
  - architecture (`ts`/`tsx`, только placement/public API/owner state): `.cursor/rules/frontend-architecture.mdc`
  - api (`ts`/`tsx`, только HTTP client/endpoint/query/mutation/cache): `.cursor/rules/frontend-api.mdc`
  - routing (`ts`/`tsx`, только route tree/navigation/redirect/params/search/framework middleware или proxy): `.cursor/rules/frontend-routing.mdc`
  - forms (`ts`/`tsx`, только form state/validation/submit): `.cursor/rules/frontend-forms.mdc`
  - security (только явный frontend/client audit, не любой `ts`/`tsx`): `.cursor/rules/frontend-security.mdc`
- Generic «is this secure?» / pre-deploy / OWASP — процессный `aif-security-checklist` в `.cursor/skills/`, не `frontend-security`
- Стек стилей скил берёт из репозитория; CSS Modules и Tailwind не дефолт

## Скилы

| Папка | Назначение |
| --- | --- |
| `frontend-layout/` | Вёрстка по стеку репозитория (Modules, Tailwind, CSS-in-JS, SCSS, UI-kit). Правило: `.cursor/rules/frontend-skills.mdc` |
| `frontend-architecture/` | Размещение кода по архитектуре репозитория (границы, public API, зависимости, owner state). Правило: `.cursor/rules/frontend-architecture.mdc` |
| `frontend-api/` | Клиентский HTTP/REST/JSON по стеку зоны: client, endpoint, query/mutation, DTO, cache, retry. Правило: `.cursor/rules/frontend-api.mdc` |
| `frontend-routing/` | Клиентская маршрутизация по стеку зоны: route tree, navigation, redirect, params/search, URL-state. Правило: `.cursor/rules/frontend-routing.mdc` |
| `frontend-forms/` | Поведение React-форм по стеку зоны: state, validation, submit, errors, lifecycle и a11y. Правило: `.cursor/rules/frontend-forms.mdc` |
| `frontend-security/` | Аудит клиентской части source-to-sink без правок кода. Правило: `.cursor/rules/frontend-security.mdc`. Generic OWASP / «надежно ли это?» — `aif-security-checklist` |
