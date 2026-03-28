# obsidian-brain CLI Refactor — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Переписать все скиллы obsidian-brain на Obsidian CLI 1.12.4, добавить новые возможности (analyze, quick audit, rename-tags), обновить README.

**Architecture:** CLI-wrapper подход — замена файловых операций (Glob, Grep, bash mv/find) на CLI-команды Obsidian (files, search, move, properties, tags, backlinks, orphans). Прямой доступ к файлам сохраняется для массового чтения, сложных regex, word count. eval используется ограниченно.

**Tech Stack:** Obsidian CLI 1.12.4, Bash, Git

**Spec:** `docs/superpowers/specs/2026-03-26-cli-refactor-design.md`

**Plugin path:** `C:\Users\yasdr\.claude\plugins\obsidian-brain\`

---

## Task 1: Восстановить отсутствующие reference-файлы

**Контекст:** `vault-config.md` и `frontmatter-spec.md` существуют в cache (`plugins/cache/user-config/obsidian-brain/9fea0fe2e8f6/references/`) но отсутствуют в рабочей директории плагина. Все скиллы зависят от этих файлов.

**Files:**
- Create: `references/vault-config.md`
- Create: `references/frontmatter-spec.md`

- [ ] **Step 1: Скопировать vault-config.md из cache**

```bash
cp "/c/Users/yasdr/.claude/plugins/cache/user-config/obsidian-brain/9fea0fe2e8f6/references/vault-config.md" "/c/Users/yasdr/.claude/plugins/obsidian-brain/references/vault-config.md"
```

- [ ] **Step 2: Скопировать frontmatter-spec.md из cache**

```bash
cp "/c/Users/yasdr/.claude/plugins/cache/user-config/obsidian-brain/9fea0fe2e8f6/references/frontmatter-spec.md" "/c/Users/yasdr/.claude/plugins/obsidian-brain/references/frontmatter-spec.md"
```

- [ ] **Step 3: Проверить файлы на месте**

```bash
ls -la "/c/Users/yasdr/.claude/plugins/obsidian-brain/references/"
```

Expected: 5 файлов — vault-config.md, frontmatter-spec.md, scan-patterns.md, interaction-patterns.md, и позже cli-operations.md

- [ ] **Step 4: Commit**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add references/vault-config.md references/frontmatter-spec.md
git commit -m "fix: restore missing reference files (vault-config, frontmatter-spec)"
```

---

## Task 2: Создать cli-operations.md

**Контекст:** Этот файл — фундамент для всех скиллов. Заменяет `scan-patterns.md`. Содержит маппинг операций на CLI-команды и правила, когда использовать прямой доступ к файлам.

**Files:**
- Create: `references/cli-operations.md`

- [ ] **Step 1: Создать cli-operations.md**

Записать файл `references/cli-operations.md` со следующим содержимым:

```markdown
# CLI Operations

Common CLI operations used by all skills. Read this reference at the start of any skill that interacts with the vault.

**Requirement:** Obsidian must be running for CLI commands to work.

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
```

- [ ] **Step 2: Commit**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add references/cli-operations.md
git commit -m "feat: add cli-operations.md reference (replaces scan-patterns.md)"
```

---

## Task 3: Обновить interaction-patterns.md

**Контекст:** Добавить паттерн daily-log и обновить commit convention для нового скилла rename-tags.

**Files:**
- Modify: `references/interaction-patterns.md`

- [ ] **Step 1: Добавить daily-log паттерн и обновить commit convention**

Заменить содержимое `references/interaction-patterns.md` на:

```markdown
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

```bash
obsidian daily:append content="\n### [Skill name] — HH:MM\n- Обработано: N заметок\n- Изменено: M\n- [краткий список действий]"
```

Use the current time (HH:MM) and actual counts from the report.

## Commit Convention

Each skill ends with a git commit. Use `git add <specific files>`, NOT `git add -A`. Only add files changed during the current operation.

