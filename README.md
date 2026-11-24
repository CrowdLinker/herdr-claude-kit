# Claude Code session restore kit (Herdr)

Bring every long-running Claude Code session back after a reboot with one command, read a session's full history including everything from before compaction, and a small agent roster to go with it.

Built and tested on macOS with zsh, herdr-core 0.1.20 to 0.1.22, and Claude Code 2.1.322 to 2.1.330. Tested through a real macOS reboot (seven sessions restored with one command) and by stopping and restarting Herdr's core, which leaves every tab in the same state. Re-test after you update either tool.

Moved to [Orca](https://orca.dev/)? It resumes sessions after a restart by itself. See [Using Orca instead of Herdr](#using-orca-instead-of-herdr).

## What is here

| Path | What it is |
|---|---|
| `bin/claude-resume-hook` | Records which Claude session runs in which Herdr terminal (SessionStart / SessionEnd hook) |
| `bin/claude-restore` | Resumes every recorded session in its own tab |
| `bin/claude-history` | Renders a session's whole transcript as a searchable page |
| `hooks-snippet.json` | The two hook entries to merge into `~/.claude/settings.json` |
| `agents/` | Four subagent definitions: worker, reviewer, researcher, explorer |
| `CLAUDE.md.example` | Rules for when the main session consults the advisor and which agent gets which task |
| `CHEATSHEET.md` | The commands on one page |

## Install

1. Get the kit and copy the three scripts into `~/.local/bin`. They need `jq` and Python 3, and `~/.local/bin` has to be on your PATH.

   ```
   git clone https://github.com/CrowdLinker/herdr-claude-kit.git && cd herdr-claude-kit
   mkdir -p ~/.local/bin && cp bin/claude-restore bin/claude-history bin/claude-resume-hook ~/.local/bin/
   chmod +x ~/.local/bin/claude-restore ~/.local/bin/claude-history ~/.local/bin/claude-resume-hook
   ```
2. Add the two hook entries in `hooks-snippet.json` to the `hooks` section of `~/.claude/settings.json`. Run this from the kit folder. It keeps the hooks you already have, saves a one-time backup, and only replaces your settings if the merge succeeded:

   ```
   mkdir -p ~/.claude
   [ -f ~/.claude/settings.json ] || echo '{}' > ~/.claude/settings.json
   cp -n ~/.claude/settings.json ~/.claude/settings.json.before-restore-kit
   jq -s '.[0] as $s | .[1].hooks as $h
     | def merged(ev): (($s.hooks[ev] // []) | map(select((.hooks // []) | any((.command // "") | test("claude-resume-hook")) | not))) + $h[ev];
       $s | .hooks.SessionStart = merged("SessionStart") | .hooks.SessionEnd = merged("SessionEnd")' \
     ~/.claude/settings.json hooks-snippet.json > ~/.claude/settings.json.new \
     && mv ~/.claude/settings.json.new ~/.claude/settings.json
   ```

   It is safe to run again: it replaces the kit's two entries instead of adding them twice.
3. Start or resume each long-lived session once, each in its own Herdr tab. The hook only records sessions in terminals that carry Herdr's tab label, so sessions started outside a Herdr tab are ignored. Then `claude-restore -l` should list every tab.
4. Test it when no session is mid-task, from a plain terminal outside Herdr: `hc server stop; hc server start; sleep 3; claude-restore`.
5. Optional: copy `agents/*.md` to `~/.claude/agents/` and adapt `CLAUDE.md.example` into your own `~/.claude/CLAUDE.md`.

## Things to know

- Never restart Herdr's core, or run `claude-restore`, from a shell that a Claude session started. Sessions restored that way are treated as child sessions and are left out of resume and history. Run it yourself from a plain terminal.
- `claude-restore` brings every session back on the model in your settings (on its own, a resumed session keeps the model it was using). `claude-restore --keep-model` keeps each session's old model.
- Quit a session with `/exit` when you are done with it, so it does not come back after the next reboot.
- The scripts call `herdr-core` at `~/.local/bin/herdr-core`. If yours lives elsewhere, change the `HC=` line near the top of `claude-restore` and `claude-resume-hook`. A wrong path fails quietly: `claude-restore -l` stays empty and `claude-restore` says Herdr isn't running.
- A shell that is waiting for typed input (a half-finished command) looks idle to `claude-restore`.
- Herdr's terminal version documents a built-in option that resumes agents after a restore. Check whether your version already does this before installing anything.

## Using Orca instead of Herdr

[Orca](https://orca.dev/) saves each agent's session. When it starts again after its terminals were stopped (as a reboot does), it reopens your workspaces and resumes each Claude Code session in its own tab: the tab you were looking at in each workspace right away, the others when you open them. You don't need `claude-restore` or the hook there. `claude-history` still works (`claude-history -l`, or `claude-history <id prefix>`), and so do the agents and `CLAUDE.md.example`.

To move a session from Herdr to Orca:

1. Note its session id and the folder it was started in: `claude-restore -l` lists the ids, and `jq -r .cwd ~/.local/state/claude-resume/<terminal>.json` shows the folder. Do this first, because step 2 deletes the record.
2. Wait until the session is idle at an empty prompt, then `/exit` it in Herdr. The hook deletes its record, so `claude-restore` won't bring it back.
3. In an Orca terminal, in the workspace you want: `cd <folder> && claude --resume <session id> --model <model>`, then send it one message. Orca resumes sessions later on the model of their last reply, so until the session replies on the new model it comes back on the old one.

Never run one session in Herdr and Orca at the same time.

Things to know (tested with Orca 1.4.272 and Claude Code 2.1.331 on macOS, through workspace Sleep, an app quit, and stopping Orca's background terminal process, which is what a reboot does; not yet through a real reboot):

- Orca installs its own agent status hooks in `~/.claude/settings.json`. Leave them on.
- Quitting the app (Cmd+Q) doesn't stop your sessions: Orca keeps terminals in a separate background process. To test the reboot path on macOS, when no session is mid-task: quit Orca, run `pkill -f "orca/daemon-host"` from a plain terminal (it ends every Orca terminal), and reopen Orca.
- Orca resumes a session without a model option, so it comes back on the model of its last reply. After you change models, run `/model` in each tab and send one message.
- A session resumes in the folder it was started in, even if that isn't the workspace's folder.

## License

MIT. See `LICENSE`.
