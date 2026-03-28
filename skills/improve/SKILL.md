---
name: improve
description: Improve and structure note content — add headings, tables, formatting, rewrite, summarize, wikify. Use when user asks to improve a note, structure content, format note, or says /improve.
---

# Improve Note Content

Interactively improve the content of a single Obsidian note. The user picks operations from a menu, the skill warns about order conflicts, and executes one by one with preview and confirmation.

## Setup

1. Read `references/vault-config.md` from this plugin directory for vault path and folder rules.
2. Read `references/frontmatter-spec.md` for frontmatter templates and field rules.
3. Read `references/cli-operations.md` for CLI commands reference.
4. Read `references/interaction-patterns.md` for interaction and safety rules.

## Process

### 1. Identify Target Note

Follow the Hybrid Entry Point pattern from `references/cli-operations.md`:

**User provided a note name/path** → use it directly:

```bash
obsidian read file="note name"
```

**User provided nothing** → ask for the note name. Do NOT scan the entire vault — this skill works on a single note.

Save the original content in memory for potential rollback.

### 2. Analyze Note

Read the note and evaluate each of the 6 operations. For each operation, determine relevance: высокая / средняя / низкая / не нужна.

**Criteria for relevance assessment:**

| Operation | Высокая | Средняя | Низкая | Не нужна |
|-----------|---------|---------|--------|----------|
| Structure | Есть текст без заголовков, пустые секции, текст после таблиц вне секции | Заголовки есть, но уровни непоследовательны | Структура хорошая, мелкие улучшения | Заметка уже хорошо структурирована |
| Format | Много терминов без выделения, нет списков где они уместны | Частичное форматирование | Форматирование в целом ок | Уже хорошо отформатирована |
| Tabulate | Есть перечисления/сравнения, явно просящиеся в таблицу | Есть кандидаты, но спорные | Один мелкий кандидат | Нет подходящих данных |
| Rewrite | Длинные запутанные предложения, вода, повторы | Есть 1-2 фрагмента для улучшения | Текст в целом ясный | Текст чёткий и лаконичный |
| Summarize | Заметка >1500 слов, нет TL;DR | >800 слов, может быть полезно | <800 слов | <300 слов или уже есть TL;DR |
| Wikify | >3 термина, совпадающих с заметками в vault | 1-3 совпадения | Термины есть, но совпадений мало | Все связи уже проставлены |

**Для оценки Wikify** — найти кандидатов на wikilinks:

```bash
obsidian search query="термин из заметки" limit=5
```

Проверять по ключевым терминам заметки. Учитывать точные совпадения с именами файлов и очевидные синонимы/аббревиатуры.

**Проверка размера** — если заметка >10KB или содержит >2 тем верхнего уровня (заголовки H1):

```
⚠️ Заметка содержит несколько крупных тем. Рекомендуется сначала /decompose, потом /improve по отдельным заметкам.
Продолжить /improve как есть? (да/нет)
```

### 3. Show Menu

Present the menu to the user (in Russian):

```
## Improve: [Название заметки]

| # | Операция | Релевантность | Комментарий |
|---|----------|--------------|-------------|
| 1 | Structure | ⬆ высокая | 3 пустые секции, текст вне заголовков |
| 2 | Format | ➡ средняя | частичное выделение терминов |
| 3 | Tabulate | ➡ средняя | 1 кандидат на таблицу |
| 4 | Rewrite | ⬆ высокая | 2 длинных фрагмента |
| 5 | Summarize | ⬇ низкая | <800 слов |
| 6 | Wikify | ➡ средняя | 4 кандидата |

Операции с релевантностью «не нужна» не показываются.

Выбери номера через запятую (например 1,3,6):
```

WAIT for user response.

### 4. Check Order and Warn About Conflicts

The safe execution order (from structure to content):

1. Structure
2. Format
3. Tabulate
4. Rewrite
5. Summarize
6. Wikify

**Conflict rules:**

| Выбор пользователя | Конфликт | Рекомендация |
|--------------------|----------|-------------|
| Summarize перед Rewrite | Summarize сожмёт текст, Rewrite будет нечего улучшать | Сначала Rewrite, потом Summarize |
| Rewrite перед Structure | Rewrite может улучшить текст, который Structure потом перенесёт | Сначала Structure, потом Rewrite |
| Wikify перед Rewrite/Summarize | Rewrite/Summarize могут переписать текст с wikilinks | Wikify в конце |
| Format перед Structure | Format выделит текст, который Structure перераспределит | Сначала Structure, потом Format |
| Tabulate перед Structure | Structure может перенести перечисление в другую секцию | Сначала Structure, потом Tabulate |

