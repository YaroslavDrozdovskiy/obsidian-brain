---
name: decompose
description: Decompose a monolithic Obsidian note into atomic notes + MOC. Use when user asks to break apart, split, or decompose a large note, or says /decompose.
---

# Decompose Monolithic Note

Split a large monolithic note into atomic notes and convert the original into a MOC.

## Setup

1. Read `references/vault-config.md` from this plugin directory for vault path and folder rules.
2. Read `references/frontmatter-spec.md` for frontmatter templates and field rules.
3. Read `references/cli-operations.md` for CLI commands reference.
4. Read `references/interaction-patterns.md` for interaction and safety rules.

## Input

User provides a path or filename. If not provided, find candidates >10KB:

```bash
obsidian files path="literature/"
obsidian files path="notes/"
```

Then check sizes via `bash stat` and present candidates >10KB as a table.

## Process

### 1. Analyze Structure

```bash
obsidian read file="filename"
```

- Identify logical sections by headings (##, ###)
- Assess each section: self-contained concept? 100-2000 words? Useful standalone?
- Determine atomic note titles

### 2. Propose Decomposition Plan

Present in Russian:

```
## План декомпозиции: [Original File Name]

Оригинал: [size] KB, [N] секций

Предлагаемые атомарные заметки:
1. **[Note Title]** — [1-line description] → `notes/[filename].md`
   Теги: [tag1], [tag2]
...

Секции, остающиеся в MOC:
- [Section name] — [reason]

Оригинальный файл станет MOC.
```

### 3. WAIT FOR USER APPROVAL

Do NOT proceed without explicit confirmation.

### 4. Execute

For each approved atomic note:

**Create the note:**

```bash
obsidian create name="notes/Note Title" content="[extracted content with ## See also section]" silent
```

**Set frontmatter:**

```bash
obsidian property:set file="Note Title" name="type" value="note"
obsidian property:set file="Note Title" name="tags" value="topic/subtopic"
obsidian property:set file="Note Title" name="source" value="Original File Name"
obsidian property:set file="Note Title" name="status" value="active"
obsidian property:set file="Note Title" name="created" value="YYYY-MM-DD"
```

**Update the original file:**

If >70% decomposed — convert to MOC:

```bash
obsidian property:set file="original" name="type" value="moc"
obsidian property:remove file="original" name="status"
```

Then use `Read` + `Edit` to replace body with structured links to created notes.

If ≤70% — use `Edit` to replace extracted sections with `→ See [[Atomic Note Title]]`.

**Move MOC if needed:**

```bash
obsidian move file="original" to="_moc/"
```

### 5. Report + Daily Log + Commit

Follow `references/interaction-patterns.md`.

```bash
cd "{vault_path}"
git add <created atomic notes> <modified original>
git commit -m "refactor: decompose [original filename] into atomic notes"
```

## Atomicity Criteria

- ONE concept, technique, or idea
- Understandable without surrounding sections
- 100-2000 words
- Useful in a different context

Stay in MOC if: <50 words, only contextual, table of contents.

## Important

- NEVER delete the original file — convert to MOC
- NEVER rename the original file — preserve existing links
- Always wait for user approval
- Respond in Russian
