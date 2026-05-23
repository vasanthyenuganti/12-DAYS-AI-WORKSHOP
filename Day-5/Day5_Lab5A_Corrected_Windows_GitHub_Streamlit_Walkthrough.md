# Day 5 — Lab 5A: Résumé-Scorer Streamlit  
## Corrected Turnkey Walkthrough for Windows + GitHub + Streamlit Cloud

**Time:** 90 minutes  
**Format:** Individual hands-on lab in VS Code  
**Goal:** Build and deploy a public Streamlit app that scores a résumé against a job description using Gemini API and Continue.dev.

---

## Setup Checklist

Before starting, each participant should have:

- Python 3.10+ installed
- VS Code installed
- Git installed
- GitHub account
- Streamlit Community Cloud account
- Gemini API key from Google AI Studio
- Continue.dev extension installed in VS Code

---

# Step 1 — Install Continue.dev VS Code Extension

In VS Code:

1. Open Extensions sidebar.
2. Search for:

```text
Continue
```

3. Install:

```text
Continue - Codestral, Claude, and more
```

4. Click the Continue icon in the left sidebar.
5. In first-run setup, select:

```text
Continue with Gemini
```

6. Paste your Gemini API key when prompted.
7. Test using:

```text
Ctrl + L
```

Ask:

```text
what is recursion?
```

**Acceptance:** Continue panel responds using Gemini.

---

# Step 2 — Project Setup

Open PowerShell or VS Code terminal.

```bash
mkdir resume-scorer
cd resume-scorer
```

Create virtual environment:

```bash
py -m venv venv
```

Activate environment:

```bash
venv\Scripts\activate
```

Install required packages:

```bash
py -m pip install streamlit google-genai pydantic
```

Open folder in VS Code:

```bash
code .
```

---

# Step 3 — Correct Folder Structure

Your project should look like this:

```text
resume-scorer/
│
├── app.py
├── requirements.txt
├── .gitignore
├── acceptance_log.md
├── venv/
│
└── .streamlit/
    └── secrets.toml
```

Important:

```text
app.py should be inside resume-scorer, NOT inside venv.
```

Wrong:

```text
resume-scorer/
└── venv/
    └── app.py   ❌
```

Correct:

```text
resume-scorer/
├── app.py       ✅
└── venv/
```

---

# Step 4 — Create `app.py`

Inside `resume-scorer`, create:

```text
app.py
```

Paste this corrected code:

```python
import streamlit as st
from google import genai
import json

st.set_page_config(page_title="Résumé Scorer", layout="wide")

st.title("Résumé vs JD Fit Scorer")
st.caption("Day 5 Lab 5A — Gemini + Continue.dev + Streamlit")

col1, col2 = st.columns(2)

with col1:
    resume = st.text_area("Paste résumé", height=400)

with col2:
    jd = st.text_area("Paste job description", height=400)

api_key = st.text_input("Gemini API key", type="password")

try:
    api_key = st.secrets["GEMINI_API_KEY"]
except Exception:
    pass

if st.button("Score") and resume and jd and api_key:
    with st.spinner("Scoring..."):
        client = genai.Client(api_key=api_key)

        prompt = f"""
You are a placement coach.

Given this résumé and job description, return only valid JSON in this exact format:

{{
  "score": 0,
  "rationale": "short explanation",
  "missing_skills": ["skill1", "skill2"],
  "suggestions": ["suggestion1", "suggestion2"],
  "technical_skills_match": 0,
  "soft_skills_match": 0,
  "experience_relevance": 0,
  "project_fit": 0,
  "learning_resources": [
    {{
      "skill": "Docker",
      "resource_type": "YouTube",
      "link": "https://youtube.com/example"
    }}
  ]
}}

Rules:
- score must be between 0 and 100.
- sub-scores must be between 0 and 100.
- missing_skills must contain only real missing skills, not generic words.
- learning_resources must suggest free learning resources.

Résumé:
{resume}

Job Description:
{jd}
"""

        resp = client.models.generate_content(
            model="gemini-2.5-flash",
            contents=prompt,
            config={"response_mime_type": "application/json"}
        )

        try:
            result = json.loads(resp.text)

            st.metric("Fit Score", f"{result['score']}/100")

            st.subheader("Score Breakdown")
            breakdown = {
                "Technical Skills Match": result.get("technical_skills_match", 0),
                "Soft Skills Match": result.get("soft_skills_match", 0),
                "Experience Relevance": result.get("experience_relevance", 0),
                "Project Fit": result.get("project_fit", 0),
            }
            st.bar_chart(breakdown)

            st.subheader("Rationale")
            st.write(result.get("rationale", ""))

            st.subheader("Missing Skills")
            for skill in result.get("missing_skills", []):
                st.write(f"- {skill}")

            st.subheader("Suggestions")
            for suggestion in result.get("suggestions", []):
                st.write(f"- {suggestion}")

            st.subheader("Top Missing Skills with Learning Resources")
            for item in result.get("learning_resources", []):
                st.write(
                    f"- **{item.get('skill', '')}** "
                    f"({item.get('resource_type', '')}): "
                    f"{item.get('link', '')}"
                )

        except json.JSONDecodeError:
            st.error("Could not parse Gemini response as JSON.")
            st.write(resp.text)
```