Commit messages by skill:
- process-inbox: `feat: process inbox notes`
- decompose: `refactor: decompose [original filename] into atomic notes`
- dedup: `refactor: merge duplicate notes`
- cluster: `feat: create [MOC/summary] for [topic]`
- audit: `fix: audit and fix notes quality`
- rename-tags: `refactor: rename tags`

## Safety Rules

- NEVER delete files — always archive or convert
- NEVER rename files without user confirmation (wikilinks may break)
- NEVER proceed without explicit user approval
- Preserve all original content — nothing gets lost
```

- [ ] **Step 2: Commit**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add references/interaction-patterns.md
git commit -m "feat: add daily-log pattern and rename-tags commit convention"
```

---

## Task 4: Переписать process-inbox на CLI

**Files:**
- Modify: `skills/process-inbox/SKILL.md`

- [ ] **Step 1: Переписать SKILL.md**

Заменить содержимое `skills/process-inbox/SKILL.md` на:

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add skills/process-inbox/SKILL.md
git commit -m "refactor: rewrite process-inbox skill to use Obsidian CLI"
```

---

## Task 5: Переписать decompose на CLI

**Files:**
- Modify: `skills/decompose/SKILL.md`

- [ ] **Step 1: Переписать SKILL.md**

Заменить содержимое `skills/decompose/SKILL.md` на:

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add skills/decompose/SKILL.md
git commit -m "refactor: rewrite decompose skill to use Obsidian CLI"
```

---

## Task 6: Переписать dedup на CLI

**Files:**
- Modify: `skills/dedup/SKILL.md`

- [ ] **Step 1: Переписать SKILL.md**

Заменить содержимое `skills/dedup/SKILL.md` на:

```markdown
---
name: dedup
description: Find and merge duplicate notes in the Obsidian vault. Use when user asks to deduplicate, merge duplicates, find duplicates, or says /dedup.
---

# Deduplicate Notes

Find duplicate notes and merge them, archiving the secondary copy.

## Setup

1. Read `references/vault-config.md` from this plugin directory for vault path and folder rules.
2. Read `references/frontmatter-spec.md` for frontmatter templates and field rules.
3. Read `references/cli-operations.md` for CLI commands reference.
4. Read `references/interaction-patterns.md` for interaction and safety rules.

## Input

User provides specific files to compare, or nothing. If nothing — scan the entire vault per hybrid entry point in `references/cli-operations.md`.

## Process

### 1. Find Duplicates

Three-stage algorithm:

**Stage A — Filenames:**

```bash
obsidian files path="notes/"
obsidian files path="literature/"
obsidian files path="projects/"
```

Tokenize filenames, find pairs with shared significant tokens (>= 1 token longer than 3 chars).

**Stage B — Tags:**

For candidate pairs from Stage A, compare tags:

```bash
obsidian properties file="note1"
obsidian properties file="note2"
```

Additionally, find pairs with identical tags not caught in Stage A:

```bash
obsidian tag tag="#specific-tag"
```

**Stage C — Content:**

For candidates from A and B, extract 5-10 characteristic phrases and search:

```bash
obsidian search:context query="characteristic phrase" limit=5
```

Count percentage of matched phrases.

**Ranking:**
- High: all 3 signals (name + tags + content)
- Medium: 2 signals
- Low: 1 signal

Semantically evaluate the final list — filter out false positives.

### 2. Present Candidates

Show in Russian with confidence levels and overlap description.

### 3. WAIT FOR USER APPROVAL

Do NOT proceed without explicit confirmation.

### 4. Interactive Merge

For each approved group (interactive mode per `references/interaction-patterns.md`):

1. Show content of both notes:

```bash
obsidian read file="note1"
obsidian read file="note2"
```

2. Propose primary note (more complete or newer)
3. Propose merge plan
4. User chooses: merge / skip / custom

### 5. Execute Merge

**Merge content** into primary note using `Read` + `Edit` (complex content merging requires precise editing).

**Update frontmatter:**

```bash
obsidian property:set file="primary" name="tags" value="merged-tags"
obsidian property:set file="primary" name="created" value="earliest-date"
```

**Archive secondary:**

```bash
obsidian property:set file="secondary" name="merged_into" value="[[Primary Note]]"
obsidian property:set file="secondary" name="status" value="archive"
obsidian move file="secondary" to="_archive/merged/"
```

> `move` automatically updates all wikilinks to the secondary note across the vault. This replaces the old manual grep+replace workflow.

On name conflict in `_archive/merged/` — add date suffix before moving.

### 6. Report + Daily Log + Commit

Follow `references/interaction-patterns.md`.

```bash
cd "{vault_path}"
git add <modified primary notes> <archived secondary notes> <files with updated wikilinks>
git commit -m "refactor: merge duplicate notes"
```

## Important

- NEVER delete notes — archive to `_archive/merged/`
- Always wait for user approval before merging
- Preserve all original content
- Respond in Russian
```

