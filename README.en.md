[Русский](README.md) · English

# TickTick from Claude Code: one short command and the task is in place

The `/jadlis-ticktick:ticktick` skill manages tasks, habits and focus in TickTick through the
official TickTick MCP (the tool server run by TickTick itself). You type briefly: "what's on
today", "add buy milk tomorrow", "check in meditation". The skill picks the right tool and
replies in 1–3 lines. It understands both Russian and English commands.

## What it does

- Shows today's tasks and what was done over a period.
- Adds one task or a list of tasks. It expands dates like "tomorrow", "Mon", "12.08" itself.
- Completes a task or closes it as "won't do".
- Moves a task to another list or another date.
- Checks in habits and shows check-ins.
- Shows focus time.
- Finds tasks by title.

The skill knows TickTick API pitfalls that the docs don't mention. For example, all-day tasks
store dates in two different ways, so the skill compares dates only in your timezone. It reads
the timezone from your TickTick settings and never hard-codes one.

## Installing

You need Claude Code and a TickTick account.

```
claude plugin marketplace add https://gitlab.com/beCyborg/jadlis-hub.git
claude plugin install jadlis-ticktick@jadlis
claude plugin list
```

The first command adds the marketplace, the second installs the plugin. To remove:
`claude plugin uninstall jadlis-ticktick@jadlis`.

**First sign-in.** The plugin connects the server `https://mcp.ticktick.com/`. On first use
Claude Code opens the TickTick sign-in page (OAuth: you sign in on the TickTick site, no password
goes into Claude Code). If no window opens, type `/mcp`, pick `plugin:jadlis-ticktick:ticktick`
and sign in.

**If you already have a TickTick server.** If you added your own TickTick MCP server before (for
example with `claude mcp add`), the plugin adds a second, identical one. The skill works with
either. Disable the extra one in `/mcp`.

**Personal profile — optional.** A profile describes which list gets which task, which tags to
use and how to switch lists at the start of a quarter. Without a profile new tasks go to the
Inbox, and the skill asks when something is unclear. Create a file, for example
`~/ticktick-profile.md`, and set its path — in a session via `/config` or with:

```
echo '{"PROFILE_PATH":"~/ticktick-profile.md"}' | claude plugin configure jadlis-ticktick@jadlis --values-stdin
```

Sample: [assets/profile.example.md](assets/profile.example.md) (in Russian, the format is the
same in any language).

## Using it

Type in Claude Code:

```
/jadlis-ticktick:ticktick what's on today
```

Or just say it in your own words:

- "add call the bank tomorrow at 10"
- "add: milk; bread; batteries" — three tasks in one call
- "done report"
- "move bike repair to Saturday"
- "check in water 3"
- "how much did I focus this week"
- "what did I finish yesterday"

Sample reply:

```
4 tasks today:
⬜ 🔴 Send the report — Work, 11:00
⬜ ⚪ Buy milk — Inbox
```

## Limits

- Works only through the official TickTick MCP. If a tool is missing on the server, the skill
  says so.
- Calendar is not covered: the skill does not read or create calendar events.
- "Complete every task in a list" runs only after your explicit confirmation: it cannot be
  undone.
- The API does not return task photos or attachments. The skill points to a workaround.
- The plugin is free and needs no API keys: sign-in goes through TickTick OAuth.
- The plugin stores nothing itself: tasks go straight between Claude Code and the TickTick server.
  TickTick privacy policy: https://ticktick.com/about/privacy
