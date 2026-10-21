---
name: gesso
description: All interactions with Gesso, the practice-management work OS for freelance creatives — tasks, projects, phases, deliverables, milestones, time tracking, inbox capture, contacts, notes, docs, files (upload and attach), references and bookmarks (save a link, pin it to a project or grant), the deal and grant pipelines, and finance (read, draft invoices/estimates, log and correct expenses). Use whenever someone wants to add, update, reschedule or complete work; create or amend a project; log time; capture a thought; draft an invoice; win or lose a deal; upload or attach a file; save a link as a reference; look up a client; or ask what is on today. Also handles natural-language capture ("I need to…", "remind me to…", "I've just finished X", "move X to Tuesday"). Trigger on phrasings like "remind me", "add a task", "mark X complete", "move X to Tuesday", "log time", "what's on today", "who is [client]". Do NOT trigger for questions about the Gesso codebase itself — that is ordinary development work.
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
**Stop and say so.** If Gesso was installed as a plugin, the person should check the
plugin's `origin` setting and `claude mcp list`; if it was installed by hand, `SETUP.md`
(shipped beside this file in the repo, not in the plugin) explains how to connect. Do not
fall back to `curl` — the API key is deliberately not in this file.

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

A task attaches to exactly one parent, and it need not be a project: pass **one** of
`projectId` / `dealId` / `grantApplicationId` (or `deliverableId` / `submissionId` /
`phaseId`). Preparing a deal pitch or a grant application is weeks of real work, so
"draft the budget for the Netflix deal" is a task on that deal, not a floating one — take
the id from a `gesso_search` hit's `type` so a deal id never lands in the project slot.

**Inbox triage:** `gesso_list_inbox` shows captured items not yet dealt with. Turning one
into a task is two steps: `gesso_create_task` first, then `gesso_update_inbox_item` with
`resolvedTo` (`task`, `project`, `reference` or `discarded`) and `processedAt`.
`gesso_update_inbox_item` only records the outcome; it creates nothing.

**Prefer capture when you would otherwise be guessing a project.** A misfiled task is
worse than an unfiled one: an unfiled item is visibly waiting in the inbox, whereas a task
on the wrong project silently distorts that project.

### Contacts vs contact people

- **`gesso_list_contacts`** — **organisations**: clients, venues, funders, outlets.
- **`gesso_list_contact_people`** — the **individuals** who work at them.

"Email Sarah at the Tate" involves two records: the Tate is a contact, Sarah is a contact
person affiliated with it.

### Notes vs capture

A "note" is a **Doc** now (the Note model was retired) — create one, then attach it.

- **`gesso_create_doc`** then **`gesso_link_doc`** — a record of something that *happened*,
  attached to a contact, person, project, deal or grant. Put the one-line summary in the
  doc's `title` ("Call with Acme, they want to push the deadline") and set `occurredAt` when
  logging something retrospectively; then `gesso_link_doc` it to the record.
- **`gesso_capture`** — something that needs *doing*.

`gesso_link_doc` needs an `entityType` and `entityId`, and they must agree — an id belonging
to a different type fails with `404 entity not found`. Take the type from the search hit
rather than assuming it. Read a record's log with `gesso_docs_for_entity`.

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
- **`gesso_agenda_today`** is the lower-level agenda alone — time blocks, unscheduled
  tasks, projects in focus, deadline radar, upcoming milestones. It has no nominated
  priority and no overdue list, and it leaves out calendar events. Use `gesso_today`
  unless you want only that.
- **Daily priority:** `gesso_set_daily_priority` takes a `taskId`. There is one per day and
  setting it replaces the current one, so confirm before overwriting a priority the person
  already chose. If `gesso_today` says `dailyPriorityIsPrivate: true`, a private Life
  priority is set: ask before replacing it.

Two things to respect in their output:

- **`unavailable`** — a section that failed. Report the gap; do not present the rest as
  the whole picture.
- **`truncated`** — the scan hit the 100-row cap. The list is a sample. Do **not** answer
  "you're all caught up" off a truncated scan; say it is partial.

### Life items: a separate section, never work

A **Life** project (type "Life", personal workspaces only) is the user's private, non-work
space — the boat, the flat, home admin. `gesso_today`, `gesso_current_projects`,
`gesso_list_projects` and `gesso_list_tasks` take **`includeLife`**. Leave it out and they
follow the user's choice in Settings › AI access ("Include Life in agent planning and
briefings", **on by default**), so their answers may hold Life rows, each marked
**`isLife: true`**. When they do:

- Show Life items in their own **"Life"** section, after the work.
- `gesso_today` already separates them: its top-level sections are always work only (the
  same answer as with Life off). For a user who has Life, Life comes back as a **`life`**
  object — `dailyPriority` (a Life priority sits here, and the top-level one is then null),
  `overdueOrDueToday`, and today's Life `timeblocks`, `floatingTasks`, `reminders`, `radar`
  and `inFocus`. A list that could not be read is `null`, with the reason in
  `life.unavailable`; `[]` means nothing Life. `life.overdueOrDueToday` leaves out tasks
  already in its `timeblocks`, `floatingTasks` or `radar`; `floatingTasks` and `radar` can
  share a task (as on the work agenda), so merge by id before counting. `truncated`/`capped` mean partial. When the user's setting left Life out,
  `lifeExcluded` says so (`"setting"`, or `"settings-unavailable"` when it could not be
  read — say so). A user without Life sees none of these keys, unless the Life check itself
  failed (then `life.unavailable.lifeStatus` says so).
- `dailyPriorityIsPrivate: true` means a private Life priority is set but not shown. It is
  not "no priority": do not offer to set one over it without asking.
- Never fold them into work counts, capacity, workload or priorities, and never present
  them as practice work or bill for them.
- Pass `includeLife: false` for a work-only answer — practice reporting, capacity or "how
  busy am I with clients" questions — and `true` to bring Life in regardless of the
  setting. (The list tools take the strings `"true"`/`"false"`.) That choice is for the one
  call: afterwards, omit it again so the user's setting applies.

Every report (time split, trends, finance, pipeline) stays work-only whatever the setting
says. Search and direct reads by id always find Life items. A Life project cannot hold
invoices, expenses, income, deals, grants, budgets, deliverables, milestones, client portals
or submissions — linking one (or restoring a deleted deal/grant onto one) fails with
`LIFE_PROJECT_NOT_BILLABLE`; tell the user rather than retrying.

Some reads deliberately keep Life, because they are the user's own time and records.
`gesso_calendar_week`, `gesso_list_timeblocks`, `gesso_list_time_entries` and
`gesso_get_daily_priority` mark them: rows with **`isLife: true`** are the user's private
Life items, and these reads do not know the user's setting, so they **fail closed**: leave
`isLife: true` rows out unless the user asks about their Life, or `gesso_today` (called
without `includeLife`) returned a `life` section in this conversation — that means the
user has Life on for agents. No `lifeExcluded` is not enough: `gesso_today` may not have
run, and your own `includeLife` on it, `true` or `false`, does not count. When shown, they go
in the separate Life section; never count them as practice work or capacity. A contact's timeline
(`gesso_get_contact`) and `gesso_list_recurring_tasks` may also include Life items but
carry **no** `isLife` flag, so a missing flag there proves nothing: check the row's
project type (via `gesso_get_project`) before treating it as work.

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

**`gesso_time_trends`** reports logged time bucketed over time — the shape of where hours
have gone, not individual entries.

---

## Calendar and time-blocks — writing here may reach Google Calendar

Reads: **`gesso_calendar_week`** and **`gesso_calendar_range`** show the calendar (time-
blocks and dated items) for a week or an arbitrary range. **`gesso_list_timeblocks`** /
**`gesso_get_timeblock`** read the blocks themselves.

Writes: **`gesso_create_timeblock`** blocks time (`date`, `startTime`, `endTime`, a
`title`, a `type`; optionally a `projectId`/`taskId` link or a `recurrenceRule`);
**`gesso_update_timeblock`** edits one. **`gesso_create_event`** / **`gesso_update_event`**
do the same for events (openings, talks, gigs).

**The one thing to say out loud: if the person has Google Calendar connected, creating or
updating a TIME-BLOCK ALSO writes it to their Google Calendar.** This is the one agent write
that reaches outside Gesso. It is not a *send* in the email sense, but it is visible on a
real external calendar — so treat "block me two hours Thursday" as the real,
externally-visible change it may be, and confirm what you did in the usual one line.
(**Events do not sync** — an event stays inside Gesso.) You cannot delete a block or event
(as everywhere); rescheduling is an update.

## Recurring tasks

