---
name: todo-list-refresh
description: Scans Sandeep's Microsoft 365 (Outlook email, Teams chats, Outlook calendar, recent meeting transcripts) for new actionable tasks since the last sync and inserts them into the `todo.tasks` table in the benefit-poc Supabase project. ALWAYS use this skill when the user says any of "refresh my todos", "refresh my tasks", "update my task list", "scan email for tasks", "what's new since last sync", "run the todo refresh", or when a scheduled task asks for a TODO refresh / inbox-to-task sync. This is the *inbound* side of Sandeep's TODO system (the *outbound* side, Teams reminders, is handled separately by the Supabase edge function `todo-send-reminders`).
---
 
# TODO List Refresh
 
This skill scans Sandeep's Microsoft 365 surfaces for new actionable items since the last sync and pushes them into Supabase. It is designed to run on a recurring schedule (typically hourly during work hours) but is also safe to invoke manually.
 
## What this skill does, end to end
 
1. Read the per-source `last_synced_at` timestamps from `todo.sync_state` in Supabase.
2. For each source (email, teams, calendar, meetings), fetch items created or modified since that timestamp.
3. Identify which of those items are *actionable tasks for Sandeep* — apply the judgment rules in [What counts as a task](#what-counts-as-a-task).
4. Insert new tasks into `todo.tasks`, deduplicating on `origin_ref` (the unique constraint catches accidental re-inserts).
5. Update `todo.sync_state.last_synced_at` for each source to `now()`.
6. Report back to the user: how many new tasks were found per source, and a short summary.
The Supabase MCP and Microsoft 365 MCP are both required. If either is missing, stop and tell the user.
 
## Project constants
 
- Supabase project ID: `brmrgalpyhbzlviooryq`
- Schema: `todo`
- Tables: `todo.tasks`, `todo.sync_state`, `todo.reminder_log`
- Sandeep's email: `sandeepkatti@graphenesvc.com`
## Step-by-step procedure
 
### Step 1 — Read sync state
 
Use `Supabase:execute_sql` on project `brmrgalpyhbzlviooryq`:
 
```sql
select source, last_synced_at from todo.sync_state order by source;
```
 
Note the four timestamps. Each source is scanned independently from its own watermark.
 
### Step 2 — Scan each source
 
Run these scans in any order. For each, only consider items received/modified after that source's `last_synced_at`.
 
**Email** — `Microsoft 365:outlook_email_search` with `afterDateTime` set to the sync state value, `folderName: "Inbox"`, `limit: 30`. Inspect subject, sender, summary. Read full body via `read_resource` only if subject/summary is ambiguous.
 
**Teams chats** — `Microsoft 365:chat_message_search` with `afterDateTime` set to the sync state value, `query: "*"`, `limit: 50`. Skip messages sent BY Sandeep himself (`displayName: "Sandeep Katti"`); only incoming messages can be actionable.
 
**Calendar** — `Microsoft 365:outlook_calendar_search` with `afterDateTime: now()`, `beforeDateTime: now() + 24 hours`, `query: "*"`, `limit: 20`. Capture upcoming meetings as tasks only if they require *preparation* (a doc to review, a decision to bring, an attendee profile to read) — a routine standup is not a task.
 
**Meeting transcripts** — if a calendar event in the past hour has a transcript URI, use `read_resource` with `meeting-transcript:///{meetingId}`. Look for explicit action items ("Sandeep will...", "Sandeep to follow up on...", "@Sandeep: please ..."). If transcripts aren't available for a tenant, skip this source.
 
### Step 3 — Judge what's a task
 
See [What counts as a task](#what-counts-as-a-task) below. Be conservative — better to miss a borderline item than to flood the list. The current task list size matters: if there are already 15+ open tasks, raise the bar; if there are fewer than 5, you can be a bit more generous.
 
### Step 4 — Insert with deduplication
 
For each new task, insert using parameterised SQL through `Supabase:execute_sql`. Always set `origin_ref` to the stable unique ID of the source item:
 
| Source | origin_ref format |
|---|---|
| email | `mail:///{messageId}` (URI returned by outlook_email_search) |
| teams | `teams:///chats/{chatId}/messages/{messageId}` |
| calendar | `calendar:///events/{eventId}` |
| meetings | `meeting-transcript:///{meetingId}#{actionItemHash}` (hash a short stable string for each distinct action item within one transcript) |
 
Use `on conflict (origin_ref) where origin_ref is not null do nothing`:
 
```sql
insert into todo.tasks (title, details, source, origin_type, origin_sender, origin_ref)
values
  ('...title...', '...details...', 'official', 'email', 'someone@graphenesvc.com', 'mail:///AAMk...'),
  ...
on conflict (origin_ref) where origin_ref is not null do nothing
returning id, title, origin_type;
```
 
The `returning` clause tells you which rows actually inserted (conflicts return nothing). Use the count of returned rows for the per-source summary in step 6.
 
Keep titles under ~80 characters. Front-load the verb ("Approve...", "Review...", "Respond to...", "Confirm..."). Put context in `details` not `title`.
 
### Step 5 — Update sync state
 
After all four scans complete (even if some found zero tasks):
 
```sql
update todo.sync_state
set last_synced_at = now(), updated_at = now()
where source in ('email','teams','calendar','meetings');
```
 
Only update sources whose scan completed without error. If the Teams scan failed but email succeeded, only bump email's watermark — otherwise you'd silently miss whatever the Teams scan would have found on retry.
 
### Step 6 — Report
 
A short summary in chat. Format:
 
```
TODO refresh complete.
  Email: 2 new tasks
  Teams: 1 new task
  Calendar: 0
  Meetings: 0
Total: 3 new tasks added · 7 open in total now.
```
 
If new tasks were added, list their titles below. Don't list anything else.
 
## What counts as a task
 
A task is something that **requires Sandeep's action**, has a **specific outcome**, and is **not already obviously handled**.
 
### Yes, this is a task
 
- Direct ask: "Sandeep, can you approve X?", "Please review the deck", "Need your sign-off on..."
- Decision pending on Sandeep: a thread where someone is waiting on his call
- Bookings/confirmations: hotel/restaurant/travel that needs his explicit yes
- Items where someone CC'd him and asked for input
- Action items assigned to him in meeting transcripts
- Calendar events in next 24h that need prep (slides to make, attendees to brief on, agenda to draft)
- HR/operational asks: WFH approvals, leave requests, sign-offs
### No, this is not a task
 
- FYI/notification emails (newsletters, system alerts, marketing, security digests, quarantine reports)
- Threads Sandeep is on but where the action is on someone else
- Messages Sandeep himself sent
- Routine recurring meetings on the calendar (standups, weeklies) — only flag if prep is genuinely required
- Replies that just acknowledge ("got it", "ok", "thanks")
- Bot messages, automated notifications
- Marketing/sales pitches even if addressed to him personally
- Discussion threads without a clear ask of him
### Edge cases — judgment calls
 
- A status update *from his report* might be FYI or might be an implicit ask for direction. Lean toward "task" if the report is escalating or stuck.
- A long email thread where Sandeep was the last to reply is usually *not* a task — he's already acted.
- An invite to an external event/meal is a task only if it asks for confirmation; once he's accepted, it stops being a task.
## Dedup is critical
 
This skill runs hourly. The same email may appear in two consecutive scans because the watermark advances *after* the scan starts. The `origin_ref` unique constraint will catch this, but you should still try to filter on `afterDateTime` strictly to avoid wasted work and avoid filling up the LLM context with already-seen items.
 
If you somehow create a duplicate without `origin_ref` (e.g., manual judgment splits one email into two tasks), that's fine — manual tasks aren't governed by the constraint. But within one run, don't insert the same `origin_ref` twice in one batch.
 
## When to ask Sandeep before inserting
 
Default: don't ask, just insert. The reminder cron will surface them and Sandeep can archive any that don't belong.
 
Exception: if a single item looks ambiguous *and* would be high-effort to undo (e.g., a sensitive HR matter), insert it but flag in the summary report: "Inserted X — flag if not right and I'll archive."
 
## When NOT to run
 
Skip the scan entirely if:
- The user is asking about a specific task ("What did Mohan email about?") — answer the question directly, don't refresh.
- The user is in the middle of triaging tasks — adding more during triage is disruptive. Wait for them to finish.
- A previous refresh ran within the last 5 minutes — say so and skip; suggest they wait or pass `force=true`.
## Setting this up on a schedule
 
To run this hourly, Sandeep should create a scheduled task at `claude.ai/code/scheduled` with:
- **Prompt**: "Refresh my TODO list" (or just "run the todo-list-refresh skill")
- **Schedule**: Hourly during work hours (8 AM – 6 PM IST on weekdays)
- **Connectors**: ensure Microsoft 365 and Supabase are both enabled for the task
The skill description above is intentionally aggressive on trigger keywords so the scheduled task reliably routes to it.
 