---

# Step 5 — Run Locally

In terminal:

```bash
py -m streamlit run app.py
```

Browser opens at:

```text
http://localhost:8501
```

Test with one résumé and one job description.

**Acceptance:** App loads with two text boxes, API key box, and Score button.

---

# Step 6 — Create `requirements.txt`

In `resume-scorer`, create:

```text
requirements.txt
```

Paste:

```text
streamlit
google-genai
pydantic
```

Why this is required:

```text
Streamlit Cloud installs packages only from requirements.txt.
```

Without this file, deployment may fail with:

```text
ImportError: from google import genai
```

---

# Step 7 — Create Streamlit Secrets Locally

Inside `resume-scorer`, create folder:

```text
.streamlit
```

Inside `.streamlit`, create file:

```text
secrets.toml
```

Paste:

```toml
GEMINI_API_KEY = "your_actual_gemini_api_key"
```

Example:

```toml
GEMINI_API_KEY = "AIzaSyxxxxxxxxxxxx"
```

---

# Step 8 — Create `.gitignore`

Inside `resume-scorer`, create:

```text
.gitignore
```

Paste:

```text
venv/
.streamlit/secrets.toml
__pycache__/
```

This protects your API key from uploading to GitHub.

---

# Step 9 — Create `acceptance_log.md`

Create:

```text
acceptance_log.md
```

Paste:

```markdown
# Day 5 Lab 5A — Continue.dev Acceptance Log

1. Accepted: Streamlit two-column layout
   Why: separates résumé and JD input clearly.

2. Accepted: Gemini JSON output structure
   Why: keeps the model response predictable.

3. Accepted: score breakdown chart
   Why: helps users understand the score visually.

4. Accepted: missing skills section
   Why: tells students what skills are absent.

5. Accepted: learning resources section
   Why: gives students practical next steps.

6. Accepted: Streamlit secrets handling
   Why: prevents API key exposure in GitHub.

7. Accepted: requirements.txt
   Why: required for Streamlit Cloud deployment.
```

---

# Step 10 — Initialize Git

In terminal:

```bash
git init
```

If Git asks for identity:

```bash
git config --global user.name "Dr. Gandikota Ramu"
git config --global user.email "your_github_email@gmail.com"
```

Then:

```bash
git add .
git commit -m "Day 5 resume scorer"
```

---

# Step 11 — Create New GitHub Repository

Go to GitHub:

```text
https://github.com/new
```

Repository name:

```text
resume-scorer
```

Select:

```text
Public
```

Important:

- Do not add README
- Do not add .gitignore
- Do not add license

Click:

```text
Create repository
```

---

# Step 12 — Connect Local Project to New Repository

If old remote exists, remove it:

```bash
git remote remove origin
```

Add new remote:

```bash
git remote add origin https://github.com/YOUR_USERNAME/resume-scorer.git
```

Example:

```bash
git remote add origin https://github.com/gramucse/resume-scorer.git
```

Set branch:

```bash
git branch -M main
```

Push:

