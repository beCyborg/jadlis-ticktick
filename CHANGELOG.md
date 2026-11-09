# Changelog — jadlis-ticktick

Формат: [Keep a Changelog](https://keepachangelog.com/ru/1.1.0/), версии — [SemVer](https://semver.org/lang/ru/).

## [Unreleased]

## [1.1.0] — 2026-11-09 — Короткие задачи / Short tasks

### Для человека
- Задачи в TickTick теперь короткие: название до пяти слов без эмодзи и хэштегов, описание до двух строк.
- Адрес, время и контакт скилл пишет полностью, чтобы задачу можно было выполнить, ничего больше не открывая.
- Напоминание ставится, только когда у задачи есть срок или шаг делает сам человек.

### For agents
- Added: `skills/ticktick/SKILL.md` § «Writing a task» — title ≤5 words, no emoji/`#`/list names/dates; `content` ≤2 lines (gist + next step); place, time and contact in full inside those lines; reminder only with a due time or a human step; long context stays in its note, the task carries the path. A profile may tighten, not loosen.

## [1.0.1] — 2026-11-03

### Fixed
- Removed `privacyPolicyUrl` from plugin.json: Claude Code 2.1.263 (the hub CI pin) rejects the unknown field under `--strict`.

## [1.0.0] — 2026-11-03 — Первый выпуск / First release

### Для человека

- Скилл `/jadlis-ticktick:ticktick` ведёт задачи, привычки и фокус в TickTick через официальный MCP TickTick: короткие команды, ответ в 1–3 строки.
- Плагин сам подключает сервер `https://mcp.ticktick.com/`; вход — через OAuth TickTick при первом вызове.
- Скилл знает ловушки API: две конвенции дат у задач «на весь день», фиктивный id Inbox, перевыпуск id пунктов чеклиста, чек-ины привычек.
- Часовой пояс скилл берёт из настроек TickTick, а не задаёт сам.
- Личный профиль по желанию: в какой список класть какие задачи, метки, чеклисты, смена квартала. Без профиля задачи идут в Inbox.

### For agents

- Added: `skills/ticktick/SKILL.md` — Step 0 (tool prefix detection for `mcp__plugin_jadlis-ticktick_ticktick__*` / `mcp__ticktick__*`, profile via `${user_config.PROFILE_PATH}`, timezone via `get_user_preference.timeZone` with system fallback), intent routing table, API pitfalls grouped by area, compact reply format, ≤10 tool calls.
- Added: `.mcp.json` — server `ticktick`, `{"type": "http", "url": "https://mcp.ticktick.com/"}`.
- Added: `userConfig.PROFILE_PATH` (string, optional); `assets/profile.example.md` with fictional lists and a generic quarter-rollover section.
- Added: README pair (RU/EN), `.gitlab-ci.yml` including the hub CI, `.gitignore`.
