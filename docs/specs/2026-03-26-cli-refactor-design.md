# obsidian-brain: Рефакторинг на Obsidian CLI 1.12.4

**Дата:** 2026-03-26
**Подход:** CLI-wrapper — замена файловых операций на CLI-команды с сохранением структуры плагина

## Контекст

Obsidian 1.12.4 добавил полноценный CLI — remote-control для запущенного Obsidian. Текущий плагин работает с vault через прямые файловые операции (Glob, Grep, Read/Write, bash mv), что создаёт проблемы:
- `bash mv` не обновляет wikilinks
- Grep по vault медленнее индексированного поиска Obsidian
- Прямая запись YAML frontmatter рискует сломать формат
- Нет доступа к графу связей без ручного парсинга `[[...]]`

CLI решает все эти проблемы, работая через API Obsidian.

## Ограничения

- Obsidian должен быть запущен для работы CLI
- `eval code="..."` используется ограниченно — только где стандартные команды не справляются
- Прямой доступ к файлам сохраняется для: массового чтения (>20 файлов), сложных regex, word count, работы с `_archive/_templates`

## Структура плагина после рефакторинга

```
obsidian-brain/
├── .claude-plugin/
│   └── plugin.json
├── README.md                          # Обновлённый, на русском
├── references/
│   ├── vault-config.md                # Без изменений
│   ├── frontmatter-spec.md            # Без изменений
│   ├── interaction-patterns.md        # Обновлён: daily-log паттерн
│   └── cli-operations.md              # НОВЫЙ: заменяет scan-patterns.md
├── skills/
│   ├── process-inbox/SKILL.md         # Переписан на CLI
│   ├── decompose/SKILL.md             # Переписан на CLI
│   ├── dedup/SKILL.md                 # Переписан на CLI
│   ├── cluster/SKILL.md               # Расширен: analyze + group
│   ├── audit/SKILL.md                 # Расширен: quick + full
│   └── rename-tags/SKILL.md           # НОВЫЙ
```

6 скиллов вместо изначально планируемых 9 (схлопнуты пересечения):
- audit + healthcheck → audit (quick/full режимы)
- cluster + graph-analysis → cluster (analyze/group режимы)
- daily-log → паттерн в interaction-patterns.md

## cli-operations.md — справочник CLI-операций

Заменяет `scan-patterns.md`. Единый маппинг «задача → CLI-команда»:

| Задача | Было (файловые операции) | Стало (CLI) |
|---|---|---|
| Список файлов в папке | `Glob "vault/notes/**/*.md"` | `obsidian files path="notes/"` |
| Чтение заметки | `Read файл` | `obsidian read file="name"` или `Read` для больших файлов |
| Создание заметки | `Write файл` | `obsidian create name="path" content="..." silent` |
| Добавление в конец | `Edit файл` | `obsidian append file="name" content="..."` |
| Добавление после frontmatter | `Edit файл` | `obsidian prepend file="name" content="..."` |
| Перемещение файла | `bash mv` + ручное обновление ссылок | `obsidian move file="name" to="folder/"` |
| Удаление (архивация) | `bash mv` в _archive | `obsidian move file="name" to="_archive/"` |
| Поиск по содержимому | `Grep pattern` | `obsidian search query="text" limit=20` |
| Контекстный поиск | `Grep -C 3` | `obsidian search:context query="text" limit=10` |
| Frontmatter — чтение | `Read` + парсинг YAML | `obsidian properties file="name"` |
| Frontmatter — запись | `Edit` YAML-блока | `obsidian property:set file="name" name="key" value="val"` |
| Frontmatter — удаление поля | `Edit` | `obsidian property:remove file="name" name="key"` |
| Все теги vault | `Grep` + агрегация | `obsidian tags` |
| Файлы по тегу | `Grep "#tag"` | `obsidian tag tag="#tagname"` |
| Переименование тегов | `Grep` + `Edit` по всему vault | `obsidian tags:rename old="old" new="new"` |
| Исходящие ссылки | `Grep "\[\[" в файле` | `obsidian links file="name"` |
| Входящие ссылки | `Grep "\[\[filename\]\]"` по vault | `obsidian backlinks file="name"` |
| Битые ссылки | `Grep` + проверка существования | `obsidian unresolved` |
| Заметки-сироты | `Grep` + пересечение множеств | `obsidian orphans` |
| Daily Note | Ручной путь к файлу | `obsidian daily` / `daily:read` / `daily:append` |

### Когда оставить прямой доступ к файлам

- Массовое чтение содержимого (>20 файлов) — `Read` быстрее серии `obsidian read`
- Regex-поиск со сложными паттернами — `Grep` мощнее `obsidian search`
- Word count, размер файлов — `Bash wc/stat` (CLI не предоставляет)
- Операции с `_archive`, `_templates` — вне зоны CLI-индекса

### eval — ограниченное использование

Только для данных, недоступных через стандартные команды:

```bash
# Полный граф связей из metadataCache (cluster:analyze)
obsidian eval code="JSON.stringify(Object.fromEntries(Object.entries(app.metadataCache.resolvedLinks).map(([k,v])=>[k,Object.keys(v)])))"
```

