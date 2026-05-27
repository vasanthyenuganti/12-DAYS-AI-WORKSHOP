# Day 6 — Capstone Sprint 1: PlacementDataProcessor (Turnkey Walkthrough)

**Time:** 90 minutes (14:00 — 15:30)
**Format:** Pair work in Colab; same pair as Lab 6A
**Goal:** JD URL → BeautifulSoup → Gemini structured call → clean JSON. The Data Layer of the capstone — every later sprint reads `jds.jsonl` as input.

---

## Setup (5 min)

Each pair:
- Continues with their Lab 6A notebook (or opens a fresh `Day6_PlacementProcessor.ipynb`)
- Has 5 JD URLs to process — assigned by trainer (random allocation across pairs to ensure variety)
- Has `data/jds_cached.jsonl` from the lab kit as fallback if scraping is blocked

### Suggested JD URLs (use any 5)

- TCS Digital — search "TCS Digital fresher 2026" on Naukri/LinkedIn, pick top JD
- Infosys — same
- Capgemini, Cognizant, Wipro — same
- For pairs assigned product companies: Amazon SDE Intern, Microsoft Software Engineer Intern

(Pre-cache 10 JD URLs in `data/jds_urls.txt` during T-1 week prep, in case scraping is blocked.)

---

## Step 1 — Define JD Pydantic schema (10 min)

