# Focus — Data Model

Контракт данных для трекинга питания. Используется Hermes (oldmac) и life-planner (`/weekly`).

## Источники правды

| Данные | Source of truth | Кто пишет | Кто читает |
|---|---|---|---|
| Методология сушки | `data/structured/approach.md` | Claude Code | Все ритуалы |
| Текущая фаза (ккал, белок) | `data/current-phase.md` | Claude Code | Hermes (morning brief), life-planner (`/weekly`) |
| Факт по дням | `data/logs/YYYY-MM.jsonl`, event `nutrition_log` | Hermes (вечерний приём отчёта) | life-planner (`/weekly` агрегация) |
| Сырьё переписок | `data/raw/` | Manual | Claude Code (анализ) |
| Прогресс по фазам | `data/structured/personal-progress.md` | Manual + `/weekly` | Анализ |

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

## Phase File Contract

Файл: `data/current-phase.md`.

Обязательные секции:
- `## Сейчас` — таблица или список с полями `Фаза`, `С даты`, `Калории`, `Белок`, `Учёт`.
- `## История фаз` — таблица: Период / Ккал/день / Белок/день / Заметки.

Парсер Hermes извлекает «Сейчас» как dict для morning brief.

## Связь с life-planner

`life-planner/planner_cli.py gather-weekly` импортирует focus и достаёт:
- target из `current-phase.md`
- факты за неделю из `data/logs/*.jsonl` (по полю `kind=nutrition_log` и `date` в диапазоне недели)
- агрегаты: `days_in_kcal_range`, `days_protein_ok`, `avg_kcal`, `avg_protein`, `days_missing`

Путь до focus в planner_cli — через переменную окружения `FOCUS_REPO_PATH` (по умолчанию `~/0_Projects/focus`).
