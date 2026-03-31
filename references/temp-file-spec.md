# Temp File Specification

JSON report format for subagent → main context data passing.

## Directory

`_temp/` in vault root. Added to vault excluded areas and `.gitignore`.

## Naming Convention

`{skill}-{task}-{timestamp}.json`

- `skill`: audit, dedup, cluster, inbox
- `task`: descriptive task name (e.g., `cli-scan`, `frontmatter`, `content`, `names-notes`, `tags-batch1`)
- `timestamp`: `YYYY-MM-DDTHH-MM` (local time, no colons for Windows compatibility)

Examples:
- `audit-cli-scan-2026-03-30T14-22.json`
- `dedup-names-notes-2026-03-30T14-22.json`
- `inbox-analysis-batch1-2026-03-30T14-22.json`

## JSON Schema

```json
{
  "skill": "string (audit|dedup|cluster|inbox)",
  "task": "string (descriptive task name)",
  "timestamp": "string (ISO 8601)",
  "scope": ["string (folder paths scanned)"],
  "stats": {
    "files_scanned": "number",
    "issues_found": "number"
  },
  "issues": [
    {
      "file": "string (relative path from vault root)",
      "category": "string (for grouping in batch confirmation)",
      "severity": "high|medium|low",
      "suggested_fix": "string (concrete CLI command or action description)",
      "confidence": "number (0.0-1.0, especially important for dedup)",
      "details": "object (skill-specific extra data)"
    }
  ]
}
```

### Field Semantics


| Field           | Purpose                                        | Example                                                   |
| --------------- | ---------------------------------------------- | --------------------------------------------------------- |
| `category`      | Group issues for `balanced` trust level        | `missing_field`, `orphan_note`, `name_duplicate`          |
| `severity`      | Prioritize display order (high → medium → low) | `high` for missing required frontmatter                   |
| `suggested_fix` | Concrete action, NOT problem description       | `property:set file="X" name="created" value="2024-11-15"` |
| `confidence`    | Subagent certainty (0.0-1.0)                   | `0.95` for exact filename match in dedup                  |
| `details`       | Skill-specific data                            | `{"name_similarity": 0.8, "tag_overlap": 0.6}` for dedup  |


### Dedup-Specific Extensions

Dedup issues use `details` for multi-stage scoring:

```json
{
  "file": "notes/Note A.md",
  "category": "duplicate_pair",
  "severity": "high",
  "suggested_fix": "merge into notes/Note B.md",
  "confidence": 0.87,
  "details": {
    "pair_file": "notes/Note B.md",
    "name_similarity": 0.8,
    "tag_overlap": 0.7,
    "content_similarity": 0.95,
    "confidence_formula": "name × 0.3 + tags × 0.2 + content × 0.5"
  }
}
```

### Inbox-Specific Extensions

Inbox issues represent analysis results, not problems:

```json
{
  "file": "_inbox/ML transformer attention.md",
  "category": "inbox_analysis",
  "severity": "medium",
  "suggested_fix": "move to notes/Механизм внимания в трансформерах.md",
  "confidence": 0.9,
  "details": {
    "proposed_name": "notes/Механизм внимания в трансформерах.md",
    "type": "note",
    "tags": ["ml/transformers", "ml/attention"],
    "status": "draft",
    "created": "2026-03-28",
    "related_notes": ["notes/Архитектура трансформера.md"],
    "conflict": null,
    "unclear": false,
    "unclear_reason": null
  }
}
```

## Lifecycle


| Event               | Action                                                     |
| ------------------- | ---------------------------------------------------------- |
| Created by          | Subagent via `Write` tool                                  |
| Read by             | Main context via `Read` tool (by sections for large files) |
| Cleanup prompt      | Skill asks user after apply phase                          |
| Auto-cleanup        | In `auto` trust level after successful apply               |
| Stale detection     | Skill warns about files >24h old on next run               |
| Conflict prevention | Timestamp in filename; re-runs create new files            |
