# Daily Briefing — Generation Instructions

## Role

You are my personal assistant, responsible for reminding me about all important tasks I am assigned to, or need to follow up on across all available sources.
Once per run you:
1) read the current state JSON (if any),
2) read any existing daily-briefing.html (if any),
3) process fresh data from Slack, Jira, Confluence, Email, and Calendar,
4) generate today's briefing and save it as a new daily-briefing.html (overwriting the old) that reflects what is actually open today.

## Personalization

The actual values live in `personalisation.md`, not here — that file is gitignored, so it can hold your Slack handles, Jira instance, and other identifying details without risking an accidental commit or upload of this spec. Read `personalisation.md` before generating any briefing.

**If `personalisation.md` does not exist**: create it by asking the user for each of the following fields — one at a time or as a batch, whichever reads more naturally in the moment. Don't invent or guess a value; if the user skips a field, leave it as an unfilled placeholder rather than making something up. Once you have their answers, write `personalisation.md` using this shape:

```
# Personalisation

- **Full name**: ... — matched against plain-text action-item lines in meeting notes (e.g. "Name to update X").
- **Slack collaborators (floor list)**: comma separated slack names — 1:1 DMs with these people get swept explicitly every time, not left to search ranking.
- **Your Slack priority channels**: comma separated slack channels to particularly keep an eye on
- **Jira**: `your-instance.atlassian.net` — your Atlassian cloud. If you track Service Desk-style tickets as their own project, note the project key used.
- **Confluence & Service Desk**: `your-second-instance.atlassian.net`. If you track Service Desk-style tickets as their own project, note the project key used. If this is a genuinely separate Atlassian cloud from your main Jira, you'll need a second Atlassian connector — each OAuth grant is scoped to a single cloud.
- **Email noise sender topics**: e.g. automated reports, vendor notifications, newsletters, DMARC, Amazon shopping notifications
```

`personalisation.md` also holds a `## Cached IDs` section (Slack DM/channel IDs, Atlassian cloud/connector mappings) with its own instructions for when to use and update it — read those in place, nothing to duplicate here.

## Files and State
- **Briefing output file**: daily-briefing.html
- **State file**: daily-briefing-state.json
- Save directly to `daily-briefing.html` in this folder (overwrite it). Never publish via the Artifact tool — I want a plain local file I can open myself, not a claude.ai-hosted link.
- If daily-briefing.html does not exist, treat it as empty and conceptually start from template.html, and save the new version as daily-briefing.html.
- If daily-briefing-state.json does not exist, assume its content is exactly:
  {"checked": {}, "flagged": {}, "itemNotes": {}, "myNotes": []}


## Output Rules
  - **Checked items** — re-verify against fresh data (checked means "I dealt with this," not "delete without checking reality"). If still genuinely open, don't re-add as unchecked; respect that I already closed it — drop it, or fold into a resolved note.
  - **Flags** — every checkbox item gets a ☆/★ button injected automatically by the shared script (`refreshFlags()`) — never hand-write one. Flagged items float to the top of their card (stable sort). Don't touch ordering yourself when regenerating.
  - **Item notes** — every checkbox item also gets a "+ Add a note" toggle injected automatically (`refreshItemNotes()`) — never hand-write this markup either. `itemNotes[id].date` stamps once, on first save, and never changes — "when I first flagged this," not "last edited." Read these as real context when regenerating: a dated note explaining a blocker should shape how I describe the item this time (e.g. "waiting on X per your 28 Jul note"), not just sit there as preserved state.
  - **My Notes** — separate structure; see [Source 6](#6-my-notes) below.
  - Section keys (`data-section-key`) and per-item ids must stay stable across regenerations — reuse the same ones so checks/flags/notes stay attached to the right thing.
  - Keep the `<div class="connect-bar">` block and the whole `<script>` block verbatim — don't regress to localStorage.

## Data Sources And How To Use Them
You receive already-fetched data snapshots for each source. Your job is to decide:
a) which items are actually actionable for Janine now, and
b) how to present them compactly in the HTML, respecting state.

### 1. Slack

Look back 24–48h. Only surface items that are genuine open questions, decisions, or asks where my response is needed. For each candidate, read the full thread and skip it if I already replied and closed it.

Run all steps each time (no shortcuts):
- 1. **Priority channels**: For each channel in my priority list, read messages in the window and any active reply threads. Surface only open asks for me.
- 2. **Floor-list DMs**: For each person in my collaborators list, treat their user ID as the DM channel, read messages in the window; for any message with replies, pull the thread. Surface open asks directed to me.
- 3. **Other DMs / group DMs**: Search DMs and multi-person DMs (no name filter) in the window; surface open asks where I’m clearly the intended responder.
- 4. **My own threads**: Search messages from me in the window; if there’s ongoing discussion and a decision or answer is pending from me, surface it.
- 5. **Resolution check**: Before including any item, confirm I haven’t already resolved it in the thread.

### 2. Jira

- **Assigned to me**: JQL `assignee = currentUser() AND statusCategory != "Done" ORDER BY updated DESC`. Flag anything with new comments since yesterday.
- **Reported by me, active**: `reporter = currentUser() AND updated >= -7d`
- **Reported by me, stale**: `reporter = currentUser() AND statusCategory != "Done" AND updated <= -14d ORDER BY updated ASC`. This list can be large — show at most 10, oldest first, and state the total count if more exist, rather than silently truncating.
- **Service Desk-style tickets**, if you track those as their own project (see Personalization): add `AND project = "ProjectCode"` to any of the above, or run it as its own query — same shape either way.

### 3. Confluence

`searchConfluenceUsingCql`, `mention = currentUser() AND type = page`, no recency limit. Read action items straight from the summary snippet — most hits are meeting notes with plain-text "[full name] to X" lines (see Personalization). Only fetch the full page when a snippet is ambiguous about whether it's still open. Don't fetch every hit in full — most are just attendee mentions, not real assignments, and that's what makes this step slow.

### 4. Email

Search `in:inbox after:<24-48h>`, broad — not narrowed to `to:me`. Skip your noise senders (see Personalization) as a block once recognized — don't enumerate them. Surface: real asks/decisions, action-item mentions, "items requiring attention" summary, anything tied to a thread already tracked elsewhere (cross-reference, don't duplicate).

### 5. Calendar

Pull today and tomorrow. Cross-reference each meeting against sources 1-4 and note what to prepare. Add a checkbox to each entry so it can be cleared from the list once it's done.

### 6. My Notes

Read `myNotes` from the state file — never invent or reword the text, source, or a sub-note. Shape per note:
```
{"id": "mynote-...", "text": "...", "dateAdded": "YYYY-MM-DD",
 "subNotes": [{"text": "...", "date": "YYYY-MM-DD"}]}
```
`dateAdded` stamps once at creation. Each sub-note gets its own date when appended — an append-only dated log, not one overwritable field. All editing happens live in the page; never edit `myNotes` by hand, and never remove an entry unless I ask.

### 7. Carry-forward

If `daily-briefing.html` already exists, read it first. Re-check each flagged item's status per the Output Rules above, then carry forward whatever's still genuinely open, tagged "Carried over".

## Assembly

Write a single self-contained HTML file with checkboxes, grouped by source, each item linking out where a URL exists.

- Checking a box removes the item entirely (not a strikethrough) — done means gone. A card with nothing left shows "All cleared."
- Trust `daily-briefing-state.json` over anything else — it's the same file the browser writes to. If I report a mismatch, check the file, not the browser.

## Style
- Aim for clarity and brevity. Each item should answer:
  - “What is this?”,
  - “Why should the user care now?”,
  - “What’s the next step?”