- [ ] **Step 2: Commit**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add skills/dedup/SKILL.md
git commit -m "refactor: rewrite dedup skill to use Obsidian CLI"
```

---

## Task 7: Переписать и расширить cluster на CLI

**Контекст:** Добавляется режим `analyze` (диагностика графа). Существующий функционал становится режимом `group`.

**Files:**
- Modify: `skills/cluster/SKILL.md`

- [ ] **Step 1: Переписать SKILL.md**

Заменить содержимое `skills/cluster/SKILL.md` на:

```markdown
---
name: cluster
description: Group related notes into MOC or summary note, or analyze vault graph structure. Use when user asks to cluster notes, group notes, create MOC, analyze graph, find orphans/hubs, or says /cluster.
---

# Cluster & Graph Analysis

Two modes:
- **analyze** — diagnose vault graph: hubs, orphans, dead-ends, weak clusters
- **group** — find related notes and organize into MOC or summary

## Setup

1. Read `references/vault-config.md` from this plugin directory for vault path and folder rules.
2. Read `references/frontmatter-spec.md` for frontmatter templates and field rules.
3. Read `references/cli-operations.md` for CLI commands reference.
4. Read `references/interaction-patterns.md` for interaction and safety rules.

## Mode Selection

Ask the user:

```
Выберите режим:
- **analyze** — диагностика графа (хабы, сироты, тупики, слабые связи)
- **group** — группировка заметок в MOC или обобщающую заметку
```

---

## Mode: analyze

### 1. Collect Graph Data

```bash
obsidian orphans
obsidian unresolved
```

For each note in scan area, get backlinks count:

```bash
obsidian files path="notes/"
obsidian files path="literature/"
obsidian files path="projects/"
obsidian files path="_moc/"
```

Then for each file:

```bash
obsidian backlinks file="name"
obsidian links file="name"
```

> If vault has >100 notes and individual backlinks calls are too slow, use eval:

```bash
obsidian eval code="JSON.stringify(Object.fromEntries(Object.entries(app.metadataCache.resolvedLinks).map(([k,v])=>[k,Object.keys(v)])))"
```

### 2. Build Report

Present in Russian:

```
## Анализ графа vault

Всего заметок: N

### Хабы (топ-10 по входящим ссылкам)
| # | Заметка | Входящие | Исходящие |
|---|---------|----------|-----------|
| 1 | ... | ... | ... |

### Сироты (нет входящих И исходящих)
- note1.md
- note2.md

### Тупики (есть входящие, нет исходящих)
- note3.md (← 5 входящих)

### Битые ссылки
- [[Missing Note]] ← ссылается из: file1.md, file2.md

### Слабые кластеры (2-3 заметки, связаны только между собой)
- {note4, note5} — изолированная пара
```

### 3. Recommendations

For orphans and weak clusters, suggest actions:
- Create links to existing notes
- Add to existing MOC
- Run cluster:group to create new MOC
- Archive if outdated