Cell 6 (continuing the morning's notebook):

```python
class JD(BaseModel):
    company: str
    role: str
    must_have_skills: List[str]
    nice_to_have_skills: List[str] = []
    min_cgpa: Optional[float] = None
    locations: List[str] = []
    package_lpa: Optional[float] = None
```

**Acceptance:** Schema imports without error.

---

## Step 2 — JD scraper with cached-fallback (15 min)

Cell 7:

```python
import requests
from bs4 import BeautifulSoup
import pathlib, json

def fetch_jd(url, max_chars=6000):
    """Fetch JD URL and return clean text. Returns None on block / failure."""
    try:
        r = requests.get(url, headers={'User-Agent': 'Mozilla/5.0'}, timeout=10)
        r.raise_for_status()
        soup = BeautifulSoup(r.text, 'html.parser')
        # Remove script and style tags
        for tag in soup(['script', 'style']):
            tag.decompose()
        return soup.get_text(separator='\n', strip=True)[:max_chars]
    except Exception as e:
        print(f'  Scrape failed for {url}: {e}')
        return None

# Test on one URL
test_url = 'TODO_REPLACE_WITH_ASSIGNED_URL'
text = fetch_jd(test_url)
if text:
    print(f'Got {len(text)} chars')
    print(text[:300])
else:
    print('Scrape blocked. Will use cached set.')
```

**Trainer note:** if Naukri / LinkedIn return 403, that's expected. Tell mentors to switch to cached. Real-world resilience pattern.

**Acceptance:** Scraper returns text on at least one URL, OR mentor explicitly switches to cached path.

---

## Step 3 — Normalise via Gemini structured call (15 min)

Cell 8:

```python
def normalise_jd(text: str) -> JD:
    """Send JD text to Gemini, get structured JD JSON back."""
    resp = client.models.generate_content(
        model='gemini-2.5-flash',
        contents=f'Extract a JD JSON from this text:\n\n{text}',
        config={
            'response_mime_type': 'application/json',
            'response_schema': JD.model_json_schema(),
        },
    )
    return JD.model_validate_json(resp.text)

# Test on one JD text
if text:
    jd = normalise_jd(text)
    print(jd.model_dump_json(indent=2))
```

**Acceptance:** One JD parsed into structured JSON.

---

## Step 4 — Process 5 JDs (with cached fallback) (20 min)

Cell 9:

```python
import json, pathlib

URLS = [
    # Paste your 5 assigned URLs here
    'TODO_url_1',
    'TODO_url_2',
    'TODO_url_3',
    'TODO_url_4',
    'TODO_url_5',
]

CACHE = pathlib.Path('../data/jds_cached.jsonl')
USE_CACHE = False   # set True if scraping is blocked

jds = []

if USE_CACHE and CACHE.exists():
    print(f'Using cached JDs from {CACHE}')
    for line in CACHE.read_text().splitlines():
        jds.append(JD.model_validate_json(line))
else:
    for url in URLS:
        text = fetch_jd(url)
        if text is None:
            continue
        try:
            jd = normalise_jd(text)
            jds.append(jd)
            print(f'  ✓ {jd.company} — {jd.role}')
        except Exception as e:
            print(f'  ✗ {url}: {e}')

print(f'\nProcessed {len(jds)} JDs')

# Inspect first 3
for jd in jds[:3]:
    print(f'\n{jd.company} - {jd.role}')
    print(f'  Must: {jd.must_have_skills}')
    print(f'  Nice: {jd.nice_to_have_skills}')
    print(f'  CGPA: {jd.min_cgpa}, LPA: {jd.package_lpa}')
```

**Acceptance:** ≥3 JDs successfully processed (live OR cached).

---

## Step 5 — Save jds.jsonl (15 min)

Cell 10:

```python
OUT = pathlib.Path('data/jds.jsonl')
OUT.parent.mkdir(exist_ok=True)
with open(OUT, 'w') as f:
    for jd in jds:
        f.write(jd.model_dump_json() + '\n')
print(f'Wrote {len(jds)} JDs to {OUT}')

# Verify the file
with open(OUT) as f:
    for line in f:
        d = json.loads(line)
        print(f'  {d["company"]:20} | {d["role"]:30} | {len(d["must_have_skills"])} must-haves')
```

**Acceptance:** `data/jds.jsonl` has ≥3 valid lines, each is a valid JD JSON.

**Trainer note:** This file is the input for Day 7 RAG. Tomorrow morning every pair indexes this file into ChromaDB. Make sure it's saved AND committed to repo.

---

## Step 6 — Engineer Answer in README (10 min)

Update README with the 5-question Engineer Answer:

```markdown
## Day 6 — Capstone Sprint 1: PlacementDataProcessor

### Engineer Answer

1. **PROBLEM** — JDs from Naukri / LinkedIn are messy text — placement cells need structured data to filter ("which JDs want Java + CGPA 7+?"). Manual extraction is unscalable for 50+ JDs.

2. **ARCHITECTURE** — JD URL → BeautifulSoup scraper (extract clean text) → Gemini structured-output call (response_schema=JD Pydantic) → JSON Lines file. Validation at each step; retry on schema fail.

3. **TRADE-OFFS** —
   - Cost: free Gemini ~1 JD/sec on average; ~30K tokens/day quota → ~5K JDs/day.
   - Accuracy: Pydantic catches schema violations but not semantic errors (e.g., model says skill is "Python" when source says "Python 3.12 specifically").
   - Latency: ~2-5s per JD (Gemini call dominant).
   - Complexity: scraping fragile (sites block automation). Cached fallback is mandatory.

4. **SCALE** —
   - 10 JDs/day: trivial. Today's lab.
   - 100 JDs/day: still in free quota. Add overnight batch + sleep between calls.
   - 10K JDs/day: free tier breaks. Move to paid Gemini OR self-host an open model.

5. **INTERVIEW ANSWER** — "I built a structured-output pipeline that turns scraped JDs into clean filterable JSON, using free Gemini and Pydantic. Schema-first design with retry-on-failure made it production-shaped on a free-tier API."

### Files
- `Day6_PlacementProcessor.ipynb` — the notebook
- `data/jds.jsonl` — output of this sprint, input for Day 7 RAG

### Pair: <Mentor 1 name> + <Mentor 2 name>
```

**Acceptance:** Engineer Answer is in README with all 5 questions answered specifically (not generically).

---

## Step 7 — Push to GitHub before 15:30 (mandatory) (5 min)

```bash
git add Day6_PlacementProcessor.ipynb data/jds.jsonl README.md
git commit -m "Day 6 Sprint 1: PlacementDataProcessor — JD URL to JSON"
git push
```

**Trainer note:** Walk the room. Verify each pair has a green checkmark on `Day6_PlacementProcessor.ipynb` at github.com BEFORE the 15:30 break.

---

## Common bugs + recovery

- **Naukri 403 (scraping blocked)** → switch to cached. Document in README.
- **Gemini 429 mid-batch** → either wait or switch to backup key. The afternoon doesn't have time to wait long; pre-stage backup.
- **JD has no CGPA but Pydantic raises** → check Optional was on `min_cgpa: Optional[float] = None`. Common miss.
- **JD scrape returns mostly navigation HTML** → improve scraping selector (find the JD content div), OR accept noisy input and let Gemini extract what's there.
- **Two pairs end up with 0 valid JDs** → scraping issue. Pause both pairs. Switch to cached data. Note in README that cached was used.

---

## Trainer notes

1. **Pair work continues from Lab 6A.** Same pair through Day 12. Drive/review every 30 min.
2. **Walk the room continuously.** Sprint 1 is where the capstone starts. A pair that falls behind today falls behind for 5 days.
3. **The teaching moment is "real-world resilience".** When Naukri blocks, mentors learn to fall back to cached data. This is what their students will face. Tomorrow's RAG sprint depends on `jds.jsonl` existing — ensure it does, even if cached.
4. **The Engineer Answer is your Day 12 teach-back currency.** Be strict on specificity. "We extract data from JDs" is generic and gets a 0. "We use response_schema with Pydantic to enforce 7-field JD JSON, with retry-on-validation-failure" is specific and gets a 1.
5. **Acceptance verification at 15:25:** project github.com/<one pair>/ai-mentor-portfolio. Show the green checkmark + jds.jsonl file. This is what every pair needs.

---

## Acceptance check (final 5 min — mandatory before break)

For each pair:
- ✅ Day6_PlacementProcessor.ipynb runs end-to-end
- ✅ data/jds.jsonl in repo with ≥3 valid JDs
- ✅ Engineer Answer in README with all 5 questions answered specifically
- ✅ Pair names credited in README
- ✅ Push verified at github.com (you walk the room and check the green checkmark)

If any pair is missing the push at 15:30, hold them through the break. Day 7 morning depends on this file. No mentor leaves without a push.
