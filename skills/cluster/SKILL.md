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
