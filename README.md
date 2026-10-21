# Gesso for Claude

Drive [Gesso](https://github.com/PossibleWorldsDotSpace) — the practice-management work OS
for freelance creatives — from Claude Code or Claude Desktop.

```
/plugin marketplace add PossibleWorldsDotSpace/gesso-plugin
/plugin install gesso@possibleworlds
```

Then run `/mcp`, pick `gesso`, and approve in the browser. No API key to mint, no
environment variable to export, and nothing to configure.

> **Running your own Gesso?** Set the `origin` option to your deployment's address, with
> no trailing slash. Everyone else can leave it alone.

## What you get

**44 tools** — projects, tasks, phases, deliverables, milestones, time tracking, inbox
capture, contacts, notes, global search, and finance (read-only, apart from logging and correcting expenses). Plus two aggregate reads,
`gesso_today` and `gesso_project_context`, that answer "what's on today" and "tell me
about this project" in one call instead of five.

**And the skill.** This is the half that is easy to skip and expensive to miss. The tools
carry schemas; the skill carries the judgment a schema cannot express:

- money is in **pence**, not pounds — a 100× error that reads as perfectly reasonable
- dates are **Europe/London** days, and Gesso stores UTC
- `limit` above 100 is rejected outright, not clamped
- when to capture a thought versus create a task, and which of `contacts` (organisations)
  or `contact-people` (individuals) you actually mean

Installing them together is the point of this plugin. Connecting the server alone gives
you working tools with none of the guardrails, and nothing tells you what is missing.

## What it deliberately cannot do

- **Send anything.** No tool emails a client, sends an invoice, or issues an invite — and
  no permission enables one. *Agents draft, humans dispatch.* This is enforced server-side
  too, so it holds regardless of what any client asks for.
- **Delete anything.** Tasks are completed, projects are iced or completed.
- **Bulk-edit, or touch payments, memberships or org settings.**

Finance is **read-only, apart from logging and correcting expenses**, and off unless you grant it during authorisation.

## Requirements

- A Gesso account, and a deployment served over **HTTPS** — OAuth 2.1 clients refuse to
  send a token request over plain HTTP.
- Claude Code. Claude Desktop reads the same configuration, but the plugin is installed
  from Claude Code.

## Connecting over SSH

If Gesso is on a machine you reach over SSH, the browser approval redirects to
`localhost`, which is your **laptop's** localhost rather than the server's — so the
authorization code is delivered to the wrong machine and nothing completes.

Pin the callback port and forward it. On your laptop:

```bash
ssh -L 3333:localhost:3333 your-server
```

Then add the server with `--callback-port 3333`. The tunnel is only needed for the
approval; close it afterwards, since tool calls and token refresh never touch localhost.

## Updates

`/plugin marketplace update`, then reinstall. There is deliberately no pinned `version`
in the marketplace entry, so updates follow this repository's commits rather than waiting
on a manual version bump.

## Where the source lives

The MCP server and the canonical skill live in the Gesso repository; this repository is
the published plugin. `SKILL.md` here is **generated** from that source rather than
hand-maintained, so the two cannot drift.
