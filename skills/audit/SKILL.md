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

Use `Grep` for bulk frontmatter checks (NOT per-file CLI calls):

```bash
# 1. Get all frontmatter fields at once
Grep pattern="^(type|tags|status|created|source):" path="{vault_path}/notes/" output_mode=content
Grep pattern="^(type|tags|status|created|source):" path="{vault_path}/literature/" output_mode=content
Grep pattern="^(type|tags|status|created|source):" path="{vault_path}/projects/" output_mode=content
Grep pattern="^(type|tags|status|created|source):" path="{vault_path}/_moc/" output_mode=content

# 2. Find files without frontmatter
Grep pattern="^---$" path="{vault_path}/notes/" output_mode=files_with_matches
# Compare with Glob to find files missing frontmatter
```

Parse results to check:
- Missing frontmatter entirely (file not in Grep results for `^---$`)
- Missing required fields (type, tags, status, created) per `references/frontmatter-spec.md`
- Invalid values (type ∉ {note|literature|project|moc|daily}, status ∉ {draft|active|archive}, created not YYYY-MM-DD)
- Template mismatch by type (e.g., literature without source)

> Use `obsidian properties` only for individual files during interactive fix phase, not for bulk scanning.

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

Reuse frontmatter data from step A (already collected via Grep). For each file, compare `type:` value against actual folder path:
- `notes/` → type should be `note`
- `literature/` → type should be `literature`
- `projects/` → type should be `project` or `moc` (project-local MOCs are OK)
- `_moc/` → type should be `moc`

> Do NOT run `obsidian files` or `obsidian properties` per file here — use the Grep results from step A.

**E. Content:**

Use batch word count and Grep for structure checks (NOT per-file CLI reads):

```bash
# Word count for all scan area files at once
bash: find "{vault_path}/notes" "{vault_path}/literature" "{vault_path}/projects" "{vault_path}/_moc" -name "*.md" -exec wc -w {} +

# Find files without ## headings
Grep pattern="^## " path="{vault_path}/notes/" output_mode=files_with_matches
# Compare with Glob to find files missing headings

# Find files with ## See also
Grep pattern="^## See also" path="{vault_path}/notes/" output_mode=files_with_matches
```

Check:
- Too short (<50 words, except moc)
- Too long (>3000 words) — candidate for /decompose
- Empty sections
- No ## headings

> Use `obsidian read` only during interactive fix phase for individual files, not for bulk scanning.

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
