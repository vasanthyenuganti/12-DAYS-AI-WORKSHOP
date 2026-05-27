# Day 6 — Lab 6A: Hello-Gemini Structured Output (Turnkey Walkthrough)

**Time:** 90 minutes (11:15 — 13:00)
**Format:** Pair work in Colab; trainer demos one cell live
**Goal:** Force schema-correct JSON on 5 sample résumés. Handle the 3 most common errors. Foundation for Day 6 afternoon Sprint 1 (PlacementDataProcessor).

---

## Setup (5 min)

Each pair opens:
- Colab → load `Day6_PlacementProcessor.ipynb` from the lab kit (Lab 6A is cells 1-3 of this notebook)
- Their Gemini API key
- `data/sample_resumes.txt` from the lab kit

Pair work: same notebook, two laptops side-by-side. One drives, the other reviews. Swap every 30 min.

---

## Step 1 — Install + key (5 min)

Cell 1:

```python
!pip install -q google-genai pydantic
import os, getpass
if 'GEMINI_API_KEY' not in os.environ:
    os.environ['GEMINI_API_KEY'] = getpass.getpass('Gemini API key: ')
```

**Acceptance:** No error.

---

## Step 2 — Define Resume Pydantic schema (10 min)

Cell 2:

```python
from pydantic import BaseModel
from typing import List, Optional

class Education(BaseModel):
    degree: str
    institution: str
    year: int

class Resume(BaseModel):
    name: str
    email: str
    phone: Optional[str] = None
    education: List[Education]
    skills: List[str]
    projects: List[str] = []
    experience_years: float
```

**Acceptance:** Cell runs without error.

---

## Step 3 — Provider-agnostic Gemini call with retry (15 min)

Cell 3:

```python
from google import genai
from pydantic import ValidationError

client = genai.Client(api_key=os.environ['GEMINI_API_KEY'])

def extract_resume(raw_text: str, max_retries: int = 1) -> Resume:
    """Extract a Resume JSON from raw text. Retries once on schema fail."""
    for attempt in range(max_retries + 1):
        try:
            resp = client.models.generate_content(
                model='gemini-2.5-flash',
                contents=f'Extract a Resume JSON from this text. Return ONLY JSON, no markdown.\n\n{raw_text}',
                config={
                    'response_mime_type': 'application/json',
                    'response_schema': Resume.model_json_schema(),
                },
            )
            return Resume.model_validate_json(resp.text)
        except ValidationError as e:
            if attempt == max_retries:
                raise
            fix_prompt = f'Fix this JSON to match schema. Errors: {e}. Original: {resp.text}'
            resp = client.models.generate_content(
                model='gemini-2.5-flash', contents=fix_prompt,
                config={'response_mime_type': 'application/json',
                        'response_schema': Resume.model_json_schema()})
            return Resume.model_validate_json(resp.text)
```

**Acceptance:** Function defined.

---

## Step 4 — Process 5 sample résumés (20 min)

Cell 4:

```python
# Load 5 sample résumés
with open('../data/sample_resumes.txt') as f:
    resumes = [r.strip() for r in f.read().split('---') if r.strip()]
print(f'Loaded {len(resumes)} sample résumés')

results = []
errors = []
for i, r in enumerate(resumes):
    try:
        parsed = extract_resume(r)
        results.append(parsed)
        print(f'  [{i+1}] {parsed.name} — {len(parsed.skills)} skills')
    except Exception as e:
        errors.append((i, e))
        print(f'  [{i+1}] FAILED: {type(e).__name__}: {str(e)[:120]}')

print(f'\n{len(results)}/5 succeeded, {len(errors)} failed')
```

### Expected output

5 résumés, all 5 succeed (with maybe 1-2 retries triggering invisibly).

If a résumé fails, it's almost always Priya Nair (résumé 4) — minimal résumé, sometimes Gemini doesn't produce a phone field at all (works because Optional). Or résumé 5 (Karthik Sharma) — has phone but the email line is on the same line as name.

**Acceptance:** ≥4 of 5 résumés succeed.

