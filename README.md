# 🎬 AI Content Automation — Two-Tier n8n Pipeline

> Schedule a topic. Get a fully produced video, captions, hashtags, and a LinkedIn post — reviewed via Slack, published to four platforms. Zero manual editing.

---

## How It Works

This system runs across two n8n workflows that communicate through Google Sheets and HTTP.

**Workflow 1 (Content Automation)** runs on a schedule and handles generation, rendering, and publishing.

**Workflow 2 (P2 — Slack Listener)** runs 24/7 and handles human approval by listening for a single Slack message.

---

## Architecture

![Content Automation Architecture](./architecture.png)

---

## Workflow 1 — Content Automation

### Trigger
Cron schedule: `0 9 * * 1,3,5` — every Monday, Wednesday, Friday at 9am.

### Node Flow

| Node | Type | What It Does |
|---|---|---|
| `Schedule Trigger` | Schedule | Fires Mon/Wed/Fri at 9am |
| `Get Next Topic from Google Sheets` | Google Sheets | Reads first row where `Status = Pending` |
| `Mark Topic as In Progress` | Google Sheets | Updates status + logs `Started_At` timestamp |
| `OpenAI — Generate All Content` | HTTP (OpenAI API) | GPT-4o generates full content package as JSON |
| `Parse OpenAI Response` | Code (Python) | Strips any markdown, parses to clean JSON |
| `HeyGen — Generate Avatar Video` | HTTP | Submits script to HeyGen avatar render |
| `Wait for HeyGen Render` | Wait | Pauses while HeyGen processes |
| `Poll HeyGen Video Status` | HTTP | Checks render status, returns video URL |
| `Creatomate — Render 9:16 Vertical` | HTTP | Renders TikTok/Reels/Shorts format |
| `Creatomate — Render 16:9 Horizontal` | HTTP | Renders YouTube/LinkedIn format |
| `Wait for Creatomate Renders` | Wait | Waits for both Creatomate jobs |
| `Send Approval to Slack` | Slack | Posts video links + LinkedIn preview to channel |
| `Wait for Human Approval` | Wait (webhook) | Pauses execution — resumes when P2 calls back |
| `Approved?` | IF | Routes on `decision = approve` |
| `Post to TikTok/Instagram/YouTube/LinkedIn` | HTTP | Publishes to all platforms in parallel |
| `Mark Topic as Published` | Google Sheets | Updates status + logs `Published_At` |
| `Mark Topic as Rejected` | Google Sheets | Updates status + logs `Rejected_At` |

### GPT-4o Content Package

One API call returns a structured JSON with everything:

```json
{
  "video_script": {
    "hook": "opening 5-second attention grabber",
    "body": "90–120 second spoken script",
    "cta": "closing call-to-action"
  },
  "captions": {
    "tiktok": "caption under 150 chars",
    "instagram": "caption with emoji 100–200 chars",
    "youtube": "keyword-rich 200–300 char description"
  },
  "hashtags": {
    "tiktok": ["tag1", "tag2", "tag3", "tag4", "tag5"],
    "instagram": ["tag1", "...10 tags total"],
    "youtube": ["tag1", "tag2", "tag3"]
  },
  "linkedin_post": {
    "headline": "attention-grabbing first line",
    "body": "150–200 word professional post",
    "cta": "engagement question"
  },
  "metadata": {
    "topic": "topic string",
    "generated_at": "ISO timestamp",
    "suggested_thumbnail_text": "max 6 words"
  }
}
```

---

## Workflow 2 — Slack Approval Listener (P2)

Always active. Listens on the `#all-automation-test` Slack channel for `approve` or `reject`.

| Node | Type | What It Does |
|---|---|---|
| `Slack Trigger — Listen for Approval` | Slack Trigger | Fires on every message in the channel |
| `Is approve or reject?` | IF | Filters out irrelevant messages |
| `Get Resume URL from Sheet` | Google Sheets | Reads `Resume_URL` from the In Progress row |
| `Set Decision and URL` | Set | Normalizes decision text + resume URL |
| `Resume Main Workflow` | HTTP Request | POSTs to Workflow 1's wait node webhook URL |
| `Send Confirmation to Slack` | Slack | Confirms decision received in channel |

### How the two workflows connect

When Workflow 1 reaches the `Wait for Human Approval` node, it writes its webhook resume URL to Google Sheets. P2 reads this URL and calls it with `?decision=approve` or `?decision=reject`. Workflow 1 wakes up, reads the query param, and routes accordingly.

---

## Google Sheets Schema

| Column | Description |
|---|---|
| `Topic` | Content topic (match key across all updates) |
| `Status` | `Pending` → `In Progress` → `Published` / `Rejected` |
| `Started_At` | ISO timestamp when workflow picked it up |
| `Resume_URL` | Webhook URL written by W1, read by P2 |
| `Published_At` | ISO timestamp on successful publish |
| `Rejected_At` | ISO timestamp on rejection |
| `Video_9x16_URL` | Creatomate vertical render link |
| `Video_16x9_URL` | Creatomate horizontal render link |

---

## Setup Requirements

- n8n instance (self-hosted or cloud)
- OpenAI API key (GPT-4o access)
- HeyGen API key + avatar ID
- Creatomate API key + template IDs (9:16 and 16:9)
- Slack app with bot token + channel access
- Google Sheets with OAuth2 credential
- TikTok / Instagram / YouTube / LinkedIn API credentials

---

## Request Access

The full workflow JSON files are available on request — not published here to prevent credential exposure and unsupported production cloning.

Open a [GitHub Issue](../../issues) titled **"Workflow Request"** or reach out via [LinkedIn](https://linkedin.com/in/yourprofile).

---

## Tech Stack

- **n8n** — workflow automation engine
- **OpenAI GPT-4o** — content generation
- **HeyGen** — AI avatar video synthesis
- **Creatomate** — programmatic video rendering
- **Slack** — human-in-the-loop approval interface
- **Google Sheets** — content calendar and state store
- **TikTok · Instagram · YouTube · LinkedIn** — publishing targets
