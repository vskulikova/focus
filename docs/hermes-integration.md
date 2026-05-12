# Hermes integration spec

Дата: 2026-05-11
Контекст: Hermes (Telegram-buddy на oldmac) утром даёт nutrition-цель, вечером принимает отчёт. Все данные питания живут в этом репо (`focus`), life-planner на него ссылается.

## Чтение цели

Утиль `scripts/read_current_phase.py` (живёт в этом репо или в `life-planner/scripts/hermes/`):
- читает `data/sushka-2026.md`
- находит текущую неделю: ищет строку в таблице прогноза, где сегодняшняя дата входит в диапазон «Даты»
- находит секцию `### Неделя N` с теми же датами, парсит `**Задание:**`
- возвращает dict `{kcal: 1900, protein_g_min: 120, phase: "Неделя 2 (11-17 мая)"}`
- если дата вне диапазона плана — возвращает `None`, Hermes молча пропускает nutrition-блок

## Утренний бриф

В Hermes `gather_morning.py`: дописать строку.

Пример:
```
🥗 Питание: 1900 ккал · ≥100 г белка · Неделя 1
```

## Приём отчёта

Telegram-хендлер. Триггеры:
- сообщение содержит число + «ккал» / «kcal» / «калор»
- прислан image (скриншот)
- команда `/nutrition <kcal> <protein>`

Два пути парсинга:
- **manual** — regex по тексту: «1850 ккал, 105 белка» → `{kcal: 1850, protein_g: 105}`
- **screenshot** — vision-модель (Claude Haiku 4.5, vision из коробки) → extract `{kcal, protein_g}` из скрина FatSecret

После парсинга:
1. Найти в `data/logs/YYYY-MM.jsonl` строку с `kind="nutrition_log"` и `date=сегодня`.
2. Если есть — переписать файл с заменой строки; если нет — append.
3. Ответить пользователю: «Записала: 1850 ккал, 105 г белка. Цель: 1900/≥100. ✅»
4. Если факт не дотягивает по белку или сильно выбивается по ккал — короткий нейтральный комментарий, без морализаторства.

Атомарность записи — через temp-файл + `os.replace`.

## Вечерний nudge

Если за сегодня нет `nutrition_log` к 21:30 (локальный TZ oldmac) — Hermes пишет: «Что по еде сегодня? Цифры или скрин из FatSecret». Один раз за день.

## Расширение log_event в Hermes

Текущая `scripts/hermes/log_event.py` в life-planner жёстко требует `kind/trigger/user_state/...`. Для nutrition — добавить отдельную функцию `log_nutrition(date, kcal, protein_g, source, raw=None)`, которая пишет в **focus** репо, а не в life-planner. Путь — через `FOCUS_REPO_PATH` env (default `~/life-planner-data/focus` на oldmac или `~/0_Projects/focus` локально).

## Git-sync

oldmac уже делает `git pull`/`git push` в life-planner на каждой рутине. Аналогично нужно для focus:
- pull в начале рутины (чтобы подтянуть свежий `current-phase.md`, отредактированный с основного мака)
- push в конце (чтобы факт залогированный с oldmac попал на основной мак до `/weekly`)

## Будущая интеграция FatSecret API

Не делать сейчас. Запланировать через 2-4 недели, если ритуал зайдёт.
- OAuth 2.0, регистрация developer-аккаунта на platform.fatsecret.com
- Эндпоинт `food_entries_get.v2` за дату → автоподтягивание
- Источник `fatsecret_api` в `nutrition_log`

## Открытые вопросы

- Vision: подтвердить, что у Hermes уже есть Anthropic API ключ в `.env` (или какой провайдер используется для текущих диалогов).
- Часовой пояс: подтвердить, что oldmac в `Europe/Madrid`, чтобы 23:50 не уезжало в следующий день.
