# obsidian-brain

A Claude Code plugin for managing your Obsidian vault via CLI. Provides a set of skills for processing, organizing, and maintaining your knowledge base without leaving the terminal.

## Requirements

- Obsidian >= 1.12.4 with CLI enabled
- **Obsidian installer must be up to date** — if you see a warning like `Your Obsidian installer is out of date`, download the latest installer from [obsidian.md/download](https://obsidian.md/download) and reinstall (vault and settings are preserved). An outdated installer can cause CLI write commands to fail silently.
- Obsidian must be running when using skills
- Claude Code

### Enable Obsidian CLI

1. Open Obsidian → Settings → General → Command line interface
2. Toggle **Register CLI** on
3. Verify: `obsidian version` in your terminal

### Windows: CLI wrapper (important)

On Windows, `obsidian` may resolve to `Obsidian.exe` (GUI version) instead of `Obsidian.com` (console version). The GUI exe breaks on subcommands with colon + parameters (`property:set`, `daily:append`, etc.) with exit code 127.

**Fix:** create a wrapper script at `~/bin/obsidian` (ensure `~/bin` is in your PATH):

```bash
mkdir -p ~/bin
cat > ~/bin/obsidian << 'EOF'
#!/usr/bin/env bash
exec '/c/Users/<username>/AppData/Local/Programs/Obsidian/Obsidian.com' "$@"
EOF
chmod +x ~/bin/obsidian
```

A wrapper script works in both interactive and non-interactive shells (unlike aliases).

## Installation

### As a third-party plugin (recommended)

```bash
# Add the marketplace from GitHub
claude plugin marketplace add YaroslavDrozdovskiy/obsidian-brain

# Install the plugin
claude plugin install obsidian-brain@obsidian-brain
```

### From local path

```bash
claude plugin marketplace add /path/to/obsidian-brain
claude plugin install obsidian-brain@obsidian-brain
```

## Skills

### `/process-inbox` — Process incoming notes

Processes raw notes from `_inbox/`: determines topic, sets frontmatter and tags, finds related notes, adds wikilinks, moves to target folder with automatic link updates.

**Typical flow:**
1. Drop notes into `_inbox/` (after a meeting, lecture, or reading)
2. Run `/process-inbox`
3. For each note: review the suggested name, tags, folder, and found connections
4. Confirm or adjust the plan
5. Notes are organized, linked with wikilinks, and indexed by Obsidian

---

### `/decompose` — Decompose monolithic notes

Splits large notes (>10KB) into atomic notes and creates a Map of Content (MOC).

**Typical flow:**
1. A 5000-word book summary sits in `literature/`
2. Run `/decompose` (or the plugin will suggest candidates)
3. Review the plan: which sections become separate notes, which stay in the MOC
4. Confirm
5. Result: 8 atomic notes in `notes/` + MOC in `_moc/` with links

---

### `/dedup` — Find and merge duplicates

Three-stage duplicate search (by names → tags → content) with interactive merging.

**Typical flow:**
1. Run `/dedup`
2. Plugin finds pairs with varying confidence (high / medium / low)
3. For each pair: view both files, choose the primary one
4. Duplicate is archived to `_archive/merged/`, all links updated automatically

---

### `/cluster` — Cluster and graph analysis

Two modes:

**`analyze` — graph diagnostics:**
1. Run `/cluster` → choose `analyze`
2. Get a report: hubs (most-linked notes), orphans, dead ends, broken links, weak clusters
3. For each problem — a recommendation (create links, add to MOC, archive)

**`group` — group into MOC/summary:**
1. Run `/cluster` → choose `group`
2. Plugin finds clusters of related notes (by tags, links, content)
3. For each cluster: suggests MOC (different aspects) or a summary note (common thesis)
4. Confirm composition and type → MOC in `_moc/` or summary in `notes/`

---

### `/audit` — Vault quality audit

Two modes:

**`quick` — healthcheck in 30 seconds:**
1. Run `/audit` → choose `quick`
2. Three quick checks: orphans, broken links, singleton tags
3. Brief report with numbers

**`full` — full audit with fixes:**
1. Run `/audit` → choose `full`
2. Checks across 5 categories: frontmatter, tags, links, placement, content
3. Summary report with issue counts per category
4. Choose: fix all / only category X / skip
5. Interactive mode: for each issue — suggested fix → confirmation

---

### `/rename-tags` — Tag taxonomy refactoring

Interactive refactoring of vault tag taxonomy.

**Typical flow:**
1. Run `/rename-tags`
2. See the full tag tree with note counts
3. Plugin identifies: singleton tags, duplicates with different casing, flat tags that could become hierarchical
4. Approve the rename plan (you can edit it)
5. Tags renamed across the entire vault in one command

---

### `/improve` — Improve note content

Interactive content improvement: structure, formatting, tables, rewrite, summary, wikilinks.

**Typical flow:**
1. Run `/improve` and specify a note
2. See the operations menu with relevance scores (high / medium / low)
3. Select the desired operations
4. For each operation: preview "before → after" → confirm or skip
5. Final report of applied changes

**Available operations:**

| # | Operation | What it does |
|---|-----------|--------------|
| 1 | Structure | Headings, section reorganization, empty sections |
| 2 | Format | Bold terms, lists, callouts |
| 3 | Tabulate | Enumerations → markdown tables |
| 4 | Rewrite | Better phrasing, remove filler |
| 5 | Summarize | TL;DR with key theses |
| 6 | Wikify | Add `[[wikilinks]]` to existing notes |

## Configuration

Before first use, configure the reference files in `references/` to match your vault:

```bash
# Copy the vault config template and edit it
cp references/vault-config.example.md references/vault-config.md
# Then open references/vault-config.md and set your vault path and folder structure
```

Reference files:

- `references/vault-config.md` — **your vault path and folder structure** (gitignored, copy from `.example.md`)
- `references/frontmatter-spec.md` — frontmatter templates and field rules
- `references/cli-operations.md` — Obsidian CLI commands reference
- `references/interaction-patterns.md` — interaction patterns (language, confirmations, reports)

## Vault structure expected

```
_inbox/        ← Incoming notes (processed by /process-inbox)
notes/         ← Atomic notes
literature/    ← Book/course/article summaries
projects/      ← Project notes
_moc/          ← Maps of Content (navigation files)
daily/         ← Daily notes
_archive/      ← Archive (inactive + merged duplicates)
_templates/    ← Obsidian templates
_assets/       ← Attachments (images, PDFs)
```

## Safety

- Files are never deleted — only archived
- Moving and renaming only happen after user confirmation
- Wikilinks are updated automatically via CLI when moving files
- All content is preserved — nothing is lost
- Git commits after each operation with specific file names

## License

MIT