Do NOT create/move files in analyze mode — only diagnostics and recommendations.

### 4. Daily Log

Offer to log results per `references/interaction-patterns.md`.

---

## Mode: group

### 1. Identify Clusters

**By tags:**

```bash
obsidian tags
obsidian tag tag="#specific-tag"
```

Find tag groups without corresponding MOC in `_moc/`.

**By links:**

```bash
obsidian links file="name"
obsidian backlinks file="name"
```

Connected components without a hub.

**By content:**

```bash
obsidian search query="specific term" limit=20
```

Notes with overlapping key terms (>= 3 shared specific terms).

**Filter:** clusters from 3+ notes only.

### 2. Present Candidates

Show in Russian with recommendation (MOC vs summary):

- **MOC** — notes describe different aspects of one topic (enumeration), no single thesis
- **Summary** — content reducible to one common thesis, overlapping meaning

### 3. WAIT FOR USER APPROVAL

User confirms composition and result type for each cluster.

### 4. Create MOC (if chosen)

```bash
obsidian create name="_moc/Topic Name" content="# Topic Name\n\n## Заметки\n\n- [[Note 1]] — annotation\n- [[Note 2]] — annotation\n..." silent
obsidian property:set file="Topic Name" name="type" value="moc"
obsidian property:set file="Topic Name" name="tags" value="topic"
obsidian property:set file="Topic Name" name="created" value="YYYY-MM-DD"
```

Add backlinks to each cluster note:

```bash
obsidian append file="Note 1" content="\n\n## See also\n- [[Topic Name]]"
```

> If note already has `## See also`, use `Read` + `Edit` to add link to existing section instead of appending a duplicate section.

### 5. Create Summary Note (if chosen)

```bash
obsidian create name="notes/Summary Title" content="[synthesized text with [[wikilinks]] to source notes]" silent
obsidian property:set file="Summary Title" name="type" value="note"
obsidian property:set file="Summary Title" name="tags" value="topic/subtopic"
obsidian property:set file="Summary Title" name="status" value="active"
obsidian property:set file="Summary Title" name="created" value="YYYY-MM-DD"
```

Add backlinks to each cluster note (same as MOC step).

### 6. Report + Daily Log + Commit

Follow `references/interaction-patterns.md`.

```bash
cd "{vault_path}"
git add <created MOC/summary> <modified cluster notes>
git commit -m "feat: create [MOC/summary] for [topic]"
```

## Important

- NEVER delete notes
- Always wait for user approval
- Preserve all original content
- Respond in Russian
```

- [ ] **Step 2: Commit**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add skills/cluster/SKILL.md
git commit -m "refactor: rewrite cluster skill with analyze mode, use Obsidian CLI"
```

---

## Task 8: Переписать и расширить audit на CLI

**Контекст:** Добавляется режим `quick` (быстрый healthcheck). Существующий функционал становится режимом `full`.

**Files:**
- Modify: `skills/audit/SKILL.md`

- [ ] **Step 1: Переписать SKILL.md**

Заменить содержимое `skills/audit/SKILL.md` на:

```markdown
---
name: audit
description: Audit Obsidian vault quality and interactively fix issues. Use when user asks to audit vault, check quality, review notes, run healthcheck, or says /audit.
---

# Audit Vault Quality

Two modes:
- **quick** — 30-second healthcheck: orphans, broken links, singleton tags
- **full** — complete audit across 5 categories with interactive fixes

## Setup

1. Read `references/vault-config.md` from this plugin directory for vault path and folder rules.
2. Read `references/frontmatter-spec.md` for frontmatter templates and field rules.
3. Read `references/cli-operations.md` for CLI commands reference.
4. Read `references/interaction-patterns.md` for interaction and safety rules.

## Mode Selection

Ask the user:

```
Выберите режим аудита:
- **quick** — быстрая проверка (сироты, битые ссылки, singleton-теги) ~30 сек
- **full** — полный аудит (frontmatter, теги, ссылки, размещение, контент) с фиксами
```

---

## Mode: quick

### 1. Run 3 Fast Checks

```bash
obsidian orphans
obsidian unresolved
obsidian tags
```

Parse `obsidian tags` output to find singleton tags (used in only 1 note).

### 2. Output Report

```
## Быстрый аудит vault

