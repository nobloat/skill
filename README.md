# nobloat/skill

The nobloat software taste as a [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills): small, self-hosted, stdlib-only Go tools. Distilled from [workq](https://github.com/nobloat/workq), [blog](https://github.com/nobloat/blog), [tinyviper](https://github.com/nobloat/tinyviper), [poll](https://github.com/nobloat/poll) and [up](https://github.com/nobloat/up).

All content is in [SKILL.md](SKILL.md).

## Install

Clone once, symlink into each agent's skills directory, update with `git pull`.

```sh
git clone https://github.com/nobloat/skill ~/git/nobloat/skill
```

| Agent | User-wide | Per project |
|---|---|---|
| Claude Code | `ln -s ~/git/nobloat/skill ~/.claude/skills/nobloat` | `.claude/skills/nobloat` |
| Codex | `ln -s ~/git/nobloat/skill ~/.agents/skills/nobloat` | `.agents/skills/nobloat` |
| Pi | reads `~/.claude/skills` and `~/.agents/skills`, either link above works | `.agents/skills/nobloat` |
| Other [Agent Skills](https://agentskills.io) agents | `~/.agents/skills/nobloat` | `.agents/skills/nobloat` |

Create the parent directory first if missing (`mkdir -p ~/.agents/skills`). Per project, prefer a git submodule over a copy: `git submodule add https://github.com/nobloat/skill .agents/skills/nobloat`.

Agents without skill support: add `Follow ~/git/nobloat/skill/SKILL.md.` to their `AGENTS.md`.

Use: the agent loads the skill when its description matches (starting, building or reviewing a nobloat project); force it with `/nobloat` (Claude Code), `$nobloat` (Codex) or `/skill:nobloat` (Pi).
