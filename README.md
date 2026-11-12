# Claude Code session restore kit (Herdr)

Bring every long-running Claude Code session back after a reboot with one command, read a session's full history including everything from before compaction, and a small agent roster to go with it.

Built and tested on macOS with zsh, herdr-core 0.1.20 and Claude Code 2.1.322. Tested by stopping and restarting Herdr's core, which leaves every tab as a plain shell in its folder, the same state a reboot leaves. Re-test after you update either tool.

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
- A resumed session keeps the model it was using. `claude-restore --default-model` brings every session back on the model in your settings.
- Quit a session with `/exit` when you are done with it, so it does not come back after the next reboot.
- The scripts call `herdr-core` at `~/.local/bin/herdr-core`. If yours lives elsewhere, change the `HC=` line near the top of `claude-restore` and `claude-resume-hook`. A wrong path fails quietly: `claude-restore -l` stays empty and `claude-restore` says Herdr isn't running.
- A shell that is waiting for typed input (a half-finished command) looks idle to `claude-restore`.
- Herdr's terminal version documents a built-in option that resumes agents after a restore. Check whether your version already does this before installing anything.

## License

MIT. See `LICENSE`.