`gesso_list/get/create/update_recurring_task` drive the **templates** that generate ordinary
tasks on a schedule — not the tasks themselves. The generated tasks are normal tasks: find,
complete or reschedule them with the task tools, not here. Editing the template changes
future occurrences; it does not retroactively rewrite tasks already spawned.

---

## Finance: read, DRAFT, and log/correct expenses — but never send or transition

The finance surface is: reads — `gesso_list_invoices`, `gesso_get_invoice`,
`gesso_list_expenses`, `gesso_list_income`, `gesso_finance_overview`, `gesso_finance_trends`,
and the accountancy reports `gesso_report_profit_loss` / `gesso_report_balance_sheet` /
`gesso_report_aged_receivables` / `gesso_report_trial_balance`; and **drafting** —
`gesso_create_invoice` / `gesso_update_invoice` and `gesso_create_estimate` /
`gesso_update_estimate`; and **expenses** — `gesso_create_expense` / `gesso_update_expense`. Drafting means exactly that: you can create a draft invoice or
estimate and edit its contents (line items, dates, notes). You **cannot** send one, mark
one paid, issue it, void it, add a payment, or move its status — those are separate
actions the person does in Gesso. Money everywhere is **integer minor units** (pence):
`125000` is £1,250.00, never `1250`.

Beyond drafting, the only finance writes are expense create and update. To log an expense, call `gesso_create_expense`
with `description`, `amount` (pence) and `category`; `paidAt` is the purchase date
(YYYY-MM-DD); set `isBillable` only when it will be recharged to a client. Then attach the
receipt with `gesso_presign_upload`, PUT the bytes, and `gesso_attach_file` with the new
expense's id as `expenseId`. To fix a mis-logged expense use `gesso_update_expense`; changing `amount`, `sa103Category`, `paidAt`
or `currency` reverses and re-posts its ledger entry, so only do it to correct a real mistake. You cannot
delete an expense — say so and the person removes it in Gesso.

**All of these require the `finance` permission, which is off by default.** You will
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

## Pipeline: deals, grants, pitches and submissions

Four opportunity pipelines are drivable — all list/get/create/update — plus a small,
careful set of transitions. The through-line: **the action that books income is never an
agent action**, even when a plain-looking update could otherwise reach it.

- **Deals** (sales). **`gesso_win_deal`** wins a deal — it converts it into an active
  **Project** and is idempotent. Use it, never a status write, to win:
  `gesso_create_deal`/`gesso_update_deal` cannot set `won`, because a bare status flip
  would leave a won deal with no project. **`gesso_mark_deal_lost`** sets `lost`/`on_hold`.
- **Grant applications** (funding). **`gesso_mark_grant_lost`** moves one to
  `rejected`/`withdrawn`. A grant is **awarded** (which books income) only in Gesso. See
  also **saved grants** below — the research-stage bookmarks, a different thing.
- **Pitches** (editorial). `gesso_list/get/create/update_pitch`, seeded by pitch ideas
  (`gesso_*_pitch_idea`). **Accepting a pitch books commission income and is a Gesso
  action** — so `gesso_create_pitch`/`gesso_update_pitch` deliberately have no `status`,
  `fee` or `currency` field. You cannot mark a pitch accepted; say that and stop.
- **Submissions** (open calls, awards, festivals). `gesso_list/get/create/update_submission`.
  Recording a **win books prize income and is a Gesso action** — there is no win tool here.
  A plain status update is fine; it books nothing.

A newly created deal starts at `lead`, a grant at `researching`, a pitch as a draft — you
cannot create any of them already won/successful/accepted. Creating or editing any of these
(within the fields the tools expose) never books income or sends anything: plain `write`.

- **`gesso_pipeline_summary`** — one-call rollup across all four pipelines, counts and
  values by stage. **`gesso_pipeline_trends`** — the same movement bucketed over time.

## Saved grants: research bookmarks, not applications

`gesso_list/get/create/update_saved_grant` are the **funding opportunities bookmarked while
researching** — before any application exists. Distinct from grant *applications* above: a
saved grant is a lead, a grant application is real work in the pipeline. Don't confuse the
two, and don't reach for a saved grant when someone means their live application.

---

## Docs

Rich-text documents are drivable: `gesso_list_docs`, `gesso_get_doc`, `gesso_create_doc`,
`gesso_update_doc`. Docs are **not** in `gesso_search`, so to find the docs on a project
or contact use **`gesso_docs_for_entity`** (pass `entityType` and `entityId`), and to see
which records have docs at all use **`gesso_doc_record_groups`** — rather than listing
everything.

