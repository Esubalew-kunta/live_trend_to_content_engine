# Live Trend-to-Content Engine

An n8n automation workflow that monitors **4 marketing keywords in parallel**, scores each trend with AI, and — only when the signal is strong enough — generates platform-ready posts for **Twitter, LinkedIn, and TikTok**, delivered instantly to Discord and Slack. Fully automated, every 30 minutes, at zero cost.

---

## Video Demo

> **[▶ Watch the demo](#)** ← link coming soon

---

## The Problem It Solves

Marketing teams lose the trend window. By the time someone spots a headline, writes a caption for each platform, and gets approval — the moment is 4–6 hours old.

This workflow closes that gap with a two-stage AI pipeline: it first *scores* whether a trend is worth acting on, then *writes* the posts — so your team only sees content that clears the bar.

| Before | After |
|--------|-------|
| Manual news monitoring, 1–2× per day | 4 keywords monitored every 30 min, 24/7 |
| One generic caption written from scratch | Platform-tailored posts for Twitter, LinkedIn & TikTok |
| 30–60 min from trend to draft | Under 2 min from trend to review |
| No signal filtering — everything looks urgent | AI trend score gates the pipeline — only score ≥ 7/10 proceeds |
| Missed windows during off-hours | Always-on — nights, weekends, holidays |

---

## Architecture

```
[Every 30 Minutes]
        │
        ▼
┌───────────────────────────────────────────┐
│  Parallel RSS Fetch (4 keywords)          │
│  • AI marketing      • Content AI         │
│  • Brand strategy    • Marketing automation│
└───────────────────────────────────────────┘
        │
        ▼
[Merge All Feeds] ──► [Filter: Last 24h] ──► [Dedup]
                                                  │
                                    ┌─────────────┴──────────────┐
                                 ≥ 3 articles               < 3 articles
                                    │                            │
                                    ▼                      [Log: Skipped]
                          [Groq: Score Trend]
                          llama-3.3-70b-versatile
                          Returns score 1–10
                                    │
                          ┌─────────┴──────────┐
                       Score ≥ 7           Score < 7
                          │                    │
                          ▼              [Log: Low Score]
                 [Groq: Generate Content]
                 Twitter · LinkedIn · TikTok
                          │
                          ▼
              ┌───────────────────────┐
              │  Discord Embed        │
              │  Slack Notification   │
              │  Airtable Run Log     │
              └───────────────────────┘
```

---

## Node Breakdown

| # | Node | What It Does |
|---|------|-------------|
| 1 | **Every 30 Minutes** | Schedule trigger — fires 24/7 |
| 2–5 | **Fetch Google News RSS ×4** | Parallel RSS fetches — one per keyword, no API key needed |
| 6–9 | **Edit Fields ×4** | Tags each item with its source keyword |
| 10 | **Merge All Feeds** | Combines all 4 keyword feeds into one stream |
| 11 | **Filter Recent Articles** | Keeps only articles from the last 24 hours |
| 12 | **Airtable Dedup** | Removes duplicate articles across keyword feeds |
| 13 | **3+ New Articles?** | Signal gate — stops the run if fewer than 3 articles found |
| 14 | **Prepare Scoring Input** | Formats headline list for the scoring prompt |
| 15 | **Build Groq Score Payload** | Builds a structured prompt asking Groq to rate trend relevance 1–10 |
| 16 | **Groq — Score Trend** | First AI call — returns `{ score, reasoning }` as JSON |
| 17 | **Parse Score** | Extracts the numeric score |
| 18 | **Score 7 or Above?** | Quality gate — only high-signal trends proceed to content generation |
| 19 | **Build Content Payload** | Builds a prompt with score + reasoning for content generation |
| 20 | **Generate Content via Groq** | Second AI call — returns Twitter, LinkedIn, and TikTok posts |
| 21 | **Parse Content** | Extracts the three platform posts |
| 22 | **Build Discord Payload** | Formats posts into a rich Discord embed |
| 23 | **Post to Discord** | Delivers the embed via webhook |
| 24 | **Slack Notification** | Sends a summary alert to Slack |
| 25 | **Log: Success / Skipped** | Writes every run outcome to Airtable for tracking |

---

## Why Two AI Calls?

Most trend-to-content tools generate copy for every signal, flooding your team with noise. This pipeline separates **signal detection** from **content creation**:

1. **Score Trend** — fast, cheap call asking only "is this worth writing about?" (score 1–10)
2. **Generate Content** — only fires when score ≥ 7, so the AI writes *less* but every post is worth reading

The result: your team gets a Discord/Slack ping only when the trend actually clears the bar.

---

## Prerequisites

Everything here is **free** — no credit card required for any service.

| Service | What You Need | Where to Get It |
|---------|--------------|-----------------|
| **n8n** | Self-hosted or Cloud free tier | [n8n.io](https://n8n.io) |
| **Groq** | Free API key | [console.groq.com](https://console.groq.com) |
| **Discord** | Server + webhook URL | [discord.com](https://discord.com) |
| **Slack** | Workspace + incoming webhook | [api.slack.com/messaging/webhooks](https://api.slack.com/messaging/webhooks) |
| **Airtable** | Free base + API key | [airtable.com](https://airtable.com) |
| **Google News RSS** | Nothing — no key needed | Built into the workflow |

---

## Setup

### 1. Run n8n

**Docker (recommended)**
```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```
Open `http://localhost:5678`.

**npm**
```bash
npm install n8n -g && n8n start
```

**n8n Cloud** — sign up free at [app.n8n.cloud](https://app.n8n.cloud), no install needed.

---

### 2. Get a Groq API Key

1. Go to [console.groq.com](https://console.groq.com) and sign up (no credit card)
2. **API Keys → Create API Key**
3. Copy the key — it starts with `gsk_`

---

### 3. Create a Discord Webhook

1. Right-click a channel → **Edit Channel → Integrations → Webhooks → New Webhook**
2. Name it (e.g. `Trend Engine`) and click **Copy Webhook URL**

---

### 4. Create a Slack Incoming Webhook

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App → From scratch**
2. Under **Features**, select **Incoming Webhooks → Activate**
3. Click **Add New Webhook to Workspace**, pick a channel, and copy the webhook URL

---

### 5. Set Up Airtable

1. Create a free base at [airtable.com](https://airtable.com)
2. Add a table named **RunLogs** with these fields:

| Field | Type |
|-------|------|
| RunTime | Date/time |
| KeywordsMonitored | Number |
| ArticlesFound | Number |
| TrendScore | Number |
| ContentGenerated | Single line text |
| Status | Single line text |

3. Go to [airtable.com/create/tokens](https://airtable.com/create/tokens) → create a token with **data.records:write** scope for your base
4. Copy your **Base ID** from the URL: `airtable.com/appXXXXXXXXXXXXXX/...`

---

### 6. Import the Workflow

1. In n8n: **Workflows → Import from File** (or `Ctrl+I`)
2. Select `Live Trend-to-Content Engine.json` from this repo
3. All 25 nodes load pre-connected

---

### 7. Add Your Credentials

Update these values throughout the workflow:

| Where | What to Replace |
|-------|----------------|
| `Generate Content via Groq` node → Authorization header | `Bearer YOUR_GROQ_API_KEY` |
| `Groq - Score Trend` node → Credential | Create an **HTTP Header Auth** credential with `Authorization: Bearer YOUR_GROQ_API_KEY` |
| `Post to Discord` node → URL | Your Discord webhook URL |
| `Slack Notification` node → URL | Your Slack incoming webhook URL |
| `Log: Success`, `Log: Low Score`, `Log: Low Article Count` nodes | Your Airtable token + base ID |

---

### 8. Test It

1. Click **Execute Workflow** in the top bar
2. Watch the 4 RSS fetches fire in parallel, merge, and flow through the pipeline
3. If the `3+ New Articles?` gate routes false (fewer than 3 articles), temporarily set the threshold to `1` in the IF node to test the full flow, then reset it to `3`
4. Check Discord and Slack for the embed

---

### 9. Activate

Toggle **Inactive → Active** top-right. The workflow now runs autonomously every 30 minutes.

---

## Customization

**Change or add keywords**

Each keyword has its own RSS fetch + label node. To add a fifth:
1. Duplicate one `Fetch Google News RSS` node and update the URL:
   ```
   https://news.google.com/rss/search?q=YOUR+KEYWORD&hl=en-US&gl=US&ceid=US:en
   ```
2. Duplicate the paired `Edit Fields` node and set `keyword` to your new term
3. Update the `Merge All Feeds` node to accept 5 inputs instead of 4

**Change the article threshold**

In the `3+ New Articles?` IF node, change the value `3`:
- Lower = catch smaller signals
- Higher = only major spikes

**Change the trend score cutoff**

In the `Score 7 or Above?` IF node, change `7`:
- `5` — more content, lower bar
- `9` — only exceptional trends

**Change the AI model**

Both Groq nodes use `llama-3.3-70b-versatile`. Swap to any Groq-supported model (e.g. `mixtral-8x7b-32768`) in the respective Code or HTTP nodes.

**Change the post frequency**

In the `Every 30 Minutes` Schedule Trigger, update `minutesInterval`.

**Adjust the time window**

In `Filter Recent Articles`, edit `24 * 60 * 60 * 1000`:
- 2 hours: `2 * 60 * 60 * 1000`
- 48 hours: `48 * 60 * 60 * 1000`

---

## Design Decisions

**Why parallel RSS fetches instead of one?**
A single keyword misses adjacent signals. Running 4 in parallel — AI marketing, Content AI, Brand strategy, Marketing automation — means the trend score reflects the real momentum across a topic cluster, not just one phrase.

**Why two AI calls instead of one?**
Scoring and writing are different cognitive tasks. A short, focused scoring prompt at `temperature: 0.3` gives reliable 1–10 output. A separate writing prompt at `temperature: 0.7` gives creative, platform-appropriate copy. Combining them in one prompt produces worse results on both dimensions.

**Why Groq instead of OpenAI?**
No credit card, generous free tier, and sub-2-second responses on `llama-3.3-70b-versatile`. For trend scoring and short-form copy, it performs on par with GPT-3.5 at zero cost.

**Why Airtable for logging instead of n8n's built-in execution log?**
n8n's execution history is hard to query and gets pruned. Airtable gives you a filterable, shareable run log — useful for showing stakeholders how often the engine fires and what it skips.

---

## Extending This Workflow

- **Human-in-the-loop approval** — insert a step that sends a draft to Slack and waits for a reaction before posting publicly
- **Direct social posting** — swap Discord/Slack for the Twitter API or LinkedIn API to publish automatically
- **Competitor monitoring** — add a second set of 4 keywords tracking a competitor's product or brand name
- **Sentiment filter** — add a pre-content step that skips negative or crisis-adjacent news cycles
- **Email digest** — batch the day's top-scoring trends into a morning briefing via SendGrid or Gmail

---

## License

MIT — use it, fork it, build on it.
