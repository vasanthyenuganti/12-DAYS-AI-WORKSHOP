# Day 1 — Lab 1B: Toolkit Setup (Turnkey Walkthrough)

**Time:** 90 minutes (14:00 — 15:30)
**Format:** Individual at laptops; trainer walks the room
**Goal:** Every mentor leaves with a public GitHub repo + working free Gemini API key + first Hello-Gemini call screenshot in README.

---

## Setup (5 minutes — before the lab)

Check that every mentor has:
- A working Google account (for AI Studio, Colab, GitHub OAuth)
- A laptop with Chrome / Firefox + reliable Wi-Fi
- 1Password or any password manager open

Trainer pre-condition: 12 backup AI Studio API keys + 12 backup Groq keys staged in 1Password from T-2 weeks prep.

---

## Step 1 — Provision Google AI Studio API key (10 min)

Mentor goes to **https://aistudio.google.com**

1. Sign in with the Google account they will use for the bootcamp.
2. Click **"Get API key"** (top-left or in left sidebar).
3. Click **"Create API key in new project"**.
4. Copy the key (starts with `AIza...`).
5. Paste into 1Password as `AI_Bootcamp / Gemini_Key`.

**Trainer note:** The key is shown ONCE. If a mentor closes the dialog without copying, they regenerate. Don't let them paste the key into a chat or notebook source.

**Acceptance:** Key is in password manager. Mentor confirms aloud "Gemini key saved."

---

## Step 2 — Provision Groq API key (5 min)

**https://console.groq.com**

1. Sign in (Google login works).
2. Left sidebar → **API Keys** → **Create API Key**.
3. Name it `ai-bootcamp`.
4. Copy. Paste to 1Password as `AI_Bootcamp / Groq_Key`.

**Acceptance:** Two keys in 1Password.

---

## Step 3 — Create public GitHub repo (10 min)

**https://github.com**

1. Sign in. Click **+ → New repository**.
2. **Repository name:** `ai-mentor-portfolio`
3. Visibility: **Public** (non-negotiable — the portfolio is the proof).
4. Initialise with README.
5. Click **Create repository**.

Now edit the README to one line:

```markdown
# AI Mentor Bootcamp — <Your Full Name>

Public portfolio of 12-day AI Trainer Workshop. By Day 12: 6 daily notebooks + capstone Streamlit URL.
```

Commit (button at the bottom of the in-browser editor).

**Trainer note:** Write each mentor's repo URL on the whiteboard next to their name. By Day 12 you reference these URLs for grading.

**Acceptance:** `https://github.com/<username>/ai-mentor-portfolio` is publicly accessible.

---

## Step 4 — Open Google Colab + first notebook (5 min)

**https://colab.research.google.com**

1. Sign in with the same Google account.
2. **File → New notebook**.
3. **File → Save** → confirm it saves to a Drive folder. Create a folder called `ai-mentor-portfolio` if Colab doesn't auto-create one.
4. Rename the notebook to `Day1_Setup.ipynb`.

**Acceptance:** Empty `Day1_Setup.ipynb` saved in Drive.

---

## Step 5 — First Hello-Gemini API call (15 min)

In the Colab notebook, paste this in cell 1 and run:

```python
!pip install -q google-genai
```

Cell 2 — paste:

```python
import os, getpass
os.environ['GEMINI_API_KEY'] = getpass.getpass('Paste your Google AI Studio key: ')
```

Run it. Paste your AI Studio key into the prompt that appears.

**Trainer note:** `getpass()` is non-negotiable. Mentors who paste the key as a string literal will leak it on Step 7 (push to public repo) within 90 seconds. Google revokes leaked keys within hours.

Cell 3 — paste:

```python
from google import genai
client = genai.Client(api_key=os.environ['GEMINI_API_KEY'])

resp = client.models.generate_content(
    model='gemini-2.5-flash',
    contents='Explain the 5-Layer AI Skill Pyramid in 4 short bullet points.'
)
print(resp.text)
```

Run it. Expected output: ~4 bullet points naming the 5 layers (sometimes the model lumps L1+L2 — mentors should notice this and the mentor themselves edits later).

**Acceptance:** A 4-bullet response prints below the cell.

---

## Step 6 — Take a screenshot of the working output (5 min)

Mentor takes a screenshot of cell 3's output.

Save as `gemini_first_call.png` in the same Drive folder.

**Trainer note:** This screenshot is the proof in the README. No screenshot = no acceptance.

---

## Step 7 — Push to GitHub (10 min)

In Colab: **File → Save a copy in GitHub**.

First time: Colab will prompt to authorise GitHub. Click yes.

Form that appears:
- **Repository:** `<your-username>/ai-mentor-portfolio`
- **Branch:** `main`
- **File name:** `Day1_Setup.ipynb`
- **Commit message:** `Day 1 setup complete — Gemini call working`
- ✅ Include a link to Colab

Click **OK**. Colab saves and commits.

Now go to your repo on github.com — `Day1_Setup.ipynb` should appear.

**Edit the README** in the GitHub web UI to add this section:

```markdown
## Day 1 — Setup complete

- ✅ Google AI Studio API key provisioned
- ✅ Groq API key provisioned
- ✅ Hello-Gemini call working — see [Day1_Setup.ipynb](Day1_Setup.ipynb)
- 4-tool comparison matrix from Lab 1A: see screenshot below

![Gemini first call](gemini_first_call.png)
```

(Upload the PNG via GitHub web UI: **Add file → Upload files**.)

Commit.

**Acceptance:** README has the "Day 1 — Setup complete" section visible at github.com/\<username\>/ai-mentor-portfolio.

---

## Common bugs + recovery

- **`401 Unauthorized` on Gemini call** → key is wrong. Mentor regenerates from AI Studio and re-pastes via getpass.
- **`429 Resource exhausted`** → free quota hit. Wait 60s, retry. If still hitting, switch to backup key from 1Password.
- **`Model gemini-2.5-flash not found`** → Google rotates names. Try `gemini-2.0-flash`. Update the docs.
- **Save to GitHub fails with auth error** → Colab → Tools → Settings → GitHub → re-authorise.
- **Mentor pasted key as string in cell 2** → revoke key in AI Studio, regenerate, re-do with getpass. Do NOT skip this — leaked keys in public repos get revoked by Google.

---

## Trainer notes

1. Walk the room continuously through Steps 1-3. The signup flow is where most issues happen — usually wrong Google account or browser cookies.
2. After Step 5 (first Gemini call), surface 2-3 mentors' outputs on the projector. Different runs produce different bullet wordings — that's expected, and a teaching moment ("non-determinism").
3. If a mentor's free quota is exhausted before they push (rare on Day 1, common on Day 6), give them a backup key.
4. Acceptance verification at 15:25: walk the room. Open each mentor's GitHub URL on the projector. Green checkmark on `Day1_Setup.ipynb` = acceptance. No exceptions.

---

## Acceptance check (final 5 min)

For every mentor, verify:
- ✅ Public GitHub repo `ai-mentor-portfolio` exists and is reachable
- ✅ `Day1_Setup.ipynb` is committed and renders correctly on GitHub
- ✅ README has "Day 1 — Setup complete" section
- ✅ Screenshot of working Gemini call is in the README
- ✅ Mentor states aloud: "I have my Gemini key, my Groq key, my repo, and my notebook."

If any item is missing by 15:25, pair the mentor with you for the last 5 minutes. No one leaves Day 1 without a working setup — this is the foundation for the next 11 days.
