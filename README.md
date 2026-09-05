# Multiagents Orchestration

Плагин для Cursor: оркестрация фичи через команду **Analyst → Lead → Dev/QA → ревью → тесты → приёмка**.

После установки в чате доступна команда **`/multiagents-orchestration`**.

Репозиторий: [github.com/stswoon/cursor-plugin](https://github.com/stswoon/cursor-plugin)

### Из Git-репозитория

1. Открой **Customize** (или **Cursor Settings → Plugins**).
2. Добавь плагин из репозитория: `https://github.com/stswoon/cursor-plugin`
3. Выбери scope: **user** (все проекты) или **project** (только текущий).
4. Перезагрузи окно: Command Palette (`Ctrl+Shift+A`) → **Developer: Reload Window**.
5. В чате набери `/` и проверь, что есть `/multiagents-orchestration`.

### Локально (разработка или приватная копия)

1. Включи загрузку локальных плагинов, если это запрещено политикой организации (**Allow Local Plugin Imports**).
2. Сделай junction или symlink репозитория в `~/.cursor/plugins/local/`:

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.cursor\plugins\local"
New-Item -ItemType Junction -Path "$env:USERPROFILE\.cursor\plugins\local\multiagents-orchestration" -Target "D:\mycode\cursor-plugin"
```

На macOS / Linux:

```bash
mkdir -p ~/.cursor/plugins/local
ln -s /path/to/cursor-plugin ~/.cursor/plugins/local/multiagents-orchestration
```

3. **Developer: Reload Window**.
4. В **Customize** плагин должен появиться в установленных.

Если команда не видна: проверь, что манифест лежит именно в корне плагина (`.cursor-plugin/plugin.json`, не на уровень
глубже) и что включены third-party plugins.

## Что входит в плагин

| Компонент | Имя                          | Зачем                                 |
|-----------|------------------------------|---------------------------------------|
| Command   | `/multiagents-orchestration` | Точка входа: полный цикл фичи         |
| Skill     | `multiagents-orchestration`  | Правила оркестрации для агента        |
| Agent     | `/analyst`                   | Требования, дизайн, финальная приёмка |
| Agent     | `/lead`                      | Декомпозиция и code review            |
| Agent     | `/dev-fe`                    | Имплементация по задачам Lead         |
| Agent     | `/qa`                        | Тест-кейсы и прогон                   |

Подробная схема
процесса: [skills/multiagents-orchestration/multiagents.md](skills/multiagents-orchestration/multiagents.md).

## Как работать с `/multiagents-orchestration`

### Когда вызывать

Вызывай команду в Agent-чате **целевого проекта** (не в этом репозитории плагина), когда нужна новая фича с
требованиями, дизайном и проверкой.

```
/multiagents-orchestration Добавь на главную страницу фильтр заказов по статусу
```

Можно сначала команду, потом задачу отдельным сообщением. Агент спросит формулировку, если её нет.

Не используй полный цикл для мелкой правки в одном файле — достаточно обычного агента. Отдельные роли:

- только анализ и дизайн → `/analyst`
- только ревью уже написанного кода → `/lead`

### Что происходит

Агент ведёт команду по шагам и **не перескакивает гейты**. Production-код пишет Dev, не оркестратор.

```
Ты → Analyst (1) → Lead (2) → параллельно QA (4а) + Dev (4б) → Lead review (5) → QA test (6) → Analyst accept (7) → готово
```

| Шаг     | Кто                               | Что получишь                                           | Гейт дальше                     |
|---------|-----------------------------------|--------------------------------------------------------|---------------------------------|
| 1       | `/analyst`                        | Уточнения в чат, затем `requirements.md` и `design.md` | Нет открытых критичных вопросов |
| 2       | `/lead`                           | `dev-tasks.md`, `qa-task.md`                           | Задачи нарезаны                 |
| 4а + 4б | `/qa` и `/dev-fe` **параллельно** | `test-cases.md` и код в проекте                        | Dev сдал код, QA сдал TC        |
| 5       | `/lead`                           | Ревью + e2e из дизайна; при fail — `review-notes.md`   | Ревью OK                        |
| 6       | `/qa`                             | Прогон TC; при багах — `bug-report.md`                 | Все TC зелёные                  |
| 7       | `/analyst`                        | Sunny-day сценарии заказчика                           | Приёмка OK                      |

Петли: с шагов 5, 6 и 7 работа возвращается Dev (4б), затем снова 5 → 6 → 7.

На шаге 1 отвечай на вопросы аналитика — без этого дизайн не начнётся. На шаге 4 Dev и QA стартуют вместе: тест-кейсы не
блокируют код.

### Как помогать агенту

- Опиши цель, границы и что точно **не** входит в задачу.
- Если аналитик задал вопросы — ответь по пунктам, не переформулируй всё с нуля без нужды.
- Не проси «сразу напиши код»: оркестратор сначала закроет анализ и нарезку.

### Примеры

Полный цикл:

```
/multiagents-orchestration
Сделай форму обратной связи на /contacts: имя, email, сообщение.
Валидация на клиенте, без бэкенда — покажи toast об успехе.
Не добавляй новые библиотеки.
```

Только анализ:

```
/analyst
Нужен экспорт таблицы заказов в CSV. Уточни требования и напиши design.md.
```

Только ревью:

```
/lead
Проведи шаг 5: ревью текущего diff против .cursor/artifacts/design.md
```

### Ограничения, которые соблюдает команда

- Минимальный scope: не расширять задачу и не тащить новые зависимости без запроса.
- Dev перед сдачей гоняет сборку проекта (например `npm run build`).
- Автотесты после QA — только если ты разрешил фреймворк.
- Коммиты — только по твоей просьбе.
