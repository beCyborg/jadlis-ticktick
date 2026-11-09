---
name: ticktick
description: >-
  Manage TickTick tasks, habits and focus sessions via the official TickTick MCP: parses short
  inputs, routes to the right tool, replies compactly; optional personal profile for list routing.
  Triggers: ticktick, tasks, today, add task, complete task, habits, checkin, focus.
  RU triggers: задачи, что на сегодня, добавь задачу, заверши задачу, готово, чекин, привычки,
  фокус, перенеси задачу, что сделано за. Do NOT use for: calendar events (not covered);
  other task managers; screen time or time tracking.
argument-hint: <short command — "что на сегодня", "добавь купить молоко завтра", "чекин медитация">
---

# /ticktick — tasks, habits and focus in TickTick

Drive TickTick through the official MCP (`https://mcp.ticktick.com/`). Parse the user's short
input, pick the tool, reply compactly.

**User request:** `$ARGUMENTS`

## Step 0 — tools, profile, timezone

1. **Tools.** The plugin's server shows up as `mcp__plugin_jadlis-ticktick_ticktick__<tool>`; a
   user-level server of the same service shows up as `mcp__ticktick__<tool>`. Use whichever is
   connected (`ToolSearch "ticktick"`). Below, tools are named by the short `<tool>` part. The
   inventory changes on the service side (about 50 tools in 2026: tasks, projects, habits,
   focus, tags, comments, columns, countdowns, project groups) — `ToolSearch` and
   `get_tool_details` are the only reliable source. A tool you expect is missing → say so, never
   invent one. Neither server connected → tell the user to run `/mcp` and sign in to TickTick.
2. **Profile.** Personal profile path: `${user_config.PROFILE_PATH}`. If that value is non-empty
   and is not the literal `${user_config...}` placeholder, Read the file once per session. Its
   rules (which list gets which task, tags, checklist containers, quarter rollover) override the
   defaults below. A profile never lifts the confirmation rules in "API pitfalls". No profile, or
   the file is missing → defaults, and mention a missing file in one line.
3. **Timezone.** Before any date logic call `get_user_preference` once per session and take
   `timeZone` (an IANA name). It fails → use the system timezone (`date +%Z`) and say so in one
   line. Never assume a fixed zone.

## Intent routing

| Input | Tool |
|---|---|
| «что на сегодня», today | `list_undone_tasks_by_date` |
| «добавь …» / a list split by `;` or new lines | `create_task` / `batch_add_tasks` |
| «готово …», «выполнил», «заверши …», ✓ | `search_task` → `complete_task` |
| «не буду …», «отмени задачу …» | `search_task` → `update_task {status: -1}` |
| «чекин …», «привычки за сегодня» | `list_habits` → `upsert_habit_checkins` |
| «фокус», «сколько фокусился» | `get_focuses_by_time` |
| «найди», «где задача» | `search_task` (title) / `search` / `filter_tasks` |
| «проекты», «списки» | `list_projects` |
| «что сделано за …» | `list_completed_tasks_by_date` |
| «перенеси … в …» (another list) | `search_task` → `move_task` |
| «перенеси … на …» (another date), «обнови …» | `search_task` → `update_task` |

Intent unclear → ask in one sentence. A list or habit named in words → first `list_projects` /
`list_habits` for the id, fuzzy-match by prefix; several matches → ask.

**Where a new task goes.** The profile has routing rules → follow them. Otherwise: the user named
a list → that list; no list named → Inbox. If a named list matches nothing, ask instead of
creating one.

**Defaults:** date — today (unless the user names a date or says «без даты»); priority — 0;
check-in value — 1. Expand relative dates («завтра»,
«через 3 дня», «пн», «12.08») to ISO yourself, in the user's timezone.

**Tags:** never create tags or add them to tasks without an explicit request. Delete or rename
(`delete_tag` / `rename_tag`) only on an explicit request.

