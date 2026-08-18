---
name: gesso
description: All interactions with Gesso, the practice-management work OS for freelance creatives — tasks, projects, phases, deliverables, milestones, time tracking, inbox capture, contacts, notes, and read-only finance. Use whenever someone wants to add, update, reschedule or complete work; create or amend a project; log time; capture a thought; look up a client; or ask what is on today. Also handles natural-language capture ("I need to…", "remind me to…", "I've just finished X", "move X to Tuesday"). Trigger on phrasings like "remind me", "add a task", "mark X complete", "move X to Tuesday", "log time", "what's on today", "who is [client]". Do NOT trigger for questions about the Gesso codebase itself — that is ordinary development work.
compatibility: Requires the Gesso MCP server to be configured in the client — the tools are the only way in; there is no fallback.
metadata:
  author: possible-worlds
---

# Gesso

Gesso is reached through the **`gesso_*` MCP tools** — never `curl`, never a hand-written
REST call. The tools carry their own schemas, so **trust the tool definition over anything
written here if the two ever disagree.** This file exists for the things a JSON Schema
cannot say.

If the `gesso_*` tools are not available, the MCP server is not configured in this client.
**Stop and say so**, pointing at `SETUP.md` beside this file. Do not fall back to `curl` —
the API key is deliberately not in this file.

---

## Non-negotiable rules

The reason this skill exists. Every one of these has a plausible-looking wrong answer.

- **Money is integer minor units — pence, never pounds.** `125000` is £1,250.00. A finance
  tool returning `45000` means £450.00. Say the pounds figure to the person; never do
  arithmetic that treats the integer as pounds. This is a 100× error that reads as
  reasonable.
  The rule is about **money** fields, not every number on a financial record: `vatRate` is
  whole percent (`20` means 20%, not £0.20) and `fxRate` is a decimal string. If a field
  name describes a rate rather than an amount, it is not pence.
- **Dates are `YYYY-MM-DD`, meaning that day in Europe/London.** Gesso stores UTC and
  displays London. "Tomorrow" depends on the London date, not UTC's — check the current
  date before computing one. In BST (summer) London is UTC+1, so late-evening UTC is
  already the next day in London.
- **Where `limit` exists it is capped at 100.** A request for more is rejected outright
  with a 400 — the call fails, it does not silently clamp. Never "fetch everything" by
  raising the limit; page with `offset` where the tool offers one.
  **Not every list tool paginates.** `gesso_list_deliverables`, `gesso_list_milestones`
  and `gesso_list_time_entries` take neither `limit` nor `offset` — they return what they
  return. `gesso_current_projects` caps at 20, and `gesso_search` returns at most 20 hits
  (5 per type) with no way to ask for more. If a result set looks suspiciously round,
  assume it is a ceiling and say so rather than reporting it as the complete picture.
- **You cannot send anything.** There is no tool that emails a client, sends an invoice,
  dispatches an estimate or issues an invite, and no permission that would enable one.
  This is deliberate: *agents draft, humans dispatch.* Asked to "email the client the
  invoice", do the part you can (find it, report it) and say plainly that sending is
  something they do in the app.
- **You cannot delete.** No delete tools exist. Asked to remove something, say so and
  offer the reversible alternative — a task can be set to `done`, a project to `complete`
  or `iced`.
- **Report what happened in one line.** After a write, confirm concisely:
  `Added to Gesso: Draft the Tate proposal — due 2026-07-08`. Not a paragraph, and never a
  restatement of the whole record.

---

## Finding things: search first, always

`gesso_search` is how a name becomes an id. It covers projects, tasks, contacts, people,
invoices, deals, grants, submissions, canvases, events and exhibitions, ranked across all
of them.

It is cleverer than it looks, and knowing that avoids a lot of wasted calls:

- **Invoices and deals are findable by their client's name**, grants by their funder,
  submissions by their venue. "The Acme invoice" resolves without knowing the number.
- Every hit carries an **`href`** — the real in-app path. Hand that to the person when
  they might want to open it. Do not construct URLs yourself.
- Hits carry `type`, so you can tell a project called "Zephyr" from a contact called
  "Zephyr" before acting on the wrong one.

**Never guess an id, and never page a list endpoint hunting for a name.** If search
returns several plausible matches, ask which one rather than picking the top hit — acting
on the wrong entity is worse than a question.

---

## Choosing the right tool

The pairs below are the ones most easily confused. Getting these wrong produces a
successful call that records the wrong thing.

### Capture vs create a task

- **`gesso_capture`** — a raw thought, when the project is unclear. "I should probably
  chase the framer." Goes to the inbox for the person to sort later.
- **`gesso_create_task`** — a real task, when you know where it belongs.