```bash
git push -u origin main
```

**Acceptance:** Code appears in the new GitHub repository.

---

# Step 13 — Deploy on Streamlit Community Cloud

Open:

```text
https://share.streamlit.io
```

Login with GitHub.

Click:

```text
New app
```

Select:

```text
Repository: resume-scorer
Branch: main
Main file path: app.py
```

Before deploying, open:

```text
Advanced settings → Secrets
```

Paste:

```toml
GEMINI_API_KEY = "your_actual_gemini_api_key"
```

Click:

```text
Deploy
```

Wait 1–2 minutes.

Your app URL will look like:

```text
https://resume-scorer.streamlit.app
```

---

# Step 14 — Smoke Test Deployed App

Open the public URL.

Paste sample résumé and JD.

Click:

```text
Score
```

Expected output:

- Fit Score
- Score Breakdown chart
- Rationale
- Missing skills
- Suggestions
- Learning resources

**Acceptance:** Public app works without errors.

---

# Step 15 — Create README

Create:

```text
README.md
```

Paste:

```markdown
# Day 5 — Résumé Scorer Streamlit

**Live URL:** https://your-app-name.streamlit.app  
**Code:** [app.py](app.py)  
**Acceptance Log:** [acceptance_log.md](acceptance_log.md)

## Tools Used

- Continue.dev
- Gemini 2.5 Flash
- Streamlit
- GitHub
- Streamlit Community Cloud

## Features

- Résumé vs JD fit score
- Rationale
- Missing skills
- Suggestions
- 4-axis score breakdown chart
- Free learning resources for missing skills

## Reflection

- This is an AI-assisted prototype.
- To productionise, I would add better error handling, caching, rate limits, and authentication.
- Continue.dev helped scaffold the UI quickly, but manual review was needed for prompt correctness and deployment fixes.
```

Push README:

```bash
git add README.md
git commit -m "Added README"
git push
```

---

# Common Errors and Fixes

## 1. Python not found

Error:

```text
Python was not found
```

Fix:

```bash
py --version
```

Use `py` instead of `python`.

---

## 2. pip not recognized

Error:

```text
pip is not recognized
```

Fix:

```bash
py -m pip install streamlit google-genai pydantic
```

---

## 3. app.py not found

Error:

```text
File does not exist: app.py
```

Fix:

```bash
dir
```

Make sure `app.py` is directly inside `resume-scorer`.

---

## 4. ImportError: from google import genai

Reason:

```text
requirements.txt missing
```

Fix:

```text
Create requirements.txt with:
streamlit
google-genai
pydantic
```

Commit and push.

---

## 5. Streamlit secrets error

Error:

```text
StreamlitSecretNotFoundError
```

Fix:

In Streamlit Cloud:

```text
Manage app → Settings → Secrets
```

Paste:

```toml
GEMINI_API_KEY = "your_actual_key"
```

---

## 6. Git push rejected

Error:

```text
main -> main (fetch first)
```

Best fix for this lab:

```text
Create a new empty GitHub repository and push there.
```

Or:

```bash
git pull origin main --allow-unrelated-histories
git push
```

---

## 7. JSON prompt error

Problem line:

```python
"learning_resources": [{"skill": str, "resource_type": str, "link": str}]
```

Correct format:

```python
"learning_resources": [
  {
    "skill": "Docker",
    "resource_type": "YouTube",
    "link": "https://youtube.com/example"
  }
]
```

Inside an f-string, JSON braces must be written carefully using doubled braces when needed.

---

# Final Acceptance Checklist

Each mentor should show:

- App runs locally
- App is deployed on Streamlit Cloud
- Public URL works
- GitHub repository contains app.py
- requirements.txt exists
- .gitignore protects secrets
- API key is added only in Streamlit secrets
- acceptance_log.md has 5+ explanation entries
- README contains live URL and reflection

---

# Trainer Notes

1. Use this lab to teach AI-assisted coding with verification.
2. Ask every mentor to explain every accepted Continue.dev suggestion.
3. Demonstrate why API keys must never be pushed to GitHub.
4. Show Streamlit Cloud logs when errors occur.
5. Reinforce that deployment errors are part of real software development.
