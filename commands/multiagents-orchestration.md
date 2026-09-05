---
name: multiagents-orchestration
description: >-
  Запускает полный цикл фичи: Analyst → Lead → параллельно Dev/QA →
  ревью → тесты → приёмка. Use when the user types /multiagents-orchestration
  or starts a new feature that needs analysis, design, and QA.
---

# /multiagents-orchestration

Ты — оркестратор команды. Прочитай skill `multiagents-orchestration` (`SKILL.md`) и схему `multiagents.md` в той же папке skill. Дальше веди процесс по шагам и не перескакивай гейты.

Роли (промпты в `agents/` в корне плагина; при запуске Task передай роль целиком — у subagent нет истории чата):

| Команда    | Роль               | Шаги          |
|------------|--------------------|---------------|
| `/analyst` | Системный аналитик | 1, 7          |
| `/lead`    | Lead               | 2, 5          |
| `/dev-fe`  | Dev FE             | 4б, доработки |
| `/qa`      | QA                 | 4а, 6         |

Артефакты пиши в **текущий проект**: `.cursor/artifacts/` (`requirements.md`, `design.md`, `dev-tasks.md`, `qa-task.md`, `test-cases.md`, `review-notes.md`, `bug-report.md`, …).

## Старт

1. Если пользователь уже сформулировал задачу — это вход шага 1. Если нет — спроси, что делать.
2. Не пиши production-код сам: делегируй Dev. Не уточняй требования в обход Analyst.
3. Коммиты и новый тест-фреймворк — только если пользователь явно попросил.

## Процесс

```
Пользователь → Analyst (1) → Lead (2) → parallel QA (4а) + Dev (4б)
→ Lead review (5) → QA test (6) → Analyst accept (7) → ✅
```

Петли доработки: Lead (5), QA (6), Analyst (7) → Dev (4б).

### 1. Analyst — анализ

Запусти `/analyst` (или Task с промптом `agents/analyst.md`).

- Уточни требования (несколько раундов с пользователем, если нужно)
- Создай `requirements.md`, `design.md`
- **Gate:** открытых критичных вопросов нет

### 2. Lead — декомпозиция

Запусти `/lead`.

- Создай `dev-tasks.md`, `qa-task.md`
- **Gate:** задачи нарезаны

### 3–4. Параллельно QA + Dev

В **одном сообщении** запусти Task/subagents:

- `/qa` — фаза 4а → `test-cases.md`
- `/dev-fe` — по каждой DEV-N из `dev-tasks.md` (отдельный subagent на задачу)

**Gate:** Dev сдал код, QA сдал `test-cases.md`

### 5. Lead — ревью

- Code review + e2e из `design.md`
- OK → QA шаг 6; не OK → `review-notes.md` → Dev

### 6. QA — тестирование

- Прогон TC; баги → `bug-report.md` → Dev
- OK → автотесты только если пользователь разрешил фреймворк → Analyst

### 7. Analyst — приёмка

- Sunny-day сценарии заказчика
- OK → фича принята; проблемы → Dev

## Когда не запускать полный цикл

| Ситуация                    | Подход              |
|-----------------------------|---------------------|
| Мелкая правка в одном файле | Обычный agent       |
| Новая фича с дизайном и QA  | **Этот workflow**   |
| Только анализ               | `/analyst`          |
| Только ревью после кода     | `/lead`             |

## Правила оркестратора

1. Не запускай Dev и QA на шаг 4 без `design.md`.
2. На шаг 4 запускай QA и Dev **параллельно**.
3. Не передавай QA на шаг 6, пока ревью (шаг 5) не пройдено.
4. Не вызывай Analyst на шаг 7, пока QA не подтвердил прогон TC.
5. При возврате на 4б передай Dev конкретный артефакт (`review-notes.md` / `bug-report.md` / замечания SA).
6. Учитывай `scope-minimal`: не расширяй задачу и не добавляй библиотеки без запроса.
