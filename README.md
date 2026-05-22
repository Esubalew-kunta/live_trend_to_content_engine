# Live Trend-to-Content Engine

An n8n automation workflow that monitors Google News for trending topics, detects signal spikes, generates a publish-ready social media post using free AI, and delivers it to your team's Discord channel — fully automated, every 30 minutes, at zero cost.

---

## The Problem It Solves

Marketing teams lose the trend window. By the time someone spots a headline, writes a caption, and gets approval — the moment is 4–6 hours old. The audience has moved on.

This workflow closes that gap:

| Before | After |
|--------|-------|
| Manual news monitoring, 1–2x per day | Automated monitoring every 30 minutes, 24/7 |
| Caption written from scratch each time | AI-generated post ready on arrival |
| 20–40 minutes from trend to draft | Under 2 minutes from trend to review |
| Missed windows during off-hours | Always-on — nights, weekends, holidays |

---

## How It Works

```
[Every 30 min]
      │
      ▼
[Google News RSS] ──► [XML Parser] ──► [Filter: Last 2h]
                                              │
                              ┌───────────────┴───────────────┐
                           ≥3 articles                    < 3 articles
                              │                               │
                              ▼                             (stop)
                  [Groq AI: llama-3.3-70b-versatile]
                              │
                              ▼
                    [Build Discord Embed]
                              │
                              ▼
                    [Discord Webhook 🟢]
```

### Node Breakdown

| # | Node | What It Does |
|---|------|-------------|
| 1 | **Schedule Trigger** | Fires every 30 minutes, 24/7 |
| 2 | **HTTP Request** | Fetches Google News RSS for your keyword — no API key required |
| 3 | **XML Parser** | Converts raw RSS/XML into structured JSON |
| 4 | **Code: Filter Recent Articles** | Keeps only articles from the last 2 hours, normalizes the feed structure, builds the Groq prompt |
| 5 | **IF: 3+ Articles Trending?** | Signal gate — only proceeds when a real trend is detected |
| 6 | **HTTP Request: Groq AI** | Sends headlines to `llama-3.3-70b-versatile` → returns a 3-sentence post with hashtags |
| 7 | **Code: Build Discord Payload** | Formats the output into a rich Discord embed with metadata |
| 8 | **HTTP Request: Discord** | Delivers the embed to your channel via webhook |

---

## Prerequisites

Everything used here is **100% free** — no credit card required.

| Service | What You Need | Where to Get It |
|---------|--------------|-----------------|
| **n8n** | Self-hosted or n8n Cloud free tier | [n8n.io](https://n8n.io) |
| **Groq** | Free API key | [console.groq.com](https://console.groq.com) |
| **Discord** | Free server + webhook URL | [discord.com](https://discord.com) |
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
Then open `http://localhost:5678`.

**npm**
```bash
npm install n8n -g && n8n start
```

**n8n Cloud** — sign up free at [app.n8n.cloud](https://app.n8n.cloud), no install needed.

---

### 2. Get a Groq API Key

1. Go to [console.groq.com](https://console.groq.com) and sign up (no credit card)
2. Navigate to **API Keys → Create API Key**
3. Copy the key — it starts with `gsk_`

---

### 3. Create a Discord Webhook

1. Open your Discord server → right-click a channel → **Edit Channel**
2. Go to **Integrations → Webhooks → New Webhook**
3. Name it (e.g. `Trend Engine`) and click **Copy Webhook URL**

---

### 4. Import the Workflow

1. In n8n, go to **Workflows → Import from File** (or `Ctrl+I`)
2. Select `workflow.json` from this repo
3. All 8 nodes load pre-connected

---

### 5. Add Your Credentials

You only need to update **2 values**:

**Groq API Key** — in the `Generate Post via Groq AI` node:
- Go to **Headers → Authorization**
- Replace `PASTE_YOUR_GROQ_API_KEY_HERE` with `Bearer gsk_YOURKEYHERE`

**Discord Webhook URL** — in the `Post to Discord` node:
- Replace `PASTE_YOUR_DISCORD_WEBHOOK_URL_HERE` with your full webhook URL

---

### 6. Test It

1. Click **Execute Workflow** in the top bar
2. Watch each node light up green
3. Check Discord for the embed

> If the IF node routes to the false branch (fewer than 3 recent articles), temporarily set the threshold to `1` to test the full flow end-to-end, then reset it to `3`.

---

### 7. Activate

Toggle **Inactive → Active** in the top-right. The workflow now runs on its own every 30 minutes.

---

## Customization

**Change the keyword**

Edit the URL in the `Fetch Google News RSS` node:
```
https://news.google.com/rss/search?q=YOUR+KEYWORD+HERE&hl=en-US&gl=US&ceid=US:en
```
Examples: `generative+AI`, `SaaS+funding`, `electric+vehicles`

**Change the trend threshold**

In the `3+ Articles Trending?` IF node, change `3`:
- Lower = more posts (catch smaller signals)
- Higher = only major spikes

**Change the time window**

In the `Filter Recent Articles` Code node, edit `2 * 60 * 60 * 1000`:
- 1 hour: `1 * 60 * 60 * 1000`
- 4 hours: `4 * 60 * 60 * 1000`

**Change the AI prompt**

In the `Filter Recent Articles` Code node, edit the `groqPayload.messages` array. Adjust tone, format, length, or persona to match your brand voice.

**Change the post frequency**

In the `Every 30 Minutes` Schedule Trigger node, update `minutesInterval`.

---

## Design Decisions

**Why Groq instead of OpenAI?**
Groq requires no credit card, offers a generous free tier, and returns responses in under 2 seconds. For short social copy, `llama-3.3-70b-versatile` produces sharp results at zero cost.

**Why build the payload in the Filter node?**
The headline string is constructed where the data is clean. Passing it pre-built to the Groq node avoids JSON injection issues if article titles contain quotes or special characters — `JSON.stringify()` handles final serialization.

**Why a separate Discord payload Code node?**
Discord's embed format is structured JSON. Building it explicitly in code keeps the structure readable and easy to modify — far cleaner than using the HTTP node's template editor for a nested object.

**Why the IF gate?**
Without it, Groq would fire on every run regardless of signal. The gate ensures the AI only activates when there's a genuine trend — no noise, no wasted API calls.

---

## Extending This Workflow

Some natural next steps:

- **Multi-channel delivery** — post directly to LinkedIn or Twitter/X instead of (or alongside) Discord
- **Human-in-the-loop approval** — add a step that waits for a team member to click "Approve" before the post goes live
- **Competitor monitoring** — duplicate the workflow with a second keyword tracking a competitor's brand or product category
- **Slack delivery** — swap the Discord webhook for a Slack incoming webhook with no other changes needed
- **Sentiment filter** — add a step that skips negative news cycles you don't want to amplify

---

## License

MIT — use it, fork it, build on it.
