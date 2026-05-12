# Focus — Data Model

Контракт данных для трекинга питания. Используется Hermes (oldmac) и life-planner (`/weekly`).

## Источники правды

| Данные | Source of truth | Кто пишет | Кто читает |
|---|---|---|---|
| Методология сушки | `data/structured/approach.md` | Claude Code | Справочник |
| План сушки + трекинг | `data/sushka-2026.md` | Lera + Claude Code | Hermes (morning brief), life-planner (`/weekly`) |
| Факт по дням | `data/logs/YYYY-MM.jsonl`, event `nutrition_log` | Hermes (вечерний приём отчёта) | life-planner (`/weekly` агрегация) |
| Сырьё переписок | `data/raw/` | Manual | Claude Code (анализ) |
| Исторический прогресс | `data/structured/personal-progress.md` | Manual | Архив |

## Nutrition Log Contract

Файл: `data/logs/YYYY-MM.jsonl` (append-only, одна строка JSON на запись).

Схема события:

```json
{
  "timestamp": "2026-05-11T21:30:00+02:00",
  "kind": "nutrition_log",
  "date": "2026-05-11",
  "kcal": 1850,
  "protein_g": 105,
  "source": "screenshot",
  "raw": "optional — исходный текст или имя файла скриншота"
}
```

`source` ∈ {`manual`, `screenshot`, `fatsecret_api`}.

Правила:
- одна запись на дату; повторная отправка перезаписывает предыдущую (find-by-date → rewrite, не дублировать);
- если день пропущен — записи нет; weekly агрегация считает его «не залогирован», а не «0 ккал»;
- timezone — локальный TZ устройства, которое пишет (oldmac).

## Sushka Plan Contract

Файл: `data/sushka-2026.md` — единственный source of truth для плана.

Структура:
- Таблица прогноза — строки с диапазонами дат, целевыми ккал и прогнозом веса.
- Секции `### Неделя N (даты)` — содержат `**Задание:**` и `**Отчёт:**`.

Парсер для Hermes читает текущую неделю по дате:
1. Найти строку в таблице, где сегодняшняя дата попадает в диапазон `Даты`.
2. Найти секцию `### Неделя N` с соответствующими датами.
3. Извлечь `**Задание:**` — получить ккал, белок и другие цели.
4. Вернуть dict `{kcal, protein_g_min, phase, week_dates}`.

## Связь с life-planner

`life-planner/planner_cli.py gather-weekly` читает из focus:
- цели текущей недели из `data/sushka-2026.md` (парсер по дате)
- факты за неделю из `data/logs/*.jsonl` (по полю `kind=nutrition_log` и `date` в диапазоне недели)
- агрегаты: `days_in_kcal_range`, `days_protein_ok`, `avg_kcal`, `avg_protein`, `days_missing`

Путь до focus в planner_cli — через переменную окружения `FOCUS_REPO_PATH` (по умолчанию `~/0_Projects/focus`).
