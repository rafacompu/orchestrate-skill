# /orchestrate — a Claude Code skill

Turn a plan you just discussed with Claude Code into a **supervised, parallel
[Solo](https://soloterm.com/) fan-out** — scratchpad plan, todos with blockers,
worker agents running in parallel, idle-wake supervision, and a clean close-out.

You discuss a solution, type `/orchestrate`, and Claude drives the whole thing:

1. **Captures the plan** from the conversation into a Solo scratchpad (goal,
   background, lanes, and a file-ownership manifest).
2. **Builds the work graph** — one todo per lane, with blockers for cross-lane
   dependencies, tagged `parallel` or `lead`.
3. **Pauses at a spawn gate** so you approve the breakdown before any agents launch.
4. **Dispatches workers** with tiny prompts that point each one at its scratchpad
   key (no giant inline briefs — dodges Solo's input truncation).
5. **Supervises** via Solo's built-in idle-wake: harvests each finished worker,
   reconciles the plan, and dispatches the next unblocked wave. The lead works
   its own lane while workers run.
6. **Closes out** finished workers and gives you a final per-lane report.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- The [Solo](https://soloterm.com/) MCP server connected to Claude Code
  (this skill drives Solo's scratchpad, todo, agent, and timer tools)

## Install

Clone this repo, then make the skill available to Claude Code by linking (or
copying) the `orchestrate/` folder into your personal skills directory.

**Symlink (recommended — you get updates with `git pull`):**

```bash
git clone https://github.com/rafacompu/orchestrate-skill.git
ln -s "$(pwd)/orchestrate-skill/orchestrate" ~/.claude/skills/orchestrate
```

**Or copy it (simple, but no auto-updates):**

```bash
git clone https://github.com/rafacompu/orchestrate-skill.git
cp -r orchestrate-skill/orchestrate ~/.claude/skills/orchestrate
```

To install it for a single project instead of globally, put it under that
project's `.claude/skills/` directory rather than `~/.claude/skills/`.

## Usage

After you and Claude have discussed or planned a solution:

```
/orchestrate
```

Add steering inline when you want:

```
/orchestrate keep everything on the current branch, don't touch the migrations
```

The skill only runs when you type `/orchestrate` explicitly — a worker fan-out
never launches on its own.

## License

MIT — see [LICENSE](LICENSE).
