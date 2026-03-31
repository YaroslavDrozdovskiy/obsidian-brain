# Trust Levels

User confirmation granularity for skill apply phases. Applies to all skills (both subagent-enabled and sequential).

## Levels

| Level | Behavior | Дефолт |
|-------|----------|--------|
| `cautious` | Показать каждый issue, confirm per-item | dedup |
| `balanced` | Группировать по category, confirm per-group | audit, cluster, process-inbox, rename-tags |
| `auto` | Применить весь план, показать итоговый отчёт | — (только по явному выбору пользователя) |

## Prompt Template

At skill start, after scan phase completes (or immediately for non-subagent skills):

```

Выберите уровень подтверждения:

- **cautious** — подтверждение каждого действия отдельно
- **balanced** — подтверждение группами по категориям (рекомендуется)
- **auto** — применить всё автоматически, показать отчёт

[balanced]

```

Default shown in brackets. User presses Enter to accept or types their choice.

## Constraints

These constraints OVERRIDE user selection:

| Constraint | Minimum level | Reason |
|------------|--------------|--------|
| `obsidian move` operations | `balanced` | Vault-wide wikilink side effects |
| dedup merge (content merging) | `balanced` | Irreversible content merge |
| First run of any skill | NOT `auto` | User must see what skill does at least once |

When constraint triggers, inform user:

```

⚠️ Операции перемещения (move) требуют минимум balanced.
Переключаю с auto → balanced для категории «Размещение файлов».

```

## Behavior by Level

### cautious

```

Проблема 1/87: Отсутствует поле created
Файл: notes/Обработка пропусков.md
Действие: property:set created="2024-11-15" (из git history)

Применить? (да / нет / применить все оставшиеся в этой категории)

```

The "применить все оставшиеся в этой категории" option allows upgrading to `balanced` mid-flow.

### balanced

```

Категория: Отсутствующие поля frontmatter (87 заметок)
Severity: high

Примеры (первые 5):

- notes/Обработка пропусков.md → set created: 2024-11-15
- notes/RMSLE metric.md → set status: draft
- ...

Действие: Проставить недостающие поля по frontmatter-spec
[Применить все 87] [Показать полный список] [Пропустить категорию]

```

### auto

No confirmation prompts. Apply all issues from the plan.

After completion, show full report with all applied changes.

Exception: categories with `balanced` constraint still show group confirmation.

## Integration with Skills

### Subagent-enabled skills (audit, dedup, cluster, process-inbox)

Trust level asked AFTER scan phase completes — user sees the summary first:

```

Сканирование завершено. Найдено 187 проблем в 5 категориях.

Выберите уровень подтверждения: [balanced]

```

### Sequential skills (improve, decompose, rename-tags)

Trust level asked at skill start:

- **improve**: `cautious` is built-in (preview each operation). Trust level adds `balanced` for "apply recommended order without individual confirmation" and `auto` for "apply all with default settings."
- **decompose**: `cautious` confirms each atomic note. `balanced` confirms the full plan. `auto` executes the plan.
- **rename-tags**: `cautious` confirms each rename. `balanced` confirms the full rename table. `auto` applies all renames.