**Prefer capture when you would otherwise be guessing a project.** A misfiled task is
worse than an unfiled one: an unfiled item is visibly waiting in the inbox, whereas a task
on the wrong project silently distorts that project.

### Contacts vs contact people

- **`gesso_list_contacts`** — **organisations**: clients, venues, funders, outlets.
- **`gesso_list_contact_people`** — the **individuals** who work at them.

"Email Sarah at the Tate" involves two records: the Tate is a contact, Sarah is a contact
person affiliated with it.

### Notes vs capture

- **`gesso_create_note`** — a record of something that *happened*, attached to a contact,
  person, project, deal or grant. "Call with Acme, they want to push the deadline."
- **`gesso_capture`** — something that needs *doing*.

Notes need both `entityType` and `entityId`, and they must agree — an id belonging to a
different type fails with `404 entity not found`. Take the type from the search hit
rather than assuming it.

### Tasks vs deliverables vs milestones

Three different things, one project:

- **Task** — a unit of work someone does.
- **Deliverable** — a tangible output the project owes the client.
- **Milestone** — a dated checkpoint. Requires a date; without one it is not a milestone.

Note `gesso_create_milestone` defaults **`clientVisible: true`** — it appears on the
client-facing portal. Pass `false` for anything internal.

---

## Reading the day

Prefer the aggregates. They are one call instead of five, and they encode definitions you
would otherwise have to invent.

- **`gesso_today`** — the agenda, the nominated priority, and everything overdue or due
  today. The right answer to "what's on", "what should I do now", a morning standup.
- **`gesso_project_context`** — a project with its phases, tasks, deliverables and
  milestones. The right answer to any general question about a project.

Two things to respect in their output:

- **`unavailable`** — a section that failed. Report the gap; do not present the rest as
  the whole picture.
- **`truncated`** — the scan hit the 100-row cap. The list is a sample. Do **not** answer
  "you're all caught up" off a truncated scan; say it is partial.

---

## Completing and rescheduling

- **To complete a task, set `status: "done"`** via `gesso_update_task`. There is no
  separate complete endpoint.
- **Status is `todo` | `doing` | `done`.** "In progress" is `doing`.
- **To reschedule, set `dueDate`.** A date the person states is fixed — never substitute a
  "better" one.
- **Projects** move between `active`, `iced` and `complete`. Archiving is not a status; it
  is a delete, which you cannot do.

---

## Logging time

`gesso_log_time` records time **retrospectively** — it does not start a timer, and there is
no way to start one. Both fields are required:

- `date` — the day the work happened, `YYYY-MM-DD`.
- `durationMinutes` — whole minutes, as an integer. Ninety minutes is `90`, not `1.5`.

"An hour and a half on the Tate project yesterday" → resolve the project with
`gesso_search`, then log `90` against yesterday's London date.

---

## Finance is read-only, and may not be visible at all

`gesso_list_invoices`, `gesso_get_invoice`, `gesso_list_expenses`, `gesso_list_income` and
`gesso_finance_overview` are the whole finance surface. There is no tool to create an
invoice, mark one paid, or send anything.

**These tools require the `finance` permission, which is off by default.** You will
usually still SEE them in your tool list — the list is only filtered when the client
declares its scopes — so the way you find out is a `403 INSUFFICIENT_SCOPE` on the call.

That 403 is a deliberate setting, not a fault. Relay it and stop: the fix is a new key
with that permission, minted in Settings › API keys, and retrying cannot help because a
new key with the same scopes fails identically.

**Do not reconstruct finance answers from other tools.** Without the scope, search returns
no invoices, deals or grants, and `gesso_today` reports its money figures as `redacted`
with zeroes. Those zeroes mean "not permitted to see", **not** "nothing outstanding" —
never report them as a real figure.

---

## When something goes wrong

- **403 / "missing the … scope"** — the key lacks a permission. Report it; do not retry.
- **400 on a list call** — most often `limit` above 100. Page instead.
- **A tool you expected is missing** — it is either out of scope for this key, or it does
  not exist because the capability is deliberately withheld (sends, deletes, bulk edits).
  Check the tool list before telling someone Gesso cannot do something: the *app* can do
  plenty that the *agent surface* deliberately does not.
- **Empty results** — distinguish "nothing matches" from "I filtered wrongly". A list
  endpoint scoped to the wrong project returns `[]` just as convincingly as a genuinely
  empty one.

---

## Scope, in one line

You can read most of the practice and write to the day-to-day of it: tasks, projects,
phases, deliverables, milestones, time, capture, contacts, notes. You cannot send, delete,
bulk-edit, touch payments or memberships, or change org settings. When asked for one of
those, say which part you did and which part is theirs.
