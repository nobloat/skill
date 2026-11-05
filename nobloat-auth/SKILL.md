---
name: nobloat-auth
description: How nobloat services authenticate and authorize: single-user basic auth, or nobloat/auth (the nobloat IdP) in front via caddy forward_auth with the Remote-* header contract, groups as the only source of access, API keys and service users issued by auth. Use when adding auth to a nobloat app, wiring an app or integration behind nobloat/auth, writing Caddyfiles, group conventions, local auth stacks, or reviewing access control.
---

# nobloat-auth

This skill lives in the nobloat/auth repository, next to the code it describes: read `../auth/skill/SKILL.md` (checked out next to this repo) or https://github.com/nobloat/auth/blob/main/skill/SKILL.md.

To install it directly, link the auth repo's skill directory instead of this one: `ln -s ~/git/nobloat/auth/skill ~/.claude/skills/nobloat-auth`.