Сироты: N заметок
Битые ссылки: M
Singleton-теги: K

Для подробностей и исправлений запустите /audit → full
```

### 3. Daily Log

Offer to log per `references/interaction-patterns.md`.

No interactive fixes in quick mode.

---

## Mode: full

### 1. Run All Checks

**A. Frontmatter:**

For each file in scan area:

```bash
obsidian properties file="name"
```

Check:
- Missing frontmatter entirely
- Missing required fields (type, tags, status, created) per `references/frontmatter-spec.md`
- Invalid values (type ∉ {note|literature|project|moc|daily}, status ∉ {draft|active|archive}, created not YYYY-MM-DD)
- Template mismatch by type (e.g., literature without source)

**B. Tags:**

```bash
obsidian tags
```

Check:
- Notes without tags (except type: daily)
- Singleton tags (1 note, no related hierarchy)
- Inconsistent casing (`ML` vs `ml`)

**C. Links:**

```bash
obsidian orphans
obsidian unresolved
```

For missing `## See also` — use `Read` to check each file's content.

**D. Placement:**

```bash
obsidian files path="notes/"
obsidian files path="literature/"
obsidian files path="projects/"
obsidian files path="_moc/"
```

For each file, compare `obsidian properties file="name"` type vs actual folder.

**E. Content:**

```bash
obsidian read file="name"
```

Plus `bash wc -w` for word count. Check:
- Too short (<50 words, except moc)
- Too long (>3000 words) — candidate for /decompose
- Empty sections
- No ## headings

### 2. Summary Report

```
## Аудит хранилища

Проверено: N заметок

Проблемы найдены: M
- Frontmatter: X
- Теги: Y
- Связи: Z
- Размещение: W
- Содержимое: V

Начать исправление? (да / только категория X / нет)
```

### 3. WAIT FOR USER CHOICE

### 4. Interactive Fix Mode

Per `references/interaction-patterns.md` interactive mode.

**Frontmatter fixes:**

```bash
obsidian property:set file="name" name="type" value="note"
obsidian property:set file="name" name="status" value="draft"
obsidian property:set file="name" name="created" value="YYYY-MM-DD"
```

Infer type from folder, created from file modification date.

**Tag fixes:**
- Suggest unification: `obsidian tags:rename old="ml" new="ML"` (or suggest /rename-tags for bulk operations)
- For singletons: confirm intentional or suggest hierarchy

**Link fixes:**
- Orphans: search related notes and suggest wikilinks

```bash
obsidian search query="keyword from orphan title" limit=5
```

Propose `## See also` entries. Apply:

```bash
obsidian append file="orphan" content="\n## See also\n- [[Related Note]]"
```

- Broken wikilinks: use `Read` + `Edit` to fix or remove

**Placement fixes:**

```bash
obsidian move file="misplaced" to="correct-folder/"
```

**Content flags:**
- Too long → suggest `/decompose` (do NOT invoke automatically)
- Too short → `obsidian property:set file="name" name="status" value="draft"`
- Empty sections → flag for user
- No structure → suggest headings

### 5. Report + Daily Log + Commit

Follow `references/interaction-patterns.md`.

```bash
cd "{vault_path}"
git add <modified files>
git commit -m "fix: audit and fix notes quality"
```

## Important

- NEVER delete notes
- NEVER rename without confirmation
- Audit does NOT invoke other skills — only suggests
- Respond in Russian
```

- [ ] **Step 2: Commit**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add skills/audit/SKILL.md
git commit -m "refactor: rewrite audit skill with quick mode, use Obsidian CLI"
```

---

## Task 9: Создать скилл rename-tags

