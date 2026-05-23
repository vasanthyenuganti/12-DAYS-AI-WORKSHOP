# Day 4 — Lab 4B: n8n Daily News Digest (Turnkey Walkthrough)

**Time:** 90 minutes (14:00 — 15:30)
**Format:** Individual at laptop; trainer demos node-by-node first
**Goal:** Each mentor has a self-hosted n8n flow that runs every morning at 7AM and emails them a 5-bullet placement-news digest. Workflow JSON exported and pushed.

---

## Setup (5 min)

Each mentor has:
- Docker installed and running on their laptop (Docker Desktop on Mac/Win, or `docker compose` CLI on Linux)
- Their Gemini API key (from Day 1, in 1Password)
- The personal Gmail account they want to send the digest to
- `configs/n8n_docker-compose.yml` from the lab kit copied to their working folder

**Trainer pre-condition:** verified during T-1 week prep that docker-compose works on the standard mentor laptop spec (4GB free RAM minimum).

---

## Step 1 — Run n8n locally (10 min)

In terminal:

```bash
cd <wherever you put n8n_docker-compose.yml>
docker compose -f n8n_docker-compose.yml up -d
```

Wait 30-60 seconds for the container to start.

Open browser to **http://localhost:5678**.

First-time setup:
- Username: `admin`
- Password: `changeme` (we'll change this)
- Click "Set up account" — create owner account with your email

**Acceptance:** n8n web UI loads at localhost:5678.

---

## Step 2 — Create the workflow (5 min)

In n8n UI:
- Click **+ New Workflow** (top-left)
- Name it `Daily Placement News Digest`
- Save (Cmd/Ctrl-S)

You'll see an empty canvas with one start node. Now we add 4 nodes: Schedule → RSS → HTTP (Gemini) → Gmail.

---

## Step 3 — Add RSS Read node (10 min)

Click the **+** to add a node. Search for "RSS Read".

Configure:
- **URL** (parameter): paste **one** of these RSS sources to start:
https://news.ycombinator.com/rss 
  - `https://feeds.feedburner.com/nasscom-newsroom` (NASSCOM)
  - `https://feeds.feedburner.com/TechCrunch/IndianStartups` (TechCrunch India)
  - `https://www.naukri.com/blog/feed/` (Naukri career blog)
- **Limit**: 10 (max items per fetch)

Click **Test step**. Should return ~10 RSS items in the right panel.

**Trainer note:** if a feed URL is dead (RSS feeds change), substitute another news source. Common reliable: `https://feeds.feedburner.com/feedburner/nasscom-news`, or any RSS from Inc42, YourStory.

**Acceptance:** RSS node returns ≥3 news items on test.

---

## Step 4 — Add HTTP Request node to Gemini (15 min)

Click **+** after the RSS node. Search for "HTTP Request".

Configure:
- **Method**: POST
- **URL**: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- **Authentication**: Generic Credential Type → Header Auth
  - Name: `x-goog-api-key`
  - Value: paste your Gemini key directly (or use n8n credentials store — see note below)
- **Body Content Type**: JSON
- **Specify Body**: Using JSON
- **JSON**:

```json
{
  "contents": [
    {
      "parts": [
        {
          "text": "Summarise these RSS items into 5 bullets for a placement coach. Each bullet ≤ 20 words. Highlight any item signalling new hiring, layoffs, or significant tech demand changes. Items follow:\n\n{{ $json[\"items\"].map(i => `- ${i.title}: ${i.contentSnippet || i.content || ''}`).join('\\n') }}"
        }
      ]
    }
  ]
}
```
'''
{
  "contents": [
    {
      "parts": [
        {
          "text": "{{ 'Summarise these RSS items into 5 bullets for a placement coach. Each bullet ≤ 20 words. Highlight any item signalling new hiring, layoffs, or significant tech demand changes. Items follow:\\n\\n' + $json.items.map(i => '- ' + i.title + ': ' + (i.contentSnippet || i.content || '')).join('\\n') }}"
        }
      ]
    }
  ]
}'''
(The `{{ $json["items"]... }}` is n8n's Mustache syntax — it pulls the RSS output into the prompt.)

**Trainer note (critical):** put the Gemini prompt in an n8n environment variable so iteration doesn't re-deploy. Settings → Variables → add `GEMINI_PROMPT`. Reference it as `={{ $env.GEMINI_PROMPT }}`.

Click **Test step**. Should return a Gemini response with 5 bullets in `candidates[0].content.parts[0].text`.

**Acceptance:** Gemini returns a 5-bullet summary on test.

---

## Step 5 — Add Gmail Send node (15 min)

Click **+** after the HTTP node. Search for "Gmail" → Gmail node.

Configure:
- **Resource**: Message
- **Operation**: Send
- **Email Type**: Text
- **To**: your personal Gmail (the one you want the digest to)
- **Subject**: `Placement News Digest — {{$now.format("dd MMM yyyy")}}`
- **Message**:

```
Good morning. Today's placement news digest:

{{ $json["candidates"][0]["content"]["parts"][0]["text"] }}

Sent automatically by n8n at 7:00 AM IST.
```

### OAuth setup

n8n needs to authenticate with Gmail.

**This is the tricky part.** Two options:

**Option A — Personal Gmail OAuth (recommended for the bootcamp):**
1. In n8n: click "Create New Credential" on the Gmail node
2. n8n provides a redirect URI — copy it
3. In a new tab: https://console.cloud.google.com → create a new project (or use existing)
4. APIs & Services → Library → enable "Gmail API"
5. APIs & Services → Credentials → Create Credentials → OAuth Client ID
6. Application type: Web application
7. Authorized redirect URIs: paste the URI from n8n
8. Download credentials JSON, paste Client ID + Secret back into n8n
9. Click "Sign in with Google" in n8n — grant consent

**Total: ~10 min if smooth, more if Cloud Console permissions issues.**

**Option B (simpler for the bootcamp) — SMTP via app password:**
1. Use n8n's "Send Email" node instead of Gmail node
2. Configure SMTP for Gmail: smtp.gmail.com:587, your gmail address, an app-specific password from Google account settings
3. Skip OAuth flow entirely

**Trainer note:** If a mentor's Cloud Console doesn't allow OAuth (school-managed Google accounts often don't), use Option B. Pre-check during T-1 week.

Click **Test step**. Email should arrive in your inbox in ≤ 30 seconds.

**Acceptance:** Test email visible in your Gmail inbox.

---

## Step 6 — Schedule + activate (10 min)

Add **Schedule Trigger** node BEFORE the RSS node (n8n runs left-to-right).

Configure:
- **Mode**: Cron
- **Cron Expression**: `0 7 * * *` (7 AM daily, every day)
- **Timezone**: Asia/Kolkata

Wire all 4 nodes: Schedule → RSS → HTTP → Gmail.

Click **Save** (Cmd/Ctrl-S).

Toggle the **Active** switch (top-right of the workflow). It turns green.

Click **Execute Workflow** (manual run now). All 4 nodes light up. Email should arrive again in ≤ 30 seconds.

**Acceptance:** Workflow shows ACTIVE. Manual test run sends email.

---

## Step 7 — Export workflow JSON + push (5 min)

In n8n:
- Top-right menu → **Download** → JSON file
- Save as `Day4_NewsDigest.json`

Push to ai-mentor-portfolio repo:

```bash
mv Day4_NewsDigest.json /path/to/ai-mentor-portfolio/
cd /path/to/ai-mentor-portfolio/
git add Day4_NewsDigest.json
git commit -m "Day 4: n8n daily news digest workflow"
git push
```

Update the README:

```markdown
## Day 4 — n8n Daily News Digest

- ✅ Self-hosted n8n via Docker
- ✅ Workflow: Schedule (7AM IST) → RSS → Gemini summariser → Gmail
- ✅ Workflow JSON committed: [Day4_NewsDigest.json](Day4_NewsDigest.json)
- ✅ Test email screenshot below

![Test email screenshot](daily_digest_test_email.png)
```

(Screenshot the email; upload via GitHub web UI.)

**Acceptance:** Workflow JSON + screenshot in repo.

---

## Common bugs + recovery

- **Docker daemon not running** → Open Docker Desktop. Wait for the whale icon to settle.
- **n8n container starts but UI doesn't load** → check `docker logs <n8n container>`. Usually a port conflict. Change port in compose file from 5678 to 15678.
- **RSS node returns empty array** → feed URL is dead. Substitute a different one.
- **Gemini HTTP 401** → wrong key in header. Re-paste from 1Password.
- **Gmail OAuth fails on Google Cloud Console** → school-managed account. Switch to SMTP with app password (Option B).
- **Mentor uses cloud n8n trial instead of self-hosted** → fine for the lab, but they lose the "free forever" benefit. Note it.

---

## Trainer notes

1. **Pre-test the docker-compose on your demo laptop** the morning of. Container start time can vary. Pre-warming saves 60s in the lab.
2. **Walk the room aggressively during Step 5 (Gmail OAuth).** This is where 60% of the lab time gets eaten by Cloud Console errors. Have Option B (SMTP) ready as a fallback.
3. **Surface one mentor's working flow on the projector at 15:25.** Show the test email arriving live. Mentors who are still stuck see what "done" looks like.
4. **The teaching moment is "set it and forget it".** Once the flow is active, mentors get the digest at 7AM tomorrow without doing anything. Have them screenshot tomorrow's email and add it to the README on Day 5 morning.
5. **Stretch goal:** add a Slack node alongside Gmail so the digest also posts to a #placement-news Slack channel. Useful for mentors who want to share with their cohort.

---

## Acceptance check (final 5 min)

For each mentor:
- ✅ n8n container running at localhost:5678
- ✅ Workflow with all 4 nodes wired and ACTIVE
- ✅ Test email received in inbox
- ✅ Workflow JSON exported and pushed to repo
- ✅ Screenshot of test email in repo README

If a mentor is stuck on Gmail OAuth at 15:25, shift them to SMTP option for the last 5 min. Better a working flow on SMTP than an unfinished flow on OAuth.
