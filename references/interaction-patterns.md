# Interaction Patterns

Common interaction patterns used by all skills. Read this reference at the start of any skill that interacts with the user.

## Language

All messages to the user MUST be in Russian.

## Confirmation Flow

Before making any changes:

1. Present a plan of actions (what will be done)
2. Wait for explicit user confirmation — do NOT proceed without it
3. User can: approve, partially edit, or cancel

## Interactive Mode

Used by skills that process items one by one (audit, dedup):

1. Show the problem/finding
2. Propose a specific action (fix / merge / skip)
3. Wait for user response: yes / no / custom alternative
4. Apply the action, move to the next item

## Trust Level Selection

For skills that support trust levels (all skills in v2), ask the user at the start:

```

Выберите уровень подтверждения:

- **cautious** — подтверждение каждого действия отдельно
- **balanced** — подтверждение группами по категориям (рекомендуется)
- **auto** — применить всё автоматически, показать отчёт

[default_level]

```

See `references/trust-levels.md` for full details on behavior, constraints, and defaults per skill.

## Subagent Error Handling

For skills that dispatch subagents (audit, dedup, cluster, process-inbox):

**Subagent failed:**
- Do NOT block remaining subagents — use partial results
- Inform user: "Задача [task] не завершилась ([error]). Продолжить без этих данных или повторить?"
- On retry: re-dispatch only the failed subagent

**Temp file missing after subagent completes:**
- Treat as subagent failure
- Same user prompt as above

**Subagent returned empty result:**
- Valid result — category may have zero issues
- Report as: "Категория [X]: проблем не найдено ✓"

## Report Format

After completing all operations, output a summary:

```
## Результат: [skill name]

Обработано: N заметок
Изменено: M заметок
- [Файл] — [что сделано]
- ...
Пропущено: K заметок
```

## Daily Log

After outputting the report, offer to log results to the Daily Note:

```
Записать результат в Daily Note? (y/n)
```

If yes:

### Timezone check (first time per conversation)

Before writing the timestamp, detect the system timezone and confirm it with the user:

```bash
date +"%Z (UTC%:z)"
```

Show the user:

```
Системное время: HH:MM (TIMEZONE). Это ваш часовой пояс? (y/n)
```

- If confirmed — use this timezone for all subsequent daily log entries in this conversation without asking again.
- If not — ask the user for their timezone offset (e.g. `+10`, `-5`), apply it to all timestamps, and remember for the rest of the conversation.

### Writing the log entry

1. Ensure daily note exists: `obsidian daily`
2. Get path: `obsidian daily:path`
3. Try append: `obsidian daily:append content="\n### [Skill name] — HH:MM\n- Обработано: N заметок\n- Изменено: M\n- [краткий список действий]"`
4. **Fallback:** if append fails silently (file stays empty), write directly via `Read` + `Edit` or `Write` tool using the path from step 2.

Use the **user-confirmed local time** (HH:MM) and actual counts from the report.

## Commit Convention

### С obsidian-git (по умолчанию)

Если в vault установлен плагин obsidian-git — **НЕ делать ручных коммитов**. Плагин автоматически коммитит и пушит изменения по интервалу. Скилл просто заканчивается отчётом + Daily Log.

Чтобы проверить, активен ли obsidian-git:
```bash
obsidian plugins
```
Если в списке есть `obsidian-git` — ручной коммит не нужен.

### Без obsidian-git (fallback)

Если obsidian-git не установлен, каждый скилл заканчивается ручным коммитом. Use `git add <specific files>`, NOT `git add -A`. Only add files changed during the current operation.

Commit messages by skill:
- dedup: `refactor: merge duplicate notes`
- cluster: `feat: create [MOC/summary] for [topic]`
- audit: `fix: audit and fix notes quality`
- decompose: `refactor: decompose [original filename] into atomic notes`
- process-inbox: `feat: process inbox notes`
- rename-tags: `refactor: rename tags`
- improve: `refactor: improve content of [note name]`

## Safety Rules

- NEVER delete files — always archive or convert
- NEVER rename files without user confirmation (wikilinks may break)
- NEVER proceed without explicit user approval
- Preserve all original content — nothing gets lost
