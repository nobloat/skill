# nobloat/skill

The nobloat software taste as a [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills): small, self-hosted, stdlib-only Go tools. Distilled from [workq](https://github.com/nobloat/workq), [blog](https://github.com/nobloat/blog), [tinyviper](https://github.com/nobloat/tinyviper), [poll](https://github.com/nobloat/poll) and [up](https://github.com/nobloat/up), plus bill and track for the system skill.

Three skills, one directory each:

- [nobloat](nobloat/SKILL.md): the taste for a single tool (stack, code, HTTP, auth, config, tests, UI, branding, plan-first workflow).
- [nobloat-auth](https://github.com/nobloat/auth/blob/main/skill/SKILL.md) (lives in nobloat/auth as `skill/`; `nobloat-auth/` here is a pointer): single-user basic auth or nobloat/auth in front via caddy forward auth, groups as the only source of access, API keys and service users from auth, Caddyfile, tests.
- [nobloat-system](nobloat-system/SKILL.md): building several cooperating services end to end (integrations, money and documents, own renderers, security baseline, metrics, local auth stack, verification, review loop, demo video). Used together with nobloat.

## Install

Clone once, symlink each skill directory into each agent's skills directory, update with `git pull`.

```sh
git clone https://github.com/nobloat/skill ~/git/nobloat/skill
```

| Agent | User-wide | Per project |
|---|---|---|
| Claude Code | `ln -s ~/git/nobloat/skill/nobloat ~/.claude/skills/nobloat` | `.claude/skills/nobloat` |
| Codex | `ln -s ~/git/nobloat/skill/nobloat ~/.agents/skills/nobloat` | `.agents/skills/nobloat` |
| Pi | reads `~/.claude/skills` and `~/.agents/skills`, either link above works | `.agents/skills/nobloat` |
| Other [Agent Skills](https://agentskills.io) agents | `~/.agents/skills/nobloat` | `.agents/skills/nobloat` |

Same for `nobloat-system`. For `nobloat-auth` link the auth repo's `skill/` directory (`ln -s ~/git/nobloat/auth/skill ~/.claude/skills/nobloat-auth`); linking `nobloat-auth/` from here also works and points there. Create the parent directory first if missing (`mkdir -p ~/.agents/skills`). Per project, prefer a git submodule over a copy: `git submodule add https://github.com/nobloat/skill .agents/nobloat-skill`, then symlink `.agents/skills/nobloat -> ../nobloat-skill/nobloat` (and `nobloat-system`).

Agents without skill support: add `Follow ~/git/nobloat/skill/nobloat/SKILL.md.` (and `nobloat-system/SKILL.md` for multi-service work) to their `AGENTS.md`.

Use: the agent loads a skill when its description matches (starting, building or reviewing a nobloat project); force it with `/nobloat` (Claude Code), `$nobloat` (Codex) or `/skill:nobloat` (Pi), likewise `nobloat-system`.
