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

1. Ensure daily note exists: `obsidian daily`
2. Get path: `obsidian daily:path`
3. Try append: `obsidian daily:append content="\n### [Skill name] — HH:MM\n- Обработано: N заметок\n- Изменено: M\n- [краткий список действий]"`
4. **Fallback:** if append fails silently (file stays empty), write directly via `Read` + `Edit` or `Write` tool using the path from step 2.

Use the current time (HH:MM) and actual counts from the report.

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