**Files:**
- Create: `skills/rename-tags/SKILL.md`

- [ ] **Step 1: Создать директорию и SKILL.md**

```bash
mkdir -p "/c/Users/yasdr/.claude/plugins/obsidian-brain/skills/rename-tags"
```

Записать файл `skills/rename-tags/SKILL.md`:

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add skills/rename-tags/SKILL.md
git commit -m "feat: add rename-tags skill"
```

---

## Task 10: Обновить README.md

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Переписать README.md**

Заменить содержимое `README.md` на:

```markdown
# obsidian-brain

Плагин для Claude Code — управление Obsidian vault через CLI.

## Требования

- Obsidian >= 1.12.4 с включённым CLI
- Obsidian должен быть запущен при использовании скиллов
- Claude Code с установленным плагином

## Установка

1. Включить CLI в Obsidian: Settings → General → Command line interface → Register CLI
2. Проверить работу: `obsidian version` в терминале
3. Установить плагин в Claude Code

## Скиллы

### /process-inbox — Обработка входящих заметок

Разбирает сырые заметки из `_inbox/`: определяет тему, проставляет frontmatter и теги, находит связанные заметки, добавляет wikilinks, перемещает в целевую папку с автоматическим обновлением ссылок.

**Типичный сценарий:**

1. Накидываете заметки в `_inbox/` (после митинга, лекции, чтения статьи)
2. Запускаете `/process-inbox`
3. Для каждой заметки плагин показывает: предложенное имя, теги, папку, найденные связи
4. Подтверждаете или корректируете план
5. Заметки разложены по папкам, связаны wikilinks, проиндексированы Obsidian

### /decompose — Декомпозиция монолитных заметок

Разбивает большие заметки (>10KB) на атомарные заметки + создаёт Map of Content (MOC).

**Типичный сценарий:**

1. Конспект книги на 5000 слов в `literature/`
2. Запускаете `/decompose` (или плагин сам предложит кандидатов)
3. Видите план: какие секции станут отдельными заметками, какие останутся в MOC
4. Подтверждаете
5. Результат: 8 атомарных заметок в `notes/` + MOC в `_moc/` со ссылками

### /dedup — Поиск и слияние дубликатов

Трёхстадийный поиск дубликатов (по именам → тегам → содержимому) с интерактивным слиянием.

**Типичный сценарий:**

1. Запускаете `/dedup`
2. Плагин находит пары с разной степенью уверенности (высокая/средняя/низкая)
3. Для каждой пары: видите оба файла, выбираете основной
4. Дубль архивируется в `_archive/merged/`, все ссылки обновляются автоматически

### /cluster — Кластеризация и анализ графа

Два режима работы:

**Режим `analyze` — диагностика графа:**

1. Запускаете `/cluster` → выбираете `analyze`
2. Получаете отчёт: хабы (самые связанные заметки), сироты, тупики, битые ссылки, слабые кластеры
3. Для каждой проблемы — рекомендация (создать связи, добавить в MOC, архивировать)

**Режим `group` — группировка в MOC/summary:**

1. Запускаете `/cluster` → выбираете `group`
2. Плагин находит кластеры связанных заметок (по тегам, ссылкам, содержимому)
3. Для каждого кластера предлагает: MOC (разные аспекты темы) или обобщающую заметку (общий тезис)
4. Подтверждаете состав и тип → MOC в `_moc/` или summary в `notes/`

### /audit — Аудит качества vault

Два режима работы:

**Режим `quick` — healthcheck за 30 секунд:**

1. Запускаете `/audit` → выбираете `quick`
2. Три быстрые проверки: сироты, битые ссылки, singleton-теги
3. Краткий отчёт с числами

**Режим `full` — полный аудит с фиксами:**

1. Запускаете `/audit` → выбираете `full`
2. Проверка по 5 категориям: frontmatter, теги, ссылки, размещение, контент
3. Сводный отчёт с количеством проблем по категориям
4. Выбираете: исправить всё / только категорию X / пропустить
5. Интерактивный режим: для каждой проблемы — предложение фикса → подтверждение

