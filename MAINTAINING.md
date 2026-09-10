# Maintaining this plugin

## The default origin — now public, was the tailnet

`plugins/gesso/.claude-plugin/plugin.json` defaults `userConfig.origin` to:

```
https://app.getgesso.app
```

**Changed 10 September 2026**, from the tailnet address
`https://ubuntu-2404-noble-amd64-base.taildb1547.ts.net:7443`. That old default resolved
only from the Possible Worlds tailnet, so anyone else installing the plugin got a server
they could not reach — and the failure was quiet in the worst way: the plugin installs
cleanly, `claude mcp list` shows the server, and it simply never connects.

Note the host is **`app.getgesso.app`**, not bare `getgesso.app`. An earlier version of
this file named the apex domain; that was wrong. Verified 10 Sept 2026:
`app.getgesso.app/mcp` answers `401` (server present, authentication required) and
`/health` answers `200`, while the apex does not resolve at all. The apex is the marketing
site's eventual home, not the app's.

Self-hosters keep overriding this through the same option — that is the whole reason it is
a `userConfig` entry rather than a hardcoded URL.

> **Installed copies do not pick this up.** The default is read at install time, so an
> existing install keeps the old tailnet URL until the plugin is reinstalled (or the user
> sets `origin` explicitly). Anyone who installed before 10 Sept 2026 needs to reinstall.

## Why `origin` stays a userConfig option

Because Gesso can be self-hosted, and a hardcoded URL would make that impossible. It is
optional rather than required so the common case — the hosted product — needs no
configuration at all. Required-with-no-sensible-default was the original shape and it was
wrong: it put a setup question in front of every user to serve a minority.

## SKILL.md is generated — never edit it here

`plugins/gesso/skills/gesso/SKILL.md` is copied from the private Gesso repository by
`scripts/sync-plugin-skill.sh` there. Editing it here is silently overwritten on the next
sync, and a stale skill still loads and still looks right — it just teaches the wrong
rules. That script has a `--check` mode for CI.

## No pinned version in the marketplace

`.claude-plugin/marketplace.json` deliberately omits `version`, so installs follow this
repository's commits. Pinning it means every change needs a manual bump, and the
documented failure mode is forgetting: users keep a cached copy indefinitely.

## Verifying a change

`claude plugin validate .` checks shape only. It will not tell you whether the origin
resolves, so also do a real install:

```
claude plugin uninstall gesso@possibleworlds
claude plugin marketplace update possibleworlds
claude plugin install gesso@possibleworlds
claude mcp list | grep gesso
```

The last line is the one that matters. A URL containing a literal `${user_config.origin}`
means the default did not substitute.
