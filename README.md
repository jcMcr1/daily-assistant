# Daily Briefing — README

A self-contained daily briefing that pulls from Slack, Jira/Confluence, email, and calendar into one local HTML file, with checkboxes, flags, and notes that persist across regenerations. No server, no database, no cloud storage — everything lives in this folder.

## Files in this folder

| File | Purpose |
|---|---|
| `template.html` | A blank starting point — just the shell (styling, the connect-bar, the persistence script), no content. Copy this to `daily-briefing.html` before your first run. |
| `daily-briefing.md` | The instructions your assistant reads before generating a briefing. This is the file to edit if you want to change *how* the briefing is built. Contains no personal details — safe to share or commit. |
| `personalisation.md` | Your actual personal details (name, Slack channels/collaborators, Atlassian instance URLs, noise senders) that `daily-briefing.md` refers back to. Gitignored, so these details never end up in git history or get shared alongside the spec. Not in this repo — your assistant creates it on first run by asking you for each field. |
| `daily-briefing.html` | The generated output. Overwritten each time you ask for a fresh briefing. Open this in your browser. Not in this repo — it's yours once you copy it from `template.html`. |
| `daily-briefing-state.json` | Your checkboxes, flags, per-item notes, and "My Notes" list. The HTML reads/writes this directly — it's the source of truth, not the browser's memory. Also not in this repo — it's created the first time you click "Connect notes file." |

## Requirements

- **Chrome or Edge.** The state file is read/written via the File System Access API, which Safari and Firefox don't support.
- **MCP connectors** for whichever data sources you actually use — Slack, Atlassian (Jira/Confluence), Gmail, Calendar. If you don't use one of these, delete its section from `daily-briefing.md` (see below) rather than leaving a source that will just fail or return nothing.

## Setting up the connectors

This runs on Claude's built-in connectors — no server or custom MCP setup, just OAuth-connecting each service once.

**Where to add them**
- **Claude.ai (web/desktop)**: Settings → Connectors → browse and connect.
- **Claude Code**: run `/mcp` in a session, or use the connector picker in settings.

(Exact menu wording shifts as Anthropic updates the product — if you can't find it, search your client's settings for "Connectors" or "Integrations".)

**Per data source**
- **Slack** — connect it, authorize against your workspace. Gives tools like `slack_search_public_and_private` and `slack_read_channel` that Source 1 relies on.
- **Jira/Confluence (Atlassian)** — connect it, authorize against your Atlassian site. If you use two separate Atlassian clouds, add the Atlassian connector twice — once per org — since each OAuth grant is scoped to a single cloud.
- **Gmail** — connect it, authorize against your Google account. Gives `search_threads` and similar for Source 4.
- **Calendar** — usually the same Google connector as Gmail, or a separate one depending on your client version. Gives `list_events` for Source 5.

**Checking it worked**: ask your assistant something like "what MCP tools do you have for Slack?" — a live connector lists real tool names; a missing one, it'll say so.

If a tool name in `daily-briefing.md` doesn't match what your connector actually exposes, just ask your assistant to update the doc to match — the instructions are meant to track your real toolset, not the other way round.

## Quick start

Your personal details live in `personalisation.md`, not `daily-briefing.md` — kept separate and gitignored so the spec stays shareable without leaking your Slack channels, Jira instance, etc. The first time you ask for a briefing and `personalisation.md` doesn't exist yet, your assistant will ask you for each of these fields and create the file for you:

- Your full name (used to spot "[Name] to do X" lines in meeting notes)
- Your Slack floor list — the people whose DMs get checked explicitly rather than relying on search
- Your Slack priority channels - the channels that get checked every time rather than relying on search
- Your Jira/Confluence cloud — most people have one; if your org splits Service Desk/Confluence into a genuinely separate Atlassian org, there's a note on what that costs you in setup
- Your email noise senders — whatever automated mail you want silently ignored

You can also create or edit `personalisation.md` yourself at any time instead of going through the prompts.

Beyond that block, a few things to check per data source in `daily-briefing.md`:

- **Any source you don't use** (no Slack, no Jira, etc.) — delete that numbered section under Data Sources entirely, and drop its output from the HTML template rather than leaving an empty/broken card.
- **Source 2's JQL** assumes a specific status field (`statusCategory`) and a placeholder project key — these vary by Jira instance, so check them against your own before relying on them.
- **Two Atlassian clouds**: if Confluence/Service Desk lives on a genuinely separate Atlassian org from your main Jira, you'll need a second Atlassian connector — each OAuth grant is scoped to a single cloud. Most people don't need this.

The HTML/JS itself (checkbox persistence, flagging, per-item notes, My Notes) is generic and shouldn't need changes — it's driven entirely by the JSON state file and the `data-id`/`data-section-key` attributes the instructions tell your assistant to generate consistently.

1. After you have added the files to the working folder, ask your assistant to "run today's briefing from daily-briefing.md" — it reads `daily-briefing.md` and fills in `daily-briefing.html`.
  **Note** Claude has its own morning briefing which connects to Slack & Gmail and stores as an Artifact. If you find it running this, correct Claude and tell it you want to run daily-briefing.md from the working folder. It should remember then
2. Open `daily-briefing.html` in Chrome or Edge.
3. Click **Connect notes file** and pick (or create) `daily-briefing-state.json` in this folder. Do this once per browser.
4. Check things off, star anything urgent, add notes. It all saves back to the JSON file automatically.
5. Next time, just ask for the briefing again — it reads your current state first and won't re-surface anything you've already checked off.


## Ongoing use

Treat `daily-briefing.md` as the spec and `daily-briefing-bugs.md` as the changelog. When the briefing misses something or gets a query wrong, that's a bugs.md entry with a root-cause fix folded back into the spec — not a one-off patch to the HTML.
