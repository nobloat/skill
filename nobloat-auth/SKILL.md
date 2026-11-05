---
name: nobloat-auth
description: How nobloat services authenticate and authorize: single-user basic auth, or nobloat/auth (the nobloat IdP) in front via caddy forward_auth with the Remote-* header contract, groups as the only source of access, API keys and service users issued by auth. Use when adding auth to a nobloat app, wiring an app or integration behind nobloat/auth, writing Caddyfiles, group conventions, local auth stacks, or reviewing access control.
---

# nobloat-auth

Read with the `nobloat` skill. Learned while moving bill, track and tax from Authelia plus app-owned API keys to nobloat/auth as the single source of access.

## Two modes, nothing else

Every app supports both; both may be on at once.

1. **Single user:** `auth.user` + `auth.password` in the config (together or not at all), HTTP basic auth, always admin. For one person running the tool alone, behind plain caddy `reverse_proxy` or none. No IdP needed.
2. **Behind nobloat/auth:** caddy `forward_auth` sends every request to auth; the app trusts the `Remote-*` headers only from `trusted_proxies`. `role.*` maps groups to roles.

Startup refuses a config with neither mode (except `-dev`, a fixed local admin) and `auth.user` without `auth.password`.

**`resolve` rule:** a request with an `Authorization` header never falls through to `Remote-*` headers. `Basic` with matching credentials (constant-time compare of both) is the single user; anything else is 401 (`WWW-Authenticate: Basic realm="][x app"` when basic auth is configured). Test: basic + forged `Remote-User` → 401.

The app has no login page, no user management and no API keys of its own. Only `/metrics` keeps its own `metrics_key`.

## Header contract

`Remote-User` (stable key, a lower-case e-mail), `Remote-Name`, `Remote-Email`, `Remote-Groups` (comma-separated, sorted). Fixed names; another IdP is mapped in caddy, never via configurable header names. `logout_url` and `account_url` point to auth; hide them for the basic-auth user.

## Groups are the only source of access

auth decides who may do what; apps store **no** access lists (no `groups`, `users` or `see_all` fields on objects, no groups on accounts). Each app maps groups:

- `access-<app>`: admission, checked by auth (the app file's `groups`).
- `<app>-viewer|editor|admin`: the role, mapped with `role.<role> = <group>` in the app config. Highest wins.
- Resource groups, per app, built from ids: `bill-<company>`, `tax-<company>`, `track-<company>` (sees the company's projects), `track-<company>-owner` (owns all of them, creates new ones), `track-<project>` (records time), `track-<project>-owner` (edits the project and its tags, sees everyone's entries). Admins see everything.

Consequences:
- Ids that appear in group names are chosen on creation, readable `[a-z0-9-]+`, never random.
- A shared business key (the company id) is the same in every app; only one app holds the company's data (bill), the others just carry the id.
- Self-service creation needs a scope-level owner group (`track-<company>-owner`), otherwise nobody can see the new object until an admin edits auth.
- Roles in auth bundle groups ("Space oliver" = `access-*`, the app roles and the company groups); onboarding a colleague is a role plus a user.

## API keys and service users (in auth, not in the apps)

- Users create keys on auth's account page, one forward-auth app per key; admins create them for any user. `ak_<base64url(username)>.<random>`, only the sha256 of the random part is stored, shown once.
- caddy sends bearer requests through `forward_auth` too and strips `Authorization` before the app. auth checks **on every request** that the user exists, is not disabled, owns the key, the key is for this app, and the user's current groups still allow the app; then it answers with the same `Remote-*` headers as for a browser. Losing a group, being disabled or revoking the key closes the key on the next request.
- Integrations are **service users** in auth: no password, only keys, groups that say which companies/projects the integration reaches (`bill@service.example` with track groups, `tax@service.example` with bill groups). IaC seeds them with `keys = <app>:<sha256>` in the user file (read once, like the password seed).
- Integrations call the other app **through caddy** (its public URL), not the container directly, or auth never sees the request. In a local stack, give the containers caddy's root CA (`SSL_CERT_FILE`, copied readable by a one-shot container) and network aliases for the hostnames.
- CLIs send `Authorization: Bearer $APP_KEY` (an auth key) or, for a single-user instance, `APP_USER`/`APP_PASSWORD` as basic auth.

## Caddyfile

```caddyfile
(auth) {
	forward_auth auth:8742 {
		uri /api/authz/forward-auth
		copy_headers Remote-User Remote-Groups Remote-Email Remote-Name
	}
}
(app) {
	handle /metrics {
		reverse_proxy {args[0]}          # the app's own metrics_key
	}
	handle {
		import auth                      # browsers and auth's API keys
		reverse_proxy {args[0]} {
			header_up -Authorization     # the key was auth's; the app trusts Remote-*
		}
	}
}
track.example.com {
	import app track:8740
}
```

Public paths (status pages, shared links with their own password) get their own `handle` before `import app`. Single-user mode needs only `reverse_proxy`.

## Forward auth vs. OIDC

nobloat apps use forward auth: no login code, groups current on every request, keys through the same path. OIDC is only for third-party apps that cannot read headers (Forgejo, Grafana); their group freshness ends at the session (no back-channel logout, no refresh tokens in auth).

## Tests every app keeps

- No headers, no `Authorization` → 401. Headers from an untrusted peer → 401.
- Basic ok → admin; wrong password → 401; basic without `auth.user` configured → 401; basic + forged headers → 401.
- Group mapping: role per group, resource visibility per group (404 for invisible objects, `422 unknown_<kind>` when creating into them).
- In auth: key with lost group → 403, disabled user → 401, revoked → 401, key for another app → 403.