### Writing the body — just write Markdown

**Write the `body` as plain Markdown text.** Gesso stores docs as ProseMirror JSON, but the
API converts a Markdown *string* to that structure server-side on every create and update,
so you do not build the JSON yourself. Passing a raw Markdown string is the correct, simplest
path — the doc renders as rich text, not as literal `#`/`**` glyphs.

Supported Markdown (everything StarterKit covers, plus links and tables):

- `#`/`##`/`###` headings, paragraphs, **bold**, *italic*, `inline code`
- `-` / `1.` lists, `>` blockquotes, fenced ```code blocks```, `---` horizontal rules
- `[text](https://…)` links
- **GFM tables** — `| a | b |` with a `| --- | --- |` separator row. These now become real,
  editable tables (a doc with tables that could not render was the bug this fixed); write the
  header row, the separator, then the body rows.

Two ways to get a broken doc, both avoidable:

- **Do not hand-write ProseMirror JSON as a string** (i.e. a JSON-stringified doc). The API
  detects and recovers that specific shape, but there is no reason to produce it — write
  Markdown. (Passing an already-parsed ProseMirror JSON *object* is fine and passes through
  untouched; it is the *stringified* form to avoid.)
- **Very deep nesting is rejected** with `DOC_BODY_TOO_DEEP` and an oversized body with a
  size error — keep documents to ordinary prose depth.

### Filing a doc, and folders

`gesso_create_doc` takes `tags`. To attach a doc to a **record** (project, deal, grant, …),
use **`gesso_link_doc`** — pass the doc's id and the target's `entityType` + `entityId`,
resolving a valid target first with **`gesso_list_link_targets`**. **`gesso_list_doc_links`**
reads where a doc is already attached. A doc can hang off several records at once.

**Folders are per-object, and shared between an object's docs and files.** A folder like
`"Contracts"` on a project holds both. Create one with **`gesso_create_object_folder`**
(`entityType` = the object type, `entityId`, `path` — a `/`-nested path; ancestors auto-made),
list them with **`gesso_list_object_folders`**. Then file a doc into it by passing `folder` to
`gesso_link_doc`, or move an already-attached doc with **`gesso_set_doc_folder`** (files:
`gesso_set_file_folder`). The same doc can sit in different folders on different records.

You **cannot detach** a doc, or rename/delete a folder, over MCP (no-deletes rule; delete
unfiles content) — done in the Gesso web app.

### Branching and versions

A doc can be **branched** — forked into a working copy so a draft can be developed without
touching the original (e.g. iterating a grant application off a rough first pass).

- **`gesso_fork_doc`** — pass the source doc's id; returns a new branch linked back to it. By
  default it copies the original's content and its record attachments and lands in the same
  folder. The original is snapshotted first, so nothing is lost.
- **`gesso_save_doc_version`** — snapshot a doc's current state as a named point (optional
  `label`) you can return to later. Purely additive.
- **`gesso_list_doc_versions`** / **`gesso_list_doc_branches`** — read a doc's saved versions
  (newest first) or the branches forked from it.

**Promoting a branch back onto its original, and restoring a doc to an earlier version, are
app-only** — both overwrite a doc's content wholesale, so they are a person's decision, not an
agent action (the same line that keeps sending and detaching out of the tools).

---

## Files: uploading and attaching

Uploading a file is **three steps, and you do the middle one yourself** — Gesso never
carries the bytes:

1. **`gesso_presign_upload`** with `filename`, `mimeType`, `sizeBytes` (the real byte
   count) → returns `{ presignedUrl, storageKey }`.
2. **PUT the bytes to `presignedUrl` yourself.** It is a direct Backblaze upload link,
   valid ~15 minutes. In Claude Code:
   `curl -X PUT -H 'Content-Type: <mimeType>' --upload-file <path> '<presignedUrl>'`.
   This does not go through Gesso, and if your host cannot make an outbound PUT you cannot
   upload this way — say so rather than pretending it worked.
3. **`gesso_attach_file`** with the `storageKey` from step 1, the same
   `filename`/`mimeType`/`sizeBytes`, and an attach target: `projectId`, `taskId`,
   `contactId`, `deliverableId`, `dealId`, `grantApplicationId`, `invoiceId` or `expenseId`.
   This records the file and links it — skip step 2 and you record a file that points at
   nothing.

