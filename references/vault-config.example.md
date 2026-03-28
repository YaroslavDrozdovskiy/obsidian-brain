# Vault Configuration

## Path

`/path/to/your/Obsidian/Vault`

> Copy this file to `vault-config.md` and set your vault path and folder structure.

## Folder Structure

| Folder | Purpose |
|--------|---------|
| `_inbox/` | Raw/temporary notes — entry point for processing |
| `notes/` | Atomic notes on any topic. Topic determined by tags. |
| `literature/` | Course/book/article summaries. Decomposed monoliths become MOCs here. |
| `projects/` | Project notes |
| `daily/` | Daily journal |
| `_moc/` | Map of Content navigation files |
| `_templates/` | Obsidian templates |
| `_assets/` | Attachments (images, PDFs, excalidraw) |
| `_archive/` | Inactive/outdated notes |

## File Naming

- Name by concept: `RMSLE metric.md`, `Note title.md`
- No date prefixes (date in `created` frontmatter field)
- MOC files keep numeric prefixes: `000 Home.md`

## Tag Hierarchy

Hierarchical tags with `/`:
- `topic/subtopic`, `topic/subtopic/detail`

## Name Conflicts

If target file already exists when moving/creating — ask user: merge, rename with suffix, or skip.
