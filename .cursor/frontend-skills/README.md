# Frontend skills

Сюда кладём **фронтовые** скилы: вёрстка, UI, стили, макеты, архитектура frontend-кода.

Процессные скилы AI Factory (`aif-*`) остаются в `.cursor/skills/` и сюда **не** переносятся.

Cursor автоподхватывает скилы только из `.cursor/skills/` (и `.agents/skills/`). Эта папка сама по себе не сканируется. Поэтому у каждого скила отсюда нужен тонкий указатель `.cursor/skills/<name>/SKILL.md`: `name` и `description` совпадают с полным скилом; указатель проверяет область задачи и читает полный `SKILL.md` из этой директории.

## Конвенции

- Один скил = одна папка `kebab-case/` с `SKILL.md` и при необходимости `references/`
- Язык скилов пока **русский**; YAML `name` — латиница и совпадает с именем папки
- Новые фронтовые скилы добавляем сюда, не в `.cursor/skills/` (кроме указателя)
- Правила вызова раздельные, не одно общее на любой TypeScript:
  - layout (`tsx`/`css`): `.cursor/rules/frontend-skills.mdc`
  - architecture (`ts`/`tsx`): `.cursor/rules/frontend-architecture.mdc`
- Стек стилей скил берёт из репозитория; CSS Modules и Tailwind не дефолт

## Скилы

| Папка | Назначение |
| --- | --- |
| `frontend-layout/` | Вёрстка по стеку репозитория (Modules, Tailwind, CSS-in-JS, SCSS, UI-kit) |
| `frontend-architecture/` | Размещение кода по архитектуре репозитория (границы, public API, зависимости, state/API) |
