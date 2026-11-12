# Claude Code cheat sheet: session restore, history and agents

## After a reboot or OS update

Open any terminal and run:

```
claude-restore
```

It reopens every recorded Claude session in its own Herdr tab, one second apart.

## Check before restoring

| Command | What it does |
|---|---|
| `claude-restore -n` | Shows what it would do, without doing it |
| `claude-restore -l` | Lists which session is recorded for each tab |
| `claude-restore --default-model` | Restores every session on the model in your settings |
| `claude-restore --model <id>` | Restores every session on a specific model |

Resumed sessions keep the model they were last using unless you pass one of the two model options.

## Herdr core commands (run from a plain terminal, not from inside a Claude session)

Restart Herdr's core. Your sessions keep running:

```
hc server restart
```

Simulate a reboot, closing every session, then restore them all:

```
hc server stop; hc server start; sleep 3; claude-restore
```

Only run the second one when no session is mid-task. Always run it yourself. When a restart is launched from inside a Claude session, every restored session is treated as that session's child and stops saving its conversation.

## If a tab shows `skip`

The line says why. When the session needs you, it ends with a command like `claude --resume <id>`: open the right tab and run it there. A line with no command ("already running claude", "an older record") needs nothing from you.

## Looking back at a conversation

| Command | What it does |
|---|---|
| `! claude-history` | From inside a Claude chat: opens that chat's full history |
| `claude-history t6` | Opens the history for Herdr terminal t6 |
| `claude-history -l` | Lists recent sessions with their tab names |
| `claude-history -s "some phrase"` | Searches every session for a phrase |

## Good habits

- `/rename Short name` in each new session makes it easy to spot later.
- Quit with `/exit` when you are done with a session. That stops `claude-restore` from bringing it back.
- One long-lived session per tab.

## After changing the main model

1. From a plain terminal, when no session is mid-task: `hc server stop; hc server start; sleep 3; claude-restore --default-model`
2. Type `/compact` in each tab. A conversation keeps the advisor it started with, and the advisor has to rank at or above the main model, so sessions moved to a stronger model have no advisor until they are compacted.

## Agent roster

| Role | Model | Effort | What it does |
|---|---|---|---|
| Main session | Fable 5.2 (1M context) | high | Plans, decides, does most of the work |
| Advisor | Fable 5.2 | high | Reviews the whole conversation before big plans, outward-facing actions and "done" |
| worker | Opus | high | Carries out decided plans; never sends or publishes unless told to |
| reviewer | Opus | xhigh | Checks code before it is called done; reports problems, never edits |
| researcher | Sonnet | high | Web and docs lookups, with sources |
| explorer | Sonnet | high | Read-only file and code search |

## Where things live

- Scripts: `~/.local/bin/claude-restore`, `claude-history`, `claude-resume-hook`
- Session records (one per Herdr terminal): `~/.local/state/claude-resume/`
- History pages: `~/.local/state/claude-history/`
- Hooks: two entries in `~/.claude/settings.json`
- Agents: `~/.claude/agents/`
- Rules for the advisor and agents: `~/.claude/CLAUDE.md`
