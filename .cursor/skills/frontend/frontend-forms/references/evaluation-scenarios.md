# Eval-сценарии скила

Используй после изменения `SKILL.md`, routing или examples. Для каждого сценария проверь: выбранный трек, прочитанные references, запрещённые действия и краткий handoff. Не нужно реализовывать форму целиком — достаточно dry-run решения агента.

## Матрица discovery

| Сценарий | Ожидаемый маршрут | Нельзя |
| --- | --- | --- |
| RHF + yup в зоне, задача про submit | RHF; discovery + lifecycle + examples-rhf | ставить Formik/Zod, submit только через `onClick` |
| RHF + zod в зоне, задача про schema | RHF; discovery + validation + examples-rhf | заменять zod на yup |
| Formik + yup, server field error | Formik; validation + examples-formik | использовать `setError` RHF или toast-only |
| Без form library, локальный `useReducer` | native; examples-native + тематический reference | ставить RHF/Formik |
| `@tanstack/react-form` уже используется | существующий трек зоны; discovery + локальные формы/API версии | читать RHF/Formik snippets как синтаксический шаблон |
| Server/route action управляет submit | framework-managed; discovery + lifecycle + локальный action API | дублировать action client mutation или `onClick` |
| В зоне два равноправных form-подхода | спросить пользователя | выбирать по популярности |
| «Сверстай форму по Figma» без state/submit | `frontend-layout`, остановиться | грузить полный forms skill |
| «Куда положить schema формы» | `frontend-architecture`, затем forms только для поведения | решать placement этим скилом |
| «Проверь XSS в preview» | `frontend-security`, остановиться | подменять аудит обычной validation |

## Поведенческие регрессии

Проверь, что рецепт выбранного трека гарантирует:

- уникальные `id` при двух экземплярах формы на странице;
- `<form onSubmit>` / `<Form>` / framework action и один канонический submit;
- неизвестный `field` из API становится общей ошибкой, а известный — ошибкой поля;
- nested/array error получает focus при рабочем `ref`;
- повторный submit не создаёт две опасные мутации;
- async validation не применяет устаревший response к новому значению;
- скрытое условное поле не отправляет stale value без явного контракта;
- переходы multi-step формы не теряют допустимые значения;
- RHF `field.id` не используется как доменный id DTO;
- значения полей, пароли и PII не логируются.

## Проверка пакета

- `name` совпадает с директорией, `argument-hint` заключён в кавычки.
- Основной `SKILL.md` короче 500 строк; подробности находятся в `references/`.
- Все ссылки относительные и ведут на существующие файлы.
- Нет названий конкретных продуктов, репозиториев, слайсов и путей `src/`.
- Скил лежит в `.cursor/skills/frontend/frontend-forms/`, rule ведёт на полный `SKILL.md`.
- README перечисляет скил и отдельное правило вызова.
