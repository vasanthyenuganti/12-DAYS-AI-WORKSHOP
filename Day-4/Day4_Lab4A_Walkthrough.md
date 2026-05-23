# Day 4 — Lab 4A: Research + Deck Sprint (Turnkey Walkthrough)

**Time:** 45 minutes (11:15 — 12:00). 45-minute timebox is the lesson.
**Format:** Individual; trainer demos in 90 seconds first
**Goal:** Each mentor produces a 2-page placement-prep brief PDF + 8-slide deck PDF on one Tier-1 hiring company. Free tools end-to-end.

---

## Setup (5 min)

Each mentor opens (free accounts):
- Perplexity (https://perplexity.ai) — daily quota 5 Pro searches (this is the resource constraint)
- Google AI Studio (https://aistudio.google.com) with Gemini 2.5 + **Grounding with Google Search** toggle ON
- NotebookLM (https://notebooklm.google.com)
- Gamma (https://gamma.app) — free tier ≈ 400 credits, ≈ 5 decks

Each mentor picks ONE Tier-1 hiring company:
- TCS / Infosys / Wipro / Accenture / Capgemini / Cognizant / HCL / Deloitte USI

(Random allocation: trainer assigns to ensure variety across the room.)

---

## Step 1 — Plan 5 Perplexity questions BEFORE opening Perplexity (5 min)

Mentor writes the 5 questions in their scratch doc FIRST. Free Perplexity allows only 5 Pro searches/day; freeform browsing burns the quota.

### The 5 standard questions (substitute company name)

1. "What is the hiring process at <COMPANY> for B.Tech freshers in 2025-2026? Number of rounds, types, and what each tests."
2. "What is the technical stack <COMPANY> hires for in their 2025-2026 fresher batch? Languages, frameworks, cloud, databases."
3. "What recent news (2025-2026) is relevant for B.Tech students preparing to interview at <COMPANY>? Hiring numbers, pivots, layoffs."
4. "What are the eligibility criteria at <COMPANY> for B.Tech 2026 placement — CGPA cutoff, branches, backlogs policy, package range?"
5. "What are 5 specific preparation tips for a B.Tech 3rd-year student targeting <COMPANY> placement? Concrete and actionable."

**Trainer note:** No freeform Perplexity browsing. The 5 questions are the budget. Plan, then ask.

**Acceptance:** 5 questions written before opening Perplexity.

---

## Step 2 — Run Perplexity (10 min)

Open Perplexity. Run each of the 5 questions. For each:
- Read the answer
- Click 2-3 of the cited sources
- Save the cited URLs (you'll need them for the brief)

Total: ~10 min for all 5.

**Trainer note:** Perplexity may say "I'd need more context for this question." When it does, accept the answer it can give. Don't burn a 6th search.

**Acceptance:** 5 Perplexity answers + cited URLs saved in scratch doc.

---

## Step 3 — Generate the 2-page brief via Gemini grounding (15 min)

Open AI Studio. Confirm **Grounding with Google Search** is ON (toggle in the right sidebar of the conversation).

Paste this prompt with your Perplexity findings:

> "Weave the following 5 Q&A pairs into a clean 2-page placement-prep brief on <COMPANY> for B.Tech students. Add citations using the Google grounding sources. Plain text format. Sections: Overview, Hiring Process, Technical Stack, Recent News, Eligibility, Prep Tips, Sources.
>
> Q1: <Perplexity question 1>
> A1: <Perplexity answer 1>
>
> Q2: <Perplexity question 2>
> A2: <Perplexity answer 2>
>
> [continue for all 5]"

Gemini with grounding produces a brief with live citations.

Save the output as a Google Doc. **File → Download → PDF**. Save as `<COMPANY>_brief.pdf`.

**Acceptance:** 2-page PDF with grounded citations.

---

## Step 4 — Generate the 8-slide deck via Gamma (10 min)

Open Gamma. Click **+ New Document → Generate**.

Prompt:

> "Create an 8-slide placement-prep deck for B.Tech students on <COMPANY>. Cover slide 1 (title + value prop). Slide 2: Why <COMPANY>. Slide 3: Hiring process. Slide 4: Technical stack. Slide 5: Eligibility. Slide 6: Recent news. Slide 7: 5 prep tips. Slide 8: CTA — start preparing this week."

Gamma generates 8 slides in ~20 seconds.

**Hand-edit slide 1 (cover) and slide 8 (CTA).** This is the lesson — Gamma generates the middle, humans control the ends.

**Edit-check every numeric slide.** Gamma confabulates statistics. Cross-check:
- Hiring numbers (e.g., "TCS hires 40K freshers in 2025") — must match your Perplexity findings or remove.
- Package ranges (e.g., "₹6-8 LPA") — must match official source or remove.
- CGPA cutoffs — must match the eligibility section of your brief.

Replace any unverifiable number with "[verify with company]" rather than ship it wrong.

Export: **Share → Export → PDF**. Save as `<COMPANY>_deck.pdf`.

**Acceptance:** 8-slide PDF with hand-edited cover and CTA, all numeric slides verified.

---

## Step 5 — Document edits in repo (5 min)

Push to ai-mentor-portfolio:
- `Day4_<COMPANY>_brief.pdf`
- `Day4_<COMPANY>_deck.pdf`
- Update README with this section:

```markdown
## Day 4 — Productivity sprint

**Company:** <COMPANY>
**Time:** 45 minutes (timeboxed)

### Edit notes (3 lines)

1. Gamma confabulated a "hiring 50,000 freshers in 2025" stat on slide 6. Source said 40,000. Edited.
2. Slide 4 listed "Kubernetes" as a required skill — actually nice-to-have per the JD. Edited.
3. Slide 1 (cover) — replaced Gamma's generic "Your Career Awaits" with a company-specific line.
```

Commit and push.

**Acceptance:** Brief PDF + Deck PDF + README edit-notes all in the repo.

---

## Common bugs + recovery

- **Gamma free tier credits exhausted** mid-edit → continue with the existing draft. Don't regenerate.
- **Perplexity quota hits 5/5 mid-task** → switch to Gemini grounding for the remaining questions. Document this in your edit notes.
- **Mentor goes over 45-minute timebox** → note it. The timebox IS the lesson. Push them to ship at 45 even if not perfect.
- **Mentor freeform-browses Perplexity instead of asking the 5 planned questions** → re-anchor the discipline. The 5 planned questions are the budget.
- **Gamma deck has a slide that doesn't fit the structure** (e.g., 9 slides instead of 8, or a duplicate slide) → ignore. Don't waste credits regenerating. Edit in place.

---

## Trainer notes

1. **Walk the room continuously.** This is a fast, time-pressured lab. Mentors who hit blockers lose minutes; you keep them moving.
2. **At 11:35, surface one mentor's brief on the projector.** Compare to their company's actual careers page. Where did Gemini get it right? Where did it confabulate?
3. **At 11:55, surface one Gamma deck on the projector.** Walk through which slides were edited and which weren't. The skill is knowing what to verify.
4. **The 45-minute constraint is the entire lesson.** A mentor who finishes in 60 minutes hasn't learnt the productivity habit. Hold the line.
5. **Stretch goal for fast finishers (rare):** record a 60-second Loom walkthrough of the deck. Embed in README.

---

## Acceptance check (final 5 min, 12:00 sharp)

For each mentor:
- ✅ Brief PDF in repo (≤ 2 pages, with grounded citations)
- ✅ Deck PDF in repo (8 slides, cover + CTA hand-edited)
- ✅ README has 3-line edit-notes (which numbers needed verification)
- ✅ Total time elapsed ≤ 45 min (note overruns; do NOT extend)

If a mentor is at 60+ min, they ship at 60 with whatever they have. The lab teaches the timebox. Better to ship imperfect than to learn nothing.
