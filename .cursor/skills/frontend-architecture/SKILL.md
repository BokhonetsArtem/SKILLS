---
name: frontend-architecture
description: Определяет архитектуру текущего frontend-проекта и размещает код — границы модулей, public API, зависимости, state/API, рефакторинг структуры. Use when новый слайс/модуль, куда класть код, границы feature/entity, импорты, циклы, ownership state/API или реорганизация frontend-структуры. Не для вёрстки, CSS, Figma и не для генерации .ai-factory/ARCHITECTURE.md. Не навязывает FSD и не добавляет второй архитектурный стиль или state manager, если их нет в репозитории.
disable-model-invocation: false
argument-hint: "[слайс | модуль | рефакторинг]"
---

# Frontend architecture

Если задача не про архитектуру frontend-кода — размещение слайса/модуля, границы, public API, зависимости, state/API или реорганизацию структуры — остановись здесь, полный скил не читай. Вёрстка, CSS, Figma и `.ai-factory/ARCHITECTURE.md` — не эта область.

Если задача в области скила, сразу прочитай полный скил целиком доступным инструментом чтения файлов (не фрагмент):

`.cursor/frontend-skills/frontend-architecture/SKILL.md`

Дальше выполняй только его инструкции. Не смешивай с процессными `aif-*` в `.cursor/skills/`: содержимое фронтовых скилов живёт в `.cursor/frontend-skills/`.
