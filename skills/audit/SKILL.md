---
name: audit
description: Audit Obsidian vault quality and interactively fix issues. Use when user asks to audit vault, check quality, review notes, run healthcheck, or says /audit.
---

# Audit Vault Quality

Two modes:
- **quick** — 30-second healthcheck: orphans, broken links, singleton tags
- **full** — parallel scan across 5 categories with trust-level-driven fixes

## Setup

1. Read `references/vault-config.md` from this plugin directory for vault path and folder rules.
2. Read `references/frontmatter-spec.md` for frontmatter templates and field rules.
3. Read `references/cli-operations.md` for CLI commands reference.
4. Read `references/interaction-patterns.md` for interaction and safety rules.
5. Read `references/trust-levels.md` for trust level definitions and constraints.
6. Read `references/temp-file-spec.md` for JSON report format.

## Mode Selection

Ask the user:

```

Выберите режим аудита:

- **quick** — быстрая проверка (сироты, битые ссылки, singleton-теги) ~30 сек
- **full** — полный аудит с параллельным сканированием и интерактивными фиксами

```

---

## Mode: quick

Unchanged from v1 — run 3 fast sequential checks:

```bash
obsidian orphans
obsidian unresolved
obsidian tags
```

Parse tags output to find singletons. Output report. Offer daily log. No interactive fixes.

---

## Mode: full

### Phase 1: Parallel Scan (subagents)

Generate timestamp for this run: `ts=$(date +%Y-%m-%dT%H-%M)`

Dispatch 3 subagents in parallel using `Agent` tool with `subagent_type: "vault-worker"`:

> **CLI concurrency caveat:** Obsidian CLI read-safety for concurrent requests is not confirmed. CLI-dependent tasks (tags, links) are combined into one subagent. Grep/bash tasks run in separate subagents.

**Subagent 1: CLI scan (tags + links)**

Prompt:

```
Vault path: {vault_path}
Task: Scan tags and links for the entire vault.

1. Run `obsidian tags`. For each tag, check:
   - Singleton tags (used in only 1 note, no related hierarchy) — verify with `obsidian tag tag="#name"`
   - Casing duplicates (same tag, different case)
   - Non-English tags (all tags must be English per spec)
   - Flat hierarchy candidates (could become hierarchical)

2. Run `obsidian orphans` and `obsidian unresolved`.

Categories: singleton_tag, casing_duplicate, non_english_tag, flat_hierarchy_candidate, orphan_note, unresolved_link
Severity: orphan_note=medium, unresolved_link=medium, singleton_tag=low, casing_duplicate=medium, non_english_tag=high, flat_hierarchy_candidate=low

Write JSON report to: {vault_path}/_temp/audit-cli-scan-{ts}.json
```

**Subagent 2: Frontmatter + placement (Grep)**

Prompt:

```
Vault path: {vault_path}
Task: Scan frontmatter fields and file placement across notes/, literature/, projects/, _moc/.

1. Use Grep to bulk-scan frontmatter fields:
   Grep pattern="^(type|tags|status|created|source):" across each scan folder.
   Also Grep pattern="^---$" to find files without any frontmatter.

2. For each file, check required fields per type (see Frontmatter Spec in your instructions):
   - note: type, tags, status, created required
   - literature: type, tags, status, source, created required
   - project: type, tags, status, created required (source required)
   - moc: type, tags, created required
   - daily: type, created required

3. For missing `created` field: try `git log --follow --diff-filter=A --format=%ai -- "{file}"` to extract original creation date.

4. Check placement: compare `type:` value against folder path:
   - notes/ → type: note
   - literature/ → type: literature
   - projects/ → type: project (or moc for project-local MOCs)
   - _moc/ → type: moc

Categories: missing_field, invalid_value, wrong_type, wrong_folder, type_folder_mismatch, no_frontmatter
Severity: no_frontmatter=high, missing_field=high (for required), invalid_value=medium, wrong_folder=medium, type_folder_mismatch=low

For suggested_fix on missing_field: provide exact `property:set` command with inferred value.

Write JSON report to: {vault_path}/_temp/audit-structure-{ts}.json
```

**Subagent 3: Content scan (Grep/bash)**

Prompt:

```
Vault path: {vault_path}
Task: Scan content quality across notes/, literature/, projects/, _moc/.

1. Word count: run `find "{vault_path}/notes" "{vault_path}/literature" "{vault_path}/projects" "{vault_path}/_moc" -name "*.md" -exec wc -w {} +`

2. Headings: Grep for `^## ` in each scan folder — find files WITHOUT any headings.

3. See also: Grep for `^## See also` — find files WITHOUT See also section (exclude type: moc and type: daily).

Check thresholds:
- Too short: <50 words (exclude moc)
- Too long: >2000 words — candidate for /decompose
- No headings: file has content but no ## headings
- No See also: file is type note/literature/project but has no See also section

Categories: too_short, too_long_decompose_candidate, no_headings, no_see_also
Severity: too_short=low, too_long_decompose_candidate=medium, no_headings=low, no_see_also=low

Write JSON report to: {vault_path}/_temp/audit-content-{ts}.json
```

### Phase 2: Merge and Present

After all 3 subagents complete:

1. Check for errors — if any subagent failed, inform user per `references/interaction-patterns.md` subagent error handling pattern.
2. Read all successful JSON reports via `Read` tool.
3. Merge issues into unified list, sort by severity (high → medium → low).
4. Count totals by category. Present summary:

```
## Аудит завершён

Просканировано: {total_files} заметок за ~{time}

Найдено {total_issues} проблем:
| Категория | Кол-во | Severity |
|-----------|--------|----------|
| Отсутствующие поля frontmatter | 87 | high |
| Некорректные теги | 15 | high |
| Файлы не в той папке | 12 | medium |
| Сироты | 23 | medium |
| Битые ссылки | 8 | medium |
| Нет See also | 42 | low |
| ... | ... | ... |
```

1. Ask trust level per `references/trust-levels.md` (default: `balanced`).

### Phase 3: Apply Fixes

Apply per trust level. For each category/issue:

**Frontmatter fixes:**

```bash
obsidian property:set file="name" name="type" value="note"
obsidian property:set file="name" name="status" value="draft"
obsidian property:set file="name" name="created" value="YYYY-MM-DD"
```

**Tag fixes:**

- Suggest /rename-tags for bulk operations
- For singletons: confirm intentional or suggest hierarchy

**Link fixes:**

- Orphans: search related notes, suggest See also entries

```bash
obsidian search query="keyword from orphan title" limit=5
obsidian append file="orphan" content="\n## See also\n- [[Related Note]]"
```

- Broken links: use `Read` + `Edit` to fix or remove

**Placement fixes:**

```bash
obsidian move file="misplaced" to="correct-folder/"
```

> Constraint: `obsidian move` requires minimum `balanced` trust level.

**Content flags:**

- Too long → suggest `/decompose` (do NOT invoke automatically)
- Too short → `obsidian property:set file="name" name="status" value="draft"`
- No structure → flag for user
- No See also → suggest related notes

### Phase 4: Cleanup + Report

1. Ask user: "Удалить временные файлы аудита? (да/нет)" In `auto` mode — delete automatically.
2. Delete `_temp/audit-*.json` files if confirmed.
3. Output report per `references/interaction-patterns.md`.
4. Offer daily log.
5. Commit per interaction-patterns convention.

## Important

- NEVER delete notes
- NEVER rename without confirmation
- Audit does NOT invoke other skills — only suggests
- Respond in Russian
