---
name: rename-tags
description: Interactively refactor tag taxonomy in Obsidian vault. Use when user asks to rename tags, clean up tags, refactor taxonomy, or says /rename-tags.
---

# Rename Tags

Interactive refactoring of the vault's tag taxonomy.

## Setup

1. Read `references/vault-config.md` from this plugin directory for vault path and folder rules.
2. Read `references/cli-operations.md` for CLI commands reference.
3. Read `references/interaction-patterns.md` for interaction and safety rules.
4. Read `references/trust-levels.md` for trust level definitions and constraints.

## Process

### 1. Collect Tag Data

```bash
obsidian tags
```

Parse output into a table: tag name, count of notes, nesting level.

### 2. Identify Problems

Automatically detect:

**Singleton tags** — used in only 1 note with no related tags in the same hierarchy:

```bash
obsidian tag tag="#singleton-tag"
```

Verify it's truly isolated.

**Casing duplicates** — same tag with different casing (`ML` vs `ml`, `MachineLearning` vs `machine-learning`).

**Flat candidates** — tags that could become hierarchical (e.g., `python` → `dev/python`). Suggest based on existing hierarchy patterns.

### 3. Present Analysis

```
## Анализ тегов vault

Всего тегов: N (в M заметках)

### Проблемы

#### Singleton-теги (1 заметка, нет связей в иерархии)
| # | Тег | Заметка |
|---|-----|---------|
| 1 | #physics/rare-topic | Rare Topic.md |

#### Дубли (разный casing)
| # | Вариант 1 | Вариант 2 | Заметок |
|---|-----------|-----------|---------|
| 1 | ML | ml | 12 |

#### Кандидаты на иерархию
| # | Текущий | Предлагаемый | Заметок |
|---|---------|-------------|---------|
| 1 | python | dev/python | 5 |

### План переименований
| # | Старый | Новый | Затронуто заметок |
|---|--------|-------|-------------------|
| 1 | ml | ML | 3 |
| 2 | python | dev/python | 5 |
```

### 4. WAIT FOR USER APPROVAL

User can:
- Approve full plan
- Edit individual renames
- Remove items from the plan
- Cancel

### 4b. Ask Trust Level

```

Уровень подтверждения:

- **cautious** — подтверждение каждого переименования
- **balanced** — подтверждение всей таблицы переименований разом (рекомендуется)
- **auto** — применить все переименования без подтверждений

[balanced]

```

- `cautious`: confirm each rename individually
- `balanced` (default): confirm the full rename table (step 4 approval serves as confirmation)
- `auto`: apply all renames from the approved table without individual confirmation

> In `balanced` and `auto` modes, step 4 user approval of the rename table IS the confirmation — proceed directly to execution.

### 5. Execute Renames

For each approved rename:

```bash
obsidian tags:rename old="oldname" new="newname"
```

> `tags:rename` updates all files containing the tag at once. No manual grep+replace needed.

### 6. Verify

```bash
obsidian tags
```

Confirm renamed tags appear correctly and old tags are gone.

### 7. Report + Daily Log + Commit

Follow `references/interaction-patterns.md`.

```bash
cd "{vault_path}"
git add <all files modified by tag renames>
git commit -m "refactor: rename tags"
```

## Scope

- Does NOT delete tags — only renames
- Does NOT touch files in `English/` and `_archive/` (excluded from scan area)
- Respond in Russian