**Uploads through an agent are capped at 100 MB.** This is an agent-surface limit, not a
Gesso one — if a file is bigger, tell the person to upload it in the Gesso web app under
the object's Files tab. Over-cap uploads fail with a message saying exactly this.

Reading and organising:
- **`gesso_files_for_object`** — every file on one object, each with its `linkFolder`.
- **`gesso_list_files`** — files on one object (pass its attach-point id).
- **`gesso_drive_list`** / **`gesso_drive_tree`** — the whole library, the Drive view.
  Files are **not** in `gesso_search`, so this is how you find one you can't locate by its
  object.
- **`gesso_get_file_download_url`** — a short-lived link, safe to hand to a person.

**Folders and multi-attach.** A file can be attached to **several** objects, each with its
own **folder** (a `/`-nested label scoped to that object — the same file can be in
`"Contracts"` on a project and `"Legal"` on a deal). Use **`gesso_attach_file_to_object`**
(with an optional `folder`) to add a placement, **`gesso_set_file_folder`** to move it
between folders or unfile it on one object, and **`gesso_list_file_links`** to see where it
is filed. `gesso_move_file` still re-files a file's primary Drive placement.

You **cannot delete or detach** a file — as everywhere, that is theirs to do in the app.

---

## References: the Library, Bookmarks and an object's References tab

These are **one store, not three**. A Reference is a saved link or note; the app calls the
unscoped view **Bookmarks**, the API calls it the **Library**, and an object's
**References** tab is the same rows filtered to what is pinned to that Project, Grant or
Deal. So "my bookmarks", "the Library" and "the references on this grant" all resolve to
`gesso_list_library` — the first two unfiltered, the last with `entityType` + `entityId`.

References are **not** in `gesso_search`; `gesso_list_library` is how you find one.

You **cannot unpin or delete** a reference — those tools do not exist. The asymmetry is
deliberate rather than an oversight: removing a reference from one object and deleting the
Library item are different acts, and the second pulls it off *every* object it is attached
to. Both are theirs to do in the app. Say that, rather than reporting that Gesso cannot.

The tool descriptions carry the rest — the create-then-link order, which fields each
`type` needs, and the valid `entityType` values are all in the schemas.

---

## Social: a PLANNER, not a publisher

The social tools plan content — Ideas, Posts and Campaigns — they do **not** post anything
to any network. Gesso has no connection to Instagram, X, LinkedIn or anywhere else; the
publishing integration was removed. So:

- **Ideas** (`gesso_*_social_idea`) — captured content ideas. **Posts**
  (`gesso_*_social_post`) — planned posts, promoted from an idea by creating a post with the
  idea linked. **Campaigns** (`gesso_*_social_campaign`) — groupings (a launch push, BAU).
  These are Gesso's own campaigns, unrelated to email-marketing campaigns.
- **`gesso_social_calendar`** — planned posts in a date window, a lean per-post projection.
- **`gesso_set_social_post_status`** moves a post through the *planner* stages. **This does
  not publish.** **`gesso_mark_social_post_posted`** records that a human posted it
  MANUALLY, elsewhere (with an optional live URL) — again, it publishes nothing.

Asked to "post this to Instagram", do the planning part — draft the post, mark it ready —
and say plainly that the actual posting happens outside Gesso and they mark it posted after.
Same shape as the send rule: *agents draft, humans dispatch.*

---

## Notifications

**`gesso_list_notifications`** reads the current user's alerts (invoice overdue, grant
deadline, and similar, produced by background jobs). Read-only — marking one read, or all
read, is a Gesso action, not an agent one. Useful for "anything I'm being warned about?".

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
phases, deliverables, milestones, time, capture, contacts, notes, docs, files (upload,
attach, organise), references, recurring-task templates, calendar time-blocks (which may
mirror to a connected Google Calendar) and events, the social-content planner, and all four
opportunity pipelines — deals, grant applications, pitches and submissions (including
winning a deal into a project). You can READ pipeline and time trends and notifications, and
with the `finance` permission also the finance reports, DRAFT invoices and estimates, and expense create/update. You
cannot send or publish anything, delete, bulk-edit, issue or mark-paid an invoice, award a
grant, accept a pitch, record a submission win (all three book income), touch payments or
memberships, or change org settings. When asked for one of those, say which part you did and
which part is theirs.
