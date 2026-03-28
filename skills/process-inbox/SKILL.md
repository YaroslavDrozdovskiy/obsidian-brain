---
name: process-inbox
description: Process raw notes from Obsidian vault _inbox/ folder — standardize frontmatter, rename, tag, link, and move to correct folder. Use when user asks to process inbox, handle raw notes, or says /process-inbox.
---

# Process Inbox

Process all raw/temporary notes in the vault's `_inbox/` folder.

## Setup

1. Read `references/vault-config.md` from this plugin directory for vault path and folder rules.
2. Read `references/frontmatter-spec.md` for frontmatter templates and field rules.
3. Read `references/cli-operations.md` for CLI commands reference.
4. Read `references/interaction-patterns.md` for interaction and safety rules.

## Process

### 1. Scan Inbox

```bash
obsidian files path="_inbox/"
```

If empty — report «_inbox/ пуст, нечего обрабатывать» and stop.

### 2. For Each File

#### 2.1 Read and Analyze

```bash
obsidian read file="filename"
```

- Determine the topic and appropriate tags (hierarchical: `ML/metrics`)
- Determine the type: `note` (atomic concept), `literature` (course/book summary), `project` (project-related)
- If unclear, ask the user
- Generate filename by core concept: `RMSLE metric.md`, `Обработка пропусков.md`
  - Russian topic → Russian name, English topic → English name
  - No date prefixes

#### 2.2 Check for Name Conflicts

```bash
obsidian search query="filename"
```

If a file with this name exists in the target folder, ask the user: merge, rename with suffix, or skip.

#### 2.3 Find Related Notes

```bash
obsidian search query="keyword1" limit=10
obsidian search query="keyword2" limit=10
obsidian backlinks file="filename"
```

Use 2-3 key terms from the note content. Collect related note names for wikilinks.

#### 2.4 Present Plan

Show the user (in Russian):
- Proposed filename
- Target folder (notes/, literature/, projects/)
- Tags
- Related notes found (will be linked)

WAIT for user confirmation.

#### 2.5 Set Frontmatter

After confirmation, apply frontmatter per `references/frontmatter-spec.md`:

```bash
obsidian property:set file="filename" name="type" value="note"
obsidian property:set file="filename" name="tags" value="topic/subtopic"
obsidian property:set file="filename" name="status" value="active"
obsidian property:set file="filename" name="created" value="YYYY-MM-DD"
```

For `literature` type, also set `source`.

#### 2.6 Enrich Content

Add wikilinks to related concepts in the text body and append See also section:

```bash
obsidian append file="filename" content="\n## See also\n- [[Related Note 1]]\n- [[Related Note 2]]"
```

For inline wikilinks in the body text, use `Read` + `Edit` (CLI append only adds to the end).

#### 2.7 Move to Target Folder

```bash
obsidian move file="filename" to="notes/"
```

This automatically updates all wikilinks across the vault.

### 3. Report + Daily Log + Commit

Follow `references/interaction-patterns.md`:
- Output summary report
- Offer daily-log
- Commit:

```bash
cd "{vault_path}"
git add <processed files at new locations> <modified existing notes>
git commit -m "feat: process inbox notes"
```

## Multiple Files

Process one at a time: show plan → wait confirmation → process → next file.

## Important

- NEVER delete content from user's notes — only add structure
- Always preserve the original text
- If a note is ambiguous, ask the user
- Respond in Russian
