---
name: cluster
description: Group related notes into MOC or summary note, or analyze vault graph structure. Use when user asks to cluster notes, group notes, create MOC, analyze graph, find orphans/hubs, or says /cluster.
---

# Cluster & Graph Analysis

Two modes:
- **analyze** — parallel sharded graph analysis: hubs, orphans, dead-ends, weak clusters
- **group** — find related notes and organize into MOC or summary

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

Выберите режим:

- **analyze** — диагностика графа с параллельным анализом (хабы, сироты, тупики, слабые связи)
- **group** — группировка заметок в MOC или обобщающую заметку

```

---

## Mode: analyze

Generate timestamp: `ts=$(date +%Y-%m-%dT%H-%M)`

### Phase 1: Collect Raw Graph (single subagent, CLI)

Dispatch 1 subagent with `subagent_type: "vault-worker"`:

Prompt:
```

Vault path: {vault_path}
Task: Extract the full vault link graph.

1. Run: obsidian eval code="JSON.stringify(Object.fromEntries(Object.entries(app.metadataCache.resolvedLinks).map(([k,v])=>[k,Object.keys(v)])))"
2. Parse the result into a graph structure.
3. Also run `obsidian orphans` and `obsidian unresolved` for orphan/broken link data.
4. Write the graph as JSON with structure:

{
  "skill": "cluster",
  "task": "graph-extract",
  "timestamp": "...",
  "scope": ["vault-wide"],
  "stats": { "nodes": N, "edges": M, "orphans": K, "unresolved": L },
  "graph": { "file_path": ["linked_file1", "linked_file2", ...], ... },
  "orphans": ["file1.md", ...],
  "unresolved": ["[[Missing Link]]", ...]
}

Write to: {vault_path}/_temp/cluster-graph-{ts}.json

```

### Phase 2: Analyze Shards (parallel)

After graph is collected, determine shard sizes:
- Count files in `notes/` — if >600, split into sub-shards of ~600
- `literature/`, `projects/`, `_moc/` — combine into one shard

Dispatch N subagents (typically 3-4) with `subagent_type: "vault-worker"`:

Prompt per shard:
```

Vault path: {vault_path}
Task: Analyze graph shard for {shard_description}.

Read the full graph from: {vault_path}/_temp/cluster-graph-{ts}.json

Analyze ONLY nodes matching your shard ({shard_files}), but use the FULL graph for edge counting (cross-shard edges matter).

For each node in your shard, calculate:

- in_degree (backlinks count from full graph)
- out_degree (outgoing links count)
- is_orphan (in_degree == 0 AND out_degree == 0)
- is_dead_end (in_degree > 0 AND out_degree == 0)
- is_hub (in_degree >= 5)

Find clusters: groups of 2-3 nodes connected only to each other (weak/isolated clusters).

Report:

- Top hubs in this shard (by in_degree)
- Orphans in this shard
- Dead-ends in this shard
- Weak clusters found
- Average connectivity stats

Categories: hub, orphan_node, dead_end, weak_cluster, bridge_candidate
Severity: orphan_node=medium, dead_end=low, weak_cluster=medium, hub=low (informational)

Write to: {vault_path}/_temp/cluster-analysis-{shard_name}-{ts}.json

```

### Phase 3: Merge and Present

1. Read all shard analysis JSON files.
2. Merge into unified report:

```

## Анализ графа vault

Всего заметок: N | Связей: M

### Хабы (топ-10 по входящим ссылкам)


| #   | Заметка | Входящие | Исходящие |
| --- | ------- | -------- | --------- |


### Сироты (нет связей)

- file1.md, file2.md, ...

### Тупики (входящие есть, исходящих нет)

- file3.md (← 5 входящих)

### Битые ссылки

- [[Missing]] ← из: file1.md, file2.md

### Слабые кластеры

- {note4, note5} — изолированная пара

```

3. Recommendations for orphans and weak clusters.
4. Cleanup temp files, daily log.

---

## Mode: group

Generate timestamp: `ts=$(date +%Y-%m-%dT%H-%M)`

### Phase 1: Find Cluster Candidates (parallel)

Dispatch 2 subagents with `subagent_type: "vault-worker"`:

**Subagent 1: Tag-based clusters**

Prompt:
```

Vault path: {vault_path}
Task: Find tag-based note clusters.

1. Run `obsidian tags` to get all tags with counts.
2. For each tag with >= 3 notes: run `obsidian tag tag="#tagname"`.
3. Check if a MOC exists in _moc/ for this tag/topic.
4. If no MOC exists and tag has >= 3 notes → candidate cluster.

Report each candidate with: tag, note list, note count, MOC exists (bool).
Category: tag_cluster
Write to: {vault_path}/_temp/cluster-candidates-tags-{ts}.json

```

**Subagent 2: Content-based clusters**

Prompt:
```

Vault path: {vault_path}
Task: Find content-based note clusters.

1. Use Grep to find frequently co-occurring terms across notes/, literature/, projects/.
2. For groups of notes sharing >= 3 specific terms: `obsidian search query="term" limit=20`.
3. If >= 3 notes share multiple terms and no existing MOC covers the topic → candidate cluster.

Report each candidate with: key terms, note list, note count, overlap strength.
Category: content_cluster
Write to: {vault_path}/_temp/cluster-candidates-content-{ts}.json

```

### Phase 2: Present and Confirm

1. Merge candidates, deduplicate (same notes may appear in tag and content clusters).
2. Present each cluster with recommendation (MOC vs summary):
   - **MOC** — notes describe different aspects of one topic, no single thesis
   - **Summary** — content reducible to one common thesis

3. WAIT FOR USER APPROVAL.

### Phase 3: Create (sequential, main context)

Ask trust level per `references/trust-levels.md` (default: `balanced`).

**Create MOC (if chosen):**

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

> If note already has `## See also`, use `Read` + `Edit` to add link to existing section.

**Create Summary (if chosen):** Same flow, target folder `notes/` instead of `_moc/`.

### Phase 4: Cleanup + Report

Cleanup temp files, report, daily log, commit per interaction-patterns.

## Important

- NEVER delete notes
- Always wait for user approval
- Preserve all original content
- Respond in Russian
