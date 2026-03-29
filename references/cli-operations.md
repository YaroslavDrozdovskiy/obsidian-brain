# CLI Operations

Common CLI operations used by all skills. Read this reference at the start of any skill that interacts with the vault.

**Requirement:** Obsidian must be running for CLI commands to work.

## Pre-flight Check: Installer Version

If any CLI command output contains the warning:

```
Your Obsidian installer is out of date. Please download the latest installer which includes better CLI support
```

**STOP immediately.** Inform the user:
1. Their Obsidian installer is outdated and CLI write commands may fail.
2. They need to download the latest installer from [obsidian.md/download](https://obsidian.md/download) and reinstall (vault and settings are preserved).
3. After reinstalling, restart Obsidian and retry.

Do NOT proceed with the skill — write operations (`property:set`, `daily:append`, `append`, `prepend`, `create`, etc.) are likely to fail silently or with exit code 127.

## Windows: Obsidian.exe vs Obsidian.com

On Windows (Git Bash), `obsidian` may resolve to `Obsidian.exe` (GUI) instead of `Obsidian.com` (console). The GUI version breaks on **subcommands containing `:` when combined with parameters** (e.g. `property:set name=... value=...`), returning exit code 127.

**Symptom:** read commands work, but `property:set`, `daily:append`, and other colon-subcommands with params fail with exit 127.

**Fix:** the user must create a wrapper script at `~/bin/obsidian` (with `~/bin` in PATH):
```bash
#!/usr/bin/env bash
exec '/c/Users/<username>/AppData/Local/Programs/Obsidian/Obsidian.com' "$@"
```

A wrapper works in both interactive and non-interactive shells (unlike aliases).

If you detect exit 127 on a colon-subcommand, inform the user about this wrapper fix and stop.

## Vault Targeting

All commands use the vault from `references/vault-config.md`. If the user has multiple vaults, prepend `vault="VaultName"` to commands.

## File Operations

| Task | Command |
|------|---------|
| List files in folder | `obsidian files path="notes/"` |
| Read note content | `obsidian read file="name"` |
| Create note | `obsidian create name="folder/title" content="..." silent` |
| Append to end | `obsidian append file="name" content="..."` |
| Insert after frontmatter | `obsidian prepend file="name" content="..."` |
| Move/rename file | `obsidian move file="name" to="folder/"` |
| Delete (to trash) | `obsidian delete file="name"` |

> `move` automatically updates all wikilinks across the vault. Always prefer `move` over `bash mv`.

> `create` with `silent` flag prevents opening the file in Obsidian GUI.

> For multiline content in `create`/`append`/`prepend`, use `\n` for newlines and `\t` for tabs.

## Frontmatter / Properties

| Task | Command |
|------|---------|
| Read all properties | `obsidian properties file="name"` |
| Set/update property | `obsidian property:set file="name" name="key" value="val"` |
| Remove property | `obsidian property:remove file="name" name="key"` |

> Always prefer `property:set` over manual YAML editing — it handles formatting and edge cases.

## Search

| Task | Command |
|------|---------|
| Full-text search | `obsidian search query="text" limit=20` |
| Search with context | `obsidian search:context query="text" limit=10` |

> CLI search uses Obsidian's index — faster than `Grep` for large vaults.

## Tags

| Task | Command |
|------|---------|
| List all tags | `obsidian tags` |
| Files with specific tag | `obsidian tag tag="#tagname"` |
| Rename tag across vault | `obsidian tags:rename old="oldname" new="newname"` |

> `tags:rename` updates all files at once. No need for grep+replace loops.

## Links & Graph

| Task | Command |
|------|---------|
| Outgoing links | `obsidian links file="name"` |
| Incoming links (backlinks) | `obsidian backlinks file="name"` |
| Broken links | `obsidian unresolved` |
| Orphan notes | `obsidian orphans` |

> These commands use Obsidian's live graph index.

## Daily Notes

| Task | Command |
|------|---------|
| Open/create today's note | `obsidian daily` |
| Read today's note | `obsidian daily:read` |
| Append to today's note | `obsidian daily:append content="..."` |
| Get daily note path | `obsidian daily:path` |

> **Known bug:** `daily:append` and `daily:read` fail silently if today's daily note does not exist yet. Always ensure the note exists before appending:
> 1. Run `obsidian daily` to create today's note (if not exists)
> 2. Then use `daily:append` to add content
>
> If `daily:append` still fails, fall back to direct file write: get path via `obsidian daily:path`, then use `Write`/`Edit` tool.

## Developer / Advanced

| Task | Command |
|------|---------|
| Execute JS with app access | `obsidian eval code="..."` |
| List plugins | `obsidian plugins` |
| Reload plugin | `obsidian plugin:reload id=plugin-name` |

### eval — ограниченное использование

Использовать ТОЛЬКО для данных, недоступных через стандартные команды:

```bash
# Полный граф связей из metadataCache
obsidian eval code="JSON.stringify(Object.fromEntries(Object.entries(app.metadataCache.resolvedLinks).map(([k,v])=>[k,Object.keys(v)])))"
```

Не использовать eval для операций, покрываемых стандартными командами.

## Output Format

Для парсинга вывода используй формат json:
```bash
obsidian files path="notes/" --format json
```

Поддерживаемые форматы: `json`, `csv`, `tsv`, `md`, `paths`, `text`, `tree`, `yaml`.

## Когда использовать прямой доступ к файлам

CLI не всегда лучший выбор. Используй прямой доступ через Read/Write/Grep/Glob в этих случаях:

- **Массовое чтение** (>20 файлов) — `Read` быстрее серии `obsidian read`
- **Сложные regex** — `Grep` мощнее `obsidian search` для паттернов вроде `\[\[([^\]]+)\]\]`
- **Word count / размер файла** — `bash wc -w` / `bash stat` (CLI не предоставляет)
- **Операции с _archive, _templates** — они могут быть вне индекса CLI

### Экономия токенов: CLI vs Grep/Read

Каждый вызов `obsidian` CLI возвращает 2-3 строки служебного мусора (предупреждения, загрузка пакета) помимо полезного вывода. При массовых операциях это быстро съедает контекстное окно.

**Правило: для bulk-проверок (>5 файлов) всегда предпочитай Grep/Read.**

| Задача | CLI (дорого) | Grep/Read (дёшево) |
|--------|-------------|-------------------|
| Frontmatter всех файлов | N × `obsidian properties` | 1 × `Grep` с паттерном `^(type\|tags\|status\|created\|source):` по папке |
| Проверка наличия `## See also` | N × `obsidian read` | 1 × `Grep` с паттерном `^## See also` |
| Word count | N × `bash wc -w` + N × `obsidian read` | 1 × `bash find ... -exec wc -w` |
| Поиск битых wikilinks | `obsidian unresolved` (OK) | `Grep` для поиска источников битых ссылок |
| Type vs folder mismatch | N × `obsidian properties` | 1 × `Grep` по `^type:` + сравнение с путём файла |

**CLI оправдан только для операций, использующих внутреннее состояние Obsidian:**
- `obsidian move` — обновляет wikilinks по всему vault
- `obsidian tags:rename` — массовое переименование тегов
- `obsidian property:set` — корректно форматирует YAML при записи
- `obsidian orphans` / `obsidian unresolved` — использует граф связей
- `obsidian links` / `obsidian backlinks` — использует граф связей

## Scan Area

Default folders to scan: `notes/`, `literature/`, `projects/`, `_moc/`.

Excluded: `_templates/`, `_assets/`, `Excalidraw/`, `daily/`, `_archive/`, `_inbox/`.

> `English/` is excluded from auto-scan. Include only if user explicitly requests.

## Hybrid Entry Point

All skills follow this pattern:

1. **User provided files/folder** → work only with those
2. **User provided nothing** → scan the full area, present candidates, wait for selection

## Candidate Output Format

Present candidates as a table (in Russian):

```
| # | Файл | Папка | Размер | Теги |
|---|------|-------|--------|------|
| 1 | RMSLE metric | notes/ | 2 KB | ML/metrics |
| 2 | ... | ... | ... | ... |
```

**Limit:** show top-20 by relevance. If more — ask user: show next batch or narrow filter.
