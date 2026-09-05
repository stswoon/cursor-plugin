# Multiagents Orchestration

Плагин для Cursor: оркестрация фичи через команду **Analyst → Lead → Dev/QA → ревью → тесты → приёмка**.

После установки в чате доступна команда **`/multiagents-orchestration`**.

Репозиторий: [github.com/stswoon/cursor-plugin](https://github.com/stswoon/cursor-plugin)

## Установка

Репозиторий **сам является плагином**: `.cursor-plugin/plugin.json` лежит в корне.

### Локально

Единственный способ поставить плагин себе, пока он не опубликован в Marketplace.

1. Если действует политика организации — включи **Allow Local Plugin Imports**
   (Dashboard → Settings → Security & Identity → Marketplace and Plugins).
2. Скопируй репозиторий в `~/.cursor/plugins/local/multiagents-orchestration`.

```powershell
$src = "D:\mycode\cursor-plugin"
$dest = "$env:USERPROFILE\.cursor\plugins\local\multiagents-orchestration"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
robocopy $src $dest /MIR /XD .git .idea /NFL /NDL /NJH /NJS
```

На macOS / Linux можно скопировать или, как советует документация, сделать симлинк:

```bash
rsync -a --delete --exclude .git --exclude .idea ./ ~/.cursor/plugins/local/multiagents-orchestration/
# либо
ln -s "$PWD" ~/.cursor/plugins/local/multiagents-orchestration
```

На Windows симлинк/junction не подходит — копируй. После правок в репозитории копируй снова:
локальная папка сама не обновляется.

3. **Developer: Reload Window**.
4. В **Customize** плагин должен появиться в установленных.

Проверить, что Cursor его подхватил, можно в логе: **Output → Cursor Plugins**, ищи строку
`loadUserLocalPlugin multiagents-orchestration loaded`.

### Почему ссылка на GitHub не работает

Вставить `https://github.com/stswoon/cursor-plugin` и получить плагин нельзя — такого флоу у Cursor нет:

- Импорт GitHub-репозитория как маркетплейса — это **Dashboard → Plugins → Team Marketplaces →
  Add Marketplace → «Import from Repo»**, и team marketplaces доступны только на планах
  **Teams и Enterprise** ([документация](https://cursor.com/docs/plugins)).
- Публичный Marketplace принимает плагины только через ручное ревью на `cursor.com/marketplace/publish`.

Если добавить URL всё-таки не получилось, это видно в логе `Cursor Plugins`: маркетплейс не появляется
среди источников, а в `~/.cursor/plugins/cache/` не создаётся папка под него.

## Что входит в плагин

| Компонент | Имя                          | Зачем                                 |
|-----------|------------------------------|---------------------------------------|
| Command   | `/multiagents-orchestration` | Точка входа: полный цикл фичи         |
| Skill     | `multiagents-orchestration`  | Правила оркестрации для агента        |
| Agent     | `/analyst`                   | Требования, дизайн, финальная приёмка |
| Agent     | `/lead`                      | Декомпозиция и code review            |
| Agent     | `/dev-fe`                    | Имплементация по задачам Lead         |
| Agent     | `/qa`                        | Тест-кейсы и прогон                   |

```
.cursor-plugin/plugin.json          манифест
agents/                             analyst, lead, dev-fe, qa
commands/                           /multiagents-orchestration
skills/multiagents-orchestration/   SKILL.md, multiagents.md, team.md
```

Подробная схема процесса:
[skills/multiagents-orchestration/multiagents.md](skills/multiagents-orchestration/multiagents.md).

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
