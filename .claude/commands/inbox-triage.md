---
description: Sort inbox into decide/delegate/defer/delete and draft replies
argument-hint: "[max-emails] [--since 2d] [--dry-run]"
---

# Inbox Triage

You are running the daily Inbox Triage routine. Goal: read the inbox, sort
every message into **Decide / Delegate / Defer / Delete**, draft replies for
anything that needs one, save a dated report, and print a short summary.

Arguments (all optional, parse from `$ARGUMENTS`):
- A bare number = max emails to process (default `25`).
- `--since Nd` = only messages newer than N days (default `2d`, i.e. since
  the last run for a daily cadence).
- `--dry-run` = do the sort and write the report, but skip creating Gmail
  drafts (useful for testing).

## Step 0 — Check inputs are available

This routine reads from **Gmail (inbox)** via the `Gmail` MCP connector.

- Try a small Gmail call first (e.g. `search_threads` with `in:inbox`).
- If the call fails with an auth/authorization error: **stop** and tell the
  user their Gmail connector needs to be (re)connected via claude.ai
  connector settings (Settings → Connectors → Gmail). Do not guess or
  fabricate inbox contents.
- If no Gmail connector is configured at all: tell the user exactly what to
  connect — the Gmail connector (for reading inbox + creating drafts) — and
  stop. Optionally mention Calendar as a *future* enhancement if the user
  later wants deadline-awareness, but it is not required for this routine.
- If Gmail is reachable but empty, note that in the summary and still write
  a report (all categories empty).

## Step 1 — Pull the inbox

- Use `search_threads` with query `in:inbox newer_than:<since>` (map the
  `--since` value, default `2d`), `pageSize` up to the max-emails argument
  (default 25), `view: THREAD_VIEW_MINIMAL`.
- For each thread, use `get_thread` with `messageFormat: PLAIN_TEXT` to read
  the latest message's real body (snippets from search are not enough to
  judge intent) — but only fetch full bodies for threads you actually need
  to read closely; skip obvious bulk/promo mail if the snippet alone is
  conclusive (see Step 2).

### Dedupe against previous runs

The `--since` window (default 2d) deliberately overlaps between consecutive
daily runs so nothing slips through, but that means the same thread often
reappears in more than one run. To avoid re-listing it and — more
importantly — **never create a second Gmail draft on a thread already
drafted in a prior run**:

- Read `output/inbox-triage/.state.json` if it exists: `{"seen_thread_ids":
  {"<threadId>": "<ISO date first seen>"}}`.
- Any fetched thread whose id is already a key in `seen_thread_ids` is
  **already triaged** — do not draft a reply for it again (even if it would
  land in Decide/Delegate) and do not list it individually in the report;
  just roll it into a one-line "N already-triaged threads carried over,
  skipped" note.
- Only categorize, draft, and list threads whose id is *not* yet in the
  state file.
- After the run, write back `.state.json` with every thread id processed
  this run added (new + already-seen, refresh their date), pruning any
  entries whose stored date is older than 14 days so the file doesn't grow
  unbounded.
- If `.state.json` doesn't exist yet (first run), treat every fetched
  thread as new and create the file.

## Step 2 — Categorize each thread

Sort every thread into exactly one bucket:

- **Decide** — needs the user's own judgment or a substantive reply only
  they can give: a direct question addressed to them, an approval/decision
  request, a negotiation, anything with a deadline that's their call.
- **Delegate** — actionable, but someone else should handle it (or the
  reply is just "assigning/forwarding"). Note *who* it should go to if the
  email makes that obvious (e.g. a name in cc, or a role it clearly belongs
  to).
- **Defer** — no action needed right now but not junk: FYI threads,
  newsletters worth skimming later, receipts, low-urgency updates. Revisit
  later, no reply needed today.
- **Delete** — spam, promotions, obvious noise, or threads already fully
  resolved (e.g. "thanks, all set!" tail ends). Do not actually delete or
  trash anything — this is a label/report only. Never send/trash/mark-spam
  automatically.

## Step 3 — Draft replies

Unless `--dry-run` was passed: for every thread in **Decide** or
**Delegate** that plausibly needs a reply, create a Gmail draft with
`create_draft` (`replyToMessageId` = the latest message id in that thread).

- Keep drafts short (2-5 sentences), matching the tone of the thread.
- For **Decide**, draft a reply that surfaces the decision needed and
  proposes a default answer the user can edit or send as-is.
- For **Delegate**, draft a short forward-style note ("Looping in X — can
  you take this?") if a clear delegate is obvious; otherwise draft a holding
  reply ("received, will route this to the right person").
- **Never call `send_message`, `forward`, `trash_*`, or any spam/label
  action.** This routine only reads and drafts — it never sends, deletes,
  or modifies the inbox otherwise. Drafts stay in Gmail's Drafts folder for
  the user to review and send.
- Record the resulting draft id/link for the report.

## Step 4 — Write the report

Write a Markdown file to `./output/inbox-triage/YYYY-MM-DD.md` (use today's
date; if the file already exists because this ran twice today, append a
`-2`, `-3`, ... suffix). Create the `output/inbox-triage/` directory if it
doesn't exist. Structure:

```markdown
# Inbox Triage — YYYY-MM-DD

Processed N threads (since <window>).

## Decide (n)
- **Subject** — from Sender <email> — one-line why — [Draft created](gmail draft id) | _no draft: reason_

## Delegate (n)
- **Subject** — from Sender <email> — suggested owner: X — [Draft created](...)

## Defer (n)
- **Subject** — from Sender <email> — one-line why it can wait

## Delete (n)
- **Subject** — from Sender <email> — one-line why (spam/resolved/etc.)
```

## Step 5 — Print a terminal summary

After writing the file, print a short summary (not the full report) to the
user, e.g.:

```
Inbox Triage — YYYY-MM-DD
Decide 3 · Delegate 2 · Defer 6 · Delete 4  (15 processed)
Drafts created: 5
Report: output/inbox-triage/YYYY-MM-DD.md
```

Keep this to a handful of lines — the full detail lives in the saved file.