### /rename-tags — Рефакторинг тегов

Интерактивный рефакторинг таксономии тегов vault.

**Типичный сценарий:**

1. Запускаете `/rename-tags`
2. Видите полное дерево тегов с количеством заметок
3. Плагин выявляет: singleton-теги, дубли с разным casing, плоские теги-кандидаты на иерархию
4. Утверждаете план переименований (можете отредактировать)
5. Теги переименованы во всём vault одной командой

## Логирование в Daily Note

После завершения любого скилла предлагается записать результат в Daily Note. Формат записи:

```
### [Skill name] — HH:MM
- Обработано: N заметок
- Изменено: M
- [краткий список действий]
```

## Структура vault

```
_inbox/        → Входящие заметки (обрабатываются /process-inbox)
notes/         → Атомарные заметки
literature/    → Конспекты книг/курсов/статей
projects/      → Проектные заметки
_moc/          → Map of Content (навигационные файлы)
daily/         → Ежедневные заметки
_archive/      → Архив (неактуальные + слитые дубли)
_templates/    → Шаблоны Obsidian
_assets/       → Вложения (изображения, PDF)
```

## Безопасность

- Файлы никогда не удаляются — только архивируются
- Перемещение и переименование — только после подтверждения пользователя
- Wikilinks обновляются автоматически через CLI при перемещении
- Всё содержимое сохраняется — ничего не теряется
- Git коммиты после каждой операции с указанием конкретных файлов

## Конфигурация

- `references/vault-config.md` — Путь к vault, структура папок, правила именования
- `references/frontmatter-spec.md` — Шаблоны frontmatter и правила полей
- `references/cli-operations.md` — Справочник CLI-команд Obsidian
- `references/interaction-patterns.md` — Паттерны взаимодействия (язык, подтверждения, отчёты)
```

- [ ] **Step 2: Commit**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add README.md
git commit -m "docs: rewrite README in Russian with CLI-based workflow"
```

---

## Task 11: Удалить scan-patterns.md и финальный коммит

**Контекст:** `scan-patterns.md` полностью заменён на `cli-operations.md`. Удаляем старый файл.

**Files:**
- Delete: `references/scan-patterns.md`

- [ ] **Step 1: Удалить scan-patterns.md**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git rm references/scan-patterns.md
```

- [ ] **Step 2: Commit**

```bash
git commit -m "refactor: remove scan-patterns.md (replaced by cli-operations.md)"
```

- [ ] **Step 3: Проверить финальную структуру**

```bash
ls -R "/c/Users/yasdr/.claude/plugins/obsidian-brain/"
```

Expected structure:
```
.claude-plugin/plugin.json
README.md
docs/superpowers/specs/2026-03-26-cli-refactor-design.md
docs/superpowers/plans/2026-03-26-cli-refactor-plan.md
references/vault-config.md
references/frontmatter-spec.md
references/cli-operations.md
references/interaction-patterns.md
skills/process-inbox/SKILL.md
skills/decompose/SKILL.md
skills/dedup/SKILL.md
skills/cluster/SKILL.md
skills/audit/SKILL.md
skills/rename-tags/SKILL.md
```

- [ ] **Step 4: Обновить plugin.json**

Обновить описание в `.claude-plugin/plugin.json`:

```json
{
  "name": "obsidian-brain",
  "description": "Управление Obsidian vault через CLI: обработка inbox, декомпозиция, дедупликация, кластеризация и анализ графа, аудит качества, рефакторинг тегов",
  "author": {
    "name": "YaroslavDrozdovskiy"
  }
}
```

- [ ] **Step 5: Финальный коммит**

```bash
cd "/c/Users/yasdr/.claude/plugins/obsidian-brain"
git add .claude-plugin/plugin.json
git commit -m "chore: update plugin description for CLI-based version"
```
