# Multiagents — команда

Схема процесса: [multiagents.md](../multiagents.md)  
Оркестрация: skill `multiagents-orchestration` (`/multiagents-orchestration`)

## Команды

| Команда    | Агент              | Шаги          |
|------------|--------------------|---------------|
| `/analyst` | Системный аналитик | 1, 7          |
| `/lead`    | Lead               | 2, 5          |
| `/dev-fe`  | Dev FE             | 4б, доработки |
| `/qa`      | QA                 | 4а, 6         |

## Поток

```
Пользователь
    │
    ▼
 /analyst  ── requirements.md, design.md
    │
    ▼
  /lead    ── dev-tasks.md, qa-task.md
    │
    ├── /qa     ── test-cases.md      ┐ параллельно
    └── /dev-fe ── src/               ┘
    │
    ▼
  /lead    ── review (review-notes.md при fail)
    │
    ▼
   /qa     ── тестирование → автотесты
    │
    ▼
 /analyst  ── sunny-day приёмка
    │
    ▼
   ✅
```

Артефакты: `.cursor/artifacts/`
