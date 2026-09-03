# Meet To Action — AI Agent

An autonomous AI agent that turns meeting summaries into tracked work in Jira — automatically. Built with **n8n**, **Google Gemini**, and the **Jira** and **Telegram** APIs, self-hosted via **Docker**.

## What it does

1. **Watches Gmail** for new meeting summary emails (from Google Meet's auto-generated meeting notes).
2. **Cleans and extracts** the email body into plain text.
3. **Pulls the current open Jira tickets** in the project for context.
4. **Uses Google Gemini to classify each action item** in the summary: is this brand-new work, or does it belong to a ticket that already exists?
5. **Branches automatically**:
   - **New work** → looks up the assignee's Jira account ID → creates a new Jira issue.
   - **Existing work** → posts a comment directly to the matching issue via the Jira REST API.
6. **Merges both branches** back into one stream and **sends a Telegram notification** once everything's processed.

No manual copy-pasting meeting notes into a tracker. No duplicate tickets. The agent reads the room and decides what to do.

## Why this is more than a chatbot

Most "AI agent" demos just answer questions. This one has **real write access to production tools** — it creates and comments on actual Jira issues, and it reasons about existing tickets before deciding what to do, rather than blindly generating a new item every run. That's the difference between an agent and a wrapper around a chat model.

## Architecture

```
Gmail Trigger (poll every 5 min, filtered to meeting-notes sender)
        ↓
Code — clean HTML email body into plain text
        ↓
Jira — Get many issues (project = MTA), for context
        ↓
AI Agent (Google Gemini) — classify each action item as new / existing
        ↓
Code — parse the agent's JSON output into individual items
        ↓
Code — map each item's Jira issue key to reference data
        ↓
IF — route on type
   ┌────────────┴────────────┐
   ↓ new                     ↓ existing
Code — resolve               HTTP Request (POST)
assignee's Jira accountId    → Jira REST API: add comment
   ↓                         directly to the matching issue
Jira — Create an issue           ↓
   └────────────┬────────────┘
                ↓
             Merge
                ↓
     Telegram — send notification
```

## Tech stack

- **n8n** (self-hosted via Docker) — workflow orchestration
- **Google Gemini API** (`@n8n/n8n-nodes-langchain`) — LLM reasoning and classification
- **Jira Cloud REST API** — issue creation (via n8n's Jira node) and comments (via direct HTTP request)
- **Telegram Bot API** — notifications
- **Gmail API (OAuth2)** — trigger on incoming meeting-summary emails

## What's in this repo

- `Meet To Action - AI Agent.json` — the exported n8n workflow, ready to import

## Setup

1. Run n8n via Docker (see [n8n docs](https://docs.n8n.io/hosting/installation/docker/)).
2. Import the workflow JSON into your n8n instance.
3. Connect your own credentials: Gmail (OAuth2), Jira Cloud (API token), Google Gemini (API key), Telegram (bot token).
4. Update the hardcoded values to match your own environment:
   - Jira project ID and issue-type ID in the **Create an issue** node
   - The `keyToId` / assignee mapping in the **Code** nodes
   - Your Jira site domain in the **HTTP Request** node's URL
   - Your Telegram chat ID
5. Activate the workflow.

## Notes on building this

Jira's REST API turned out to need numeric IDs rather than human-readable keys for some fields (project, issue type) but accepted the plain issue key directly in the URL path for comments — inconsistencies like that were most of the actual debugging work. The "add comment" step ended up as a direct HTTP call using Jira's Atlassian Document Format for the comment body, since the built-in node's comment operation returned 404s that the project-create operation didn't. That kind of API-integration troubleshooting is a good chunk of what real agent engineering looks like in practice — the LLM reasoning was the easy part; getting two independent third-party APIs to actually agree on data shapes was not.

---

*Built as a hands-on project to explore agentic AI workflows — LLM reasoning connected to real external tools with actual side effects, not just chat.*