If any conflict detected, show warning:

```
⚠️ Рекомендуемый порядок отличается от выбранного:

Ваш порядок: Rewrite → Structure → Wikify
Рекомендуемый: Structure → Rewrite → Wikify

Причина: Rewrite может улучшить текст, который Structure потом перенесёт в другую секцию.

Использовать рекомендуемый порядок? (да / нет, оставить мой)
```

WAIT for user response.

### 5. Execute Operations One by One

For each selected operation, in the confirmed order:

#### 5.0 Before first operation

Re-read the note to have the current state:

```bash
obsidian read file="note name"
```

#### 5.1 Show Preview

For each operation, identify ALL changes and show them as «было → стало» blocks:

```
### Операция 1/N: Structure

**Изменение 1:** Текст после таблицы «Цели» (строки 122-123) → вынести в подзаголовок

**Было:**
> в 31 цели решение принимается по скору, так как не было сформировано FA
> по 35 и 34 принимается решение из лимита по КП.

**Стало:**
> ### Принятие решений по целям
> - **Цель 31** — решение по скору (FA не было сформировано)
> - **Цели 35, 34** — решение из лимита по КП

**Изменение 2:** Пустая секция «## Общее»

Удалить / Оставить / Заполнить из контекста?

---
Применить? (✅ да / ❌ нет / ✏️ скорректировать)
```

WAIT for user response.

#### 5.2 Apply or Skip

- **✅ да** — apply changes using `Read` + `Edit` tools. Do NOT use `obsidian append/prepend` for inline edits — use `Edit` tool for precise replacements.
- **❌ нет** — skip this operation, move to next
- **✏️ скорректировать** — user provides corrections, re-show preview with adjustments

After applying, re-read the note for the next operation:

```bash
obsidian read file="note name"
```

#### 5.3 Operation-Specific Guidelines

**Structure:**
- Identify text blocks without headings → propose heading
- Find empty sections → ask: удалить / оставить / заполнить
- Fix heading hierarchy (H1 → H2 → H3, no skipped levels)
- Move orphaned text (after tables, between sections) under appropriate headings
- Do NOT change H1 (note title)

**Format:**
- Bold key terms on first use: `**PD-модель**`
- Convert inline enumerations to bullet lists
- Add `> [!note]` callouts for important definitions
- Do NOT re-bold already bold terms

**Tabulate:**
- Convert comparison lists to markdown tables
- Convert "X — Y" enumeration patterns to 2-column tables
- Only tabulate where it genuinely improves readability
- Do NOT convert lists that are better as lists

**Rewrite:**
- Shorten long sentences (>30 words)
- Remove filler words and repetition
- Improve clarity without changing meaning
- Preserve technical terms exactly
- Do NOT change meaning or add information

**Summarize:**
- Add TL;DR block after frontmatter, before first heading:
  ```
  > **TL;DR:** [3-5 bullet points with key takeaways]
  ```
- Do NOT touch tables — summarize only text sections
- Do NOT compress lists of items (they're already concise)

**Wikify:**
- Search vault for each candidate term:
  ```bash
  obsidian search query="term" limit=5
  ```
- Only link if a note with matching name exists
- Include obvious synonyms/abbreviations (e.g., PD → PD-модель)
- Link only first occurrence of each term in the note
- Do NOT link terms inside headings
- Do NOT link terms already linked
- Add new links to `## See also` section if not already present

### 6. Rollback Option

After all operations (or if user requests mid-way):

```
Откатить все изменения этой сессии? (да/нет)
```

If yes — restore the saved original content using `Edit` tool (replace entire body, preserving frontmatter).

### 7. Report + Daily Log

Follow `references/interaction-patterns.md`:

```
## Результат: improve

Заметка: [название]
Применено: N операций
- Structure — [что сделано]
- Rewrite — [что сделано]
Пропущено: M операций
- Wikify — отменено пользователем

Записать результат в Daily Note? (y/n)
```

Commit message (if obsidian-git not active):
```bash
cd "{vault_path}"
git add "path/to/modified/note.md"
git commit -m "refactor: improve content of [note name]"
```

## Important

- NEVER modify frontmatter (YAML block) — only note body
- NEVER delete existing wikilinks — only add new ones
- NEVER delete TODO checkboxes (`- [ ]`, `- [x]`)
- NEVER auto-apply changes — always show preview and wait for confirmation
- NEVER compress or modify table data in Summarize
- NEVER change the language of the note — detect and work in the same language
- NEVER proceed without explicit user approval at each step
- Always keep a copy of original content for rollback
- Respond in Russian