**"Subtasks in one task"** usually means ONE task with `kind: CHECKLIST` and `items[]`, not child
tasks via `parentId` — child tasks show up in lists as separate tasks. Use `parentId` only when
the user asks for real child tasks.

## Writing a task

A task is read on a phone between other things, so it stays short. This applies to every
`create_task` / `batch_add_tasks` / `update_task` you write, including tasks an agent sets for a
person; a profile may make it stricter, not looser.

- **Title — up to five words**, a verb first when it is an action («Записать машину на диагностику»).
  No emoji, no `#`, no list or tag names, no dates or prices in the title.
- **Description (`content`) — at most two lines**: the gist and the next step. No history, no
  reasoning, no copied letters; a link or a file path instead of the text it points to.
- **Place, time and contact go in full** whenever the task has them: the full address, the
  date and time, the phone or handle of the person — inside those two lines, never shortened:
  the user acts on the task without opening anything else.
- **Reminder** — only when the task has a due time or the next step is the user's own (go
  somewhere, sign, pay, call). No reminder on tasks an agent does itself.
- Too much context for two lines → keep it where it already lives (a note, a session's plan) and
  put the path in the description.

## API pitfalls (checked; month of the check in brackets)

> [!danger] `complete_tasks_in_project` completes EVERY task in the project
> Irreversible. Call it only after the user's explicit confirmation of that exact project.

Dates and timezones
- **All-day tasks use TWO date conventions in one list.** Tasks made in the app store day D as
  local midnight in UTC (e.g. `(D-1)T22:00Z` for UTC+2); tasks created through MCP with an
  explicit time may sit at `D T00:00Z` or another hour. A naive compare with "today" shows false
  overdue items → compare ONLY by the local date in the user's `timeZone`, and convert
  `completedTime` the same way. (2026-06, 2026-09)
- **`batch_update_tasks` is a PATCH**: fields you don't pass (`items`, `content`) stay, but it
  does not move `startDate` — the task stretches into a multi-day bar. To move a date use
  `update_task` and set both `startDate` and `dueDate`. (2026-06)

Projects and the Inbox
- **The Inbox has a fake id.** `list_projects` returns `id: "inbox"`, but task calls need the real
  `inbox<userId>` — take it from the `projectId` of any Inbox task. (2026-07)
- **`projectName` is never returned** — only `projectId`; resolve names via `list_projects`. (2026-09)
- **`update_project {closed: true}` archives a list, and its open tasks stay inside the archive**
  — they vanish from working queries. Move or close open tasks BEFORE archiving a list. (2026-09)

Response shape
- **Responses are wrapped in `{"result": …}`** — read down to `result`. (2026-06)
- **Raw `tools/call` puts a list into SEVERAL `content[]` items** (one JSON object per line), so
  `content[0].text` silently cuts the list to its first item. Read
  `result.structuredContent.result`; use `content[]` only as a fallback, joining all items. (2026-09)

Search and dedup
- **`search_task` matches a substring of the title**, recurring and service tasks included — for
  dedup compare the list and the content too, not only the name. (2026-09)
- **`closed: true` works in filters; `move_task` keeps checklists.** (2026-06)

Checklists
- **`complete_checklist_item` re-issues the ids of ALL items** in the checklist — after the first
  call the other ids from an earlier read are invalid («Checklist item … does not belong to
  task»). Close items one by one, taking the next id from `items[]` in the previous response;
  never run parallel calls on one checklist. (2026-09)
- **Rewriting a checklist**: `update_task` with `kind: CHECKLIST` and the full `items[]`; item ids
  are re-issued. Titles up to ~350 characters are accepted without cuts. (2026-09)
- **Checklist items never appear in completed lists**: `list_completed_tasks_by_date` returns only
  fully closed tasks. For items done today read `list_undone_tasks_by_date` and `items[]`. (2026-07)
- **`parentId` via `update_task` works, but the response shows it as `null`.** Deleting a parent
  does NOT delete its children — delete them one by one. (2026-09)

Closing and status
- **Close a task that no longer matters with `update_task {status: -1}`** ("won't do"), not
  `complete_task` — it leaves the open lists and does not count as done. `completedTime` of a
  closed task can be corrected with the same `update_task`. (2026-09)
- **`#word` in `title`/`desc` on `create_task` becomes a task tag** (`#4806` → tag `4806`). Write
  ticket numbers without `#`; to remove — `update_task {tags: []}`. (2026-10)
- **`repeatFlag` works** — recurring tasks can be created through the API (RRULE string).

Habits
- **Check-in field is `stamp`** (not `checkinStamp`), a flat `yyyyMMdd`. In `get_habit_checkins`
  the `to_stamp` bound is **exclusive** — with `from == to` the result is empty and every habit
  reads falsely as "no data". (2026-06)
- **Check-in shape**: `result[] → {habitId, checkins: [{stamp, value, goal, status}]}`.
  **`status: 0` with `value > 0` means partial**, not "not done" (e.g. 3 of 6 glasses). (2026-06)
- **A habit's `encouragement` is silently cut at 64 characters** — check the field in the
  response after a write. (2026-07)
- **Habit `iconRes` degrades silently**: an unknown name turns into `txt_<first letter>`.
  Built-ins that exist: `habit_plan`, `habit_exercising`. An emoji is safer: `txt_🌅` — only a
  single-code-point emoji; `txt_🏋️` (base + U+FE0F) is stored as an empty icon, `txt_💪` works.
  (2026-09, 2026-10)

Kanban columns, comments, change detection
- **Columns are set via `update_task` with `columnId`** — the response returns the new
  `columnId`/`columnName`; there is no separate "move to column". `create_column` creates,
  `update_column` renames, **there is no column delete in the API**. (2026-09)
- **A task created without `columnId` first reads as `columnId: null`**; within a minute the server
  puts it into the column with the lowest `sortOrder`. Treat `null` as "first column". (2026-09)
- **`modifiedTime` does NOT change on `update_task`** (content edit, column move) and lags on
  writes; `add_comment` moves it. Detect changes by **`etag`** — it changes on every write. (2026-09)
- **`get_comment` returns no author name** — only `userId`, `title` (the text), `createdTime`,
  `modifiedTime`, `replyCommentId`. If you need to tell your own comments apart, prefix them.
  Text limit 1024 characters, plain text. (2026-09)

Attachments
- **The API does not return photos or attachments.** In `content` they appear as
  `![image](<attachmentId>/<name>)`; the Open API answers 404 on them. Workarounds (2026-09):
  1. macOS app: once the app has synced and shown the task, originals sit in
     `~/Library/Group Containers/75TY9UT8AY.com.TickTick.task.mac/Attachments/<userDirId>/<taskId>/`.
  2. A browser logged in to ticktick.com: from that page
     `fetch('https://ticktick.com/api/v1/attachment/<projectId>/<taskId>/<attachmentId>.<ext>', {credentials: 'include'})`.
     The same request to `api.ticktick.com` fails on CORS.

## Quarter rollover and other routines

Only when the profile describes them. Without a profile do not archive or create lists on your
own. A generic safe order for archiving lists: snapshot (`list_projects` +
`get_project_with_undone_tasks`) → decide each open task with the user (move / complete / won't
do / delete one by one) → `update_project {closed: true}` only when nothing is left open →
create the new lists.

## Reply format

1–3 lines per action, in the user's language. Priorities 🔴 high (5) / 🟡 medium (3) /
🔵 low (1) / ⚪ none (0); statuses ⬜ open, ✅ done, ✖ won't do. Lists by name, never by id. Dates
in the user's timezone. Do not repeat the request parameters or explain what you did. Budget —
up to 10 tool calls (usually 1–3).

Example:

```
Сегодня 4 задачи:
⬜ 🔴 Отправить отчёт — Работа, 11:00
⬜ ⚪ Купить молоко — Inbox
```
