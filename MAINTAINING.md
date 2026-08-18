# Maintaining this plugin

## The default origin is staging, and will break for the public

`plugins/gesso/.claude-plugin/plugin.json` defaults `userConfig.origin` to:

```
https://ubuntu-2404-noble-amd64-base.taildb1547.ts.net:7443
```

That is a **tailnet address**. It resolves only from the Possible Worlds tailnet, so
anyone else installing this plugin today gets a server they cannot reach. The failure is
quiet in the worst way: the plugin installs cleanly, `claude mcp list` shows the server,
and it simply never connects.

This is deliberate and temporary. Gesso is pre-launch, the tailnet host is the only
deployment, and asking every user for a URL is wrong for a product that will have exactly
one origin.

**On the first production deploy, change that default to `https://getgesso.app`.** One
line, no other change — self-hosters keep overriding it through the same option.

Until then, treat the marketplace as effectively private even though the repository is
public.

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