## Рефакторинг существующих скиллов

### process-inbox

| Этап | Было | Стало |
|---|---|---|
| Сканирование _inbox | `Glob "_inbox/**/*.md"` | `obsidian files path="_inbox/"` |
| Чтение заметки | `Read` | `obsidian read file="name"` |
| Поиск связанных | `Grep` по ключевым словам | `obsidian search query="keyword" limit=10` |
| Frontmatter | `Edit` YAML-блока | `obsidian property:set` для каждого поля |
| See also | `Edit` конца файла | `obsidian append file="name" content="\n## See also\n..."` |
| Перемещение | `bash mv` + ручной grep/replace | `obsidian move file="name" to="notes/"` |

### decompose

| Этап | Было | Стало |
|---|---|---|
| Поиск кандидатов | `Glob` + `bash stat` | `obsidian files path="literature/"` + `bash stat` |
| Создание атомарных заметок | `Write` | `obsidian create name="notes/topic" content="..." silent` |
| Frontmatter | Встроен в content | `obsidian property:set` после create |
| Конвертация в MOC | `Edit` содержимого + frontmatter | `obsidian property:set name="type" value="moc"` + `Edit` тела |
| Перемещение MOC | `bash mv` | `obsidian move file="name" to="_moc/"` |

### dedup

| Этап | Было | Стало |
|---|---|---|
| Поиск по именам | `Glob` + токенизация | `obsidian files` + токенизация |
| Поиск по тегам | `Grep` по frontmatter | `obsidian tag tag="#name"` |
| Поиск по содержимому | `Grep` фраз | `obsidian search:context query="phrase" limit=5` |
| Обновление ссылок | `Grep "\[\[old\]\]"` + `Edit` по vault | `obsidian move` |
| Архивация дубля | `bash mv` в _archive/merged | `obsidian move file="name" to="_archive/merged/"` |

### cluster (расширен)

**Режим `analyze`:**
- `obsidian orphans` — изолированные заметки
- `obsidian unresolved` — битые ссылки
- `obsidian backlinks file="name"` — входящие связи для каждой заметки
- eval (ограниченно) — полный граф, только если >100 заметок

Выводит: хабы (топ-10), сироты, тупики, битые ссылки, слабые кластеры.
Не создаёт/перемещает файлы — только диагностика и рекомендации.

**Режим `group`:** (рефакторинг существующего)
- `obsidian tag tag="#name"` вместо Grep для поиска по тегам
- `obsidian links/backlinks` вместо regex для поиска по ссылкам
- `obsidian search` вместо Grep для поиска по контенту
- `obsidian create` вместо Write для создания MOC
- `obsidian append` вместо Edit для добавления backlinks

### audit (расширен)

**Режим `quick`:**
```bash
obsidian orphans           # сироты
obsidian unresolved        # битые ссылки
obsidian tags              # singleton-теги
```
Три команды, результат за секунды. Краткий отчёт без интерактивных фиксов.

**Режим `full`:** (рефакторинг существующего)
| Категория | Было | Стало |
|---|---|---|
| A. Frontmatter | `Read` + парсинг YAML | `obsidian properties file="name"` |
| B. Теги | `Grep` + агрегация | `obsidian tags` + `obsidian tag` |
| C. Ссылки | `Grep "\[\["` + проверки | `obsidian orphans` + `obsidian unresolved` + `obsidian backlinks` |
| D. Размещение | `Glob` + чтение type | `obsidian files` + `obsidian properties` |
| E. Контент | `Read` + `bash wc` | `obsidian read` + `bash wc` |
| Фиксы | `Edit` | `obsidian property:set`, `obsidian move`, `obsidian append` |

## Новые скиллы

### rename-tags

Интерактивный рефакторинг таксономии тегов.

1. `obsidian tags` — полное дерево тегов с количеством
2. Таблица: тег, кол-во заметок, уровень вложенности
3. Автоматическое выявление проблем: singleton-теги, дубли casing, плоские теги-кандидаты на иерархию
4. План переименований: старый → новый, затронуто N заметок
5. Подтверждение пользователя
6. `obsidian tags:rename old="old" new="new"` для каждого
7. Коммит: `refactor: rename tags`

Не удаляет теги, не трогает `English/` и `_archive/`.

### daily-log (паттерн в interaction-patterns.md)

После завершения любого скилла:
```
"Записать результат в Daily Note? (y/n)"
```
Если да:
```bash
obsidian daily:append content="\n### [Skill name] — HH:MM\n- Обработано: N заметок\n- Изменено: M\n- [краткий список действий]"
```

## README.md

Обновлённый README на русском с:
- Требованиями и установкой
- Описанием каждого скилла с типичным сценарием использования
- Структурой vault
- Правилами безопасности (не удаляет файлы, подтверждение перед действиями, авто-обновление wikilinks)

## Безопасность

Сохраняются все существующие правила:
- NEVER delete files — только архивация через `obsidian move to="_archive/"`
- NEVER rename без подтверждения пользователя
- NEVER proceed без explicit approval
- Preserve all original content
- `git add <specific files>` вместо `git add -A`
