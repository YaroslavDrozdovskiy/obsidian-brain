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
