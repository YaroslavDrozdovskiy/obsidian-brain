# Frontmatter Specification

## Format: YAML (mandatory for all notes)

## Fields

| Field | Required | Values | Description |
|-------|----------|--------|-------------|
| `type` | Always | `note`, `literature`, `project`, `moc`, `daily` | Note type |
| `tags` | Always (except daily) | Hierarchical: `ML/metrics` | Topic tags |
| `status` | Always (except moc, daily) | `draft`, `active`, `archive` | Note status |
| `source` | Required for `literature`, optional for `note` | Free text | Source reference |
| `created` | Always | `YYYY-MM-DD` | Creation date |

## Templates by Type

### Note
```yaml
---
tags:
  - topic/subtopic
type: note
source: "Source name"
status: active
created: YYYY-MM-DD
---
```

### Literature
```yaml
---
tags:
  - topic/subtopic
type: literature
source: "Book/Course/Article — Author"
status: active
created: YYYY-MM-DD
---
```

### Project
```yaml
---
tags:
  - topic/subtopic
type: project
status: active
created: YYYY-MM-DD
---
```

### MOC
```yaml
---
tags:
  - topic
type: moc
created: YYYY-MM-DD
---
```

### Daily
```yaml
---
type: daily
created: YYYY-MM-DD
---
```

## Links

- NOT in frontmatter. Use wikilinks in body + `## See also` section at end.
- Obsidian tracks all `[[wikilinks]]` automatically.

## Atomicity

An atomic note covers one self-contained idea/concept, understandable without external context (typically 100-2000 words).