---

## Step 5 — Test broken input (10 min)

Cell 5:

```python
# Empty string
try:
    bad = extract_resume('')
    print('Unexpected success:', bad.model_dump_json())
except Exception as e:
    print(f'Empty input: {type(e).__name__}: {str(e)[:200]}')

# Whitespace only
try:
    bad = extract_resume('   \n\n   ')
    print('Unexpected success:', bad.model_dump_json())
except Exception as e:
    print(f'Whitespace input: {type(e).__name__}: {str(e)[:200]}')

# Garbage non-résumé text
try:
    bad = extract_resume('the quick brown fox jumps over the lazy dog')
    print('Garbage input:', bad.model_dump_json())
except Exception as e:
    print(f'Garbage input: {type(e).__name__}: {str(e)[:200]}')
```

### Expected output

- Empty string: ValidationError caught
- Whitespace: ValidationError caught
- Garbage: either ValidationError (Gemini refuses) OR Gemini hallucinates a fake résumé (which is its own teaching moment — show it on projector and ask "what just happened?")

**Acceptance:** Empty and whitespace inputs handled gracefully. Garbage input either rejected or surfaced for discussion.

---

## Step 6 — Document the 3 errors handled (10 min)

In the notebook README cell:

```markdown
## Day 6 Lab 6A — Errors handled

1. **Markdown fence wrapping** (`\`\`\`json ... \`\`\``)
   The retry prompt asks Gemini to output raw JSON without fences. Triggers on ~5-10% of calls.

2. **Hallucinated phone number when source has none**
   `Optional[str] = None` in Pydantic — model returns `null`, schema validates.

3. **Empty / whitespace-only input**
   Pydantic raises ValidationError with "Field required". Caller catches.

**Hallucination on garbage input:** Gemini sometimes invents a plausible résumé from non-résumé text. Defence: validate input before sending (e.g., minimum length, presence of email-like pattern).
```

**Acceptance:** README documents the errors with reasoning.

---

## Common bugs + recovery

- **Markdown ```json fences in output** despite mime type → retry handles. If still failing, set `temperature=0` in config.
- **`Pydantic ValidationError: name Field required`** on a real résumé → add explicit hint to prompt: "The first line is the candidate's name."
- **`429 Resource exhausted` mid-batch** → wait 60s + retry, OR switch to backup key. The afternoon Sprint 1 wires Groq fallback to handle this automatically.
- **Hallucinated résumé from garbage** → flag in the room. This is the foundation of the Day 8 red-team: input sanity checks before LLM calls.

---

## Trainer notes

1. **Pair-work rule:** swap driver every 30 min. The reviewer catches mistakes the driver misses. After this lab, the same pair stays through Day 12.
2. **The teaching moment is `response_schema` vs "please return JSON".** Show it live: remove `response_schema` from cell 3 and run on résumé 4 (sparse). Model invents fields. Put `response_schema` back. Model now returns `null` for missing fields.
3. **Surface 2 mentors' results on the projector at 12:45.** Compare extraction quality across pairs. Where did the same résumé produce different `skills` lists? Discussion: extraction is partly subjective.
4. **Connect to Day 6 afternoon (Sprint 1).** "We just extracted résumés. This afternoon we use the same pattern on Job Descriptions. Schema-first. Production-grade."
5. **The garbage-input hallucination is the most pedagogically rich moment.** Mentors who watch Gemini invent a résumé out of "the quick brown fox" never forget the lesson: verify inputs before LLM calls.

---

## Acceptance check (final 5 min)

For each pair:
- ✅ Notebook runs end-to-end without uncaught errors
- ✅ ≥4 of 5 résumés processed successfully
- ✅ Empty + whitespace inputs handled gracefully
- ✅ README documents the 3 errors handled with technical reasoning
- ✅ Notebook pushed (each pair pushes to ONE pair-member's repo for the lab; afternoon Sprint 1 starts the capstone arc)

If a pair has fewer than 4 résumés succeeding, sit with them for 5 minutes — usually a Pydantic schema issue (missing Optional somewhere).
