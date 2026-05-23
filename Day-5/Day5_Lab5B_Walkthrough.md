# Day 5 — Lab 5B: Hugging Face Pulls (Turnkey Walkthrough)

**Time:** 90 minutes (14:00 — 15:30)
**Format:** Individual in Colab
**Goal:** Same model two ways — HF Inference API + local Colab. Build the timing table mentors will show their students.

---

## Setup (5 min)

Each mentor opens:
- Colab → load `Day5_HF_Pulls.ipynb` from the lab kit
- Their HF token (next step provisions it)

---

## Step 1 — Free Hugging Face token (5 min)

- Go to https://huggingface.co/settings/tokens
- Click **New token**
- Name: `ai-bootcamp`
- Type: **Read** (not write — read-only is enough for inference)
- Save token to 1Password as `AI_Bootcamp / HF_Token`

**Acceptance:** Token in password manager.

---

## Step 2 — Install + auth (5 min)

In Colab cell 1:

```python
!pip install -q transformers requests sentence-transformers
import os, getpass
if 'HF_TOKEN' not in os.environ:
    os.environ['HF_TOKEN'] = getpass.getpass('HF token (read scope): ')
```

**Acceptance:** No error.

---

## Step 3 — Zero-shot classifier via Inference API (15 min)

Cell 2:

```python
import os, requests, time

HF_TOKEN = os.environ['HF_TOKEN']

def hf_zero_shot_api(text, labels):
    r = requests.post(
        'https://api-inference.huggingface.co/models/facebook/bart-large-mnli',
        headers={'Authorization': f'Bearer {HF_TOKEN}'},
        json={'inputs': text, 'parameters': {'candidate_labels': labels}})
    return r.json()

# Test data — 5 résumé excerpts
resumes = [
    'Built React dashboards for 3 startups',
    'Implemented Spring Boot microservices in Java for fintech app',
    'Trained CNN for image classification using PyTorch, 87% accuracy',
    'Cleaned 100k row dataset using pandas + plotted in seaborn for monthly reports',
    'Wrote SQL queries against PostgreSQL, optimised 3 slow queries by 10x',
]
labels = ['frontend dev', 'backend dev', 'data analyst', 'ML engineer']

start = time.time()
for r in resumes:
    result = hf_zero_shot_api(r, labels)
    if 'error' in result:
        print(f'  Error: {result["error"]}')
    else:
        print(f'  {r[:50]:50} -> {result["labels"][0]} ({result["scores"][0]:.2f})')
print(f'\nAPI time: {time.time()-start:.2f}s')
```

### Expected output

```
  Built React dashboards for 3 startups              -> frontend dev (0.94)
  Implemented Spring Boot microservices in Java fo... -> backend dev (0.91)
  Trained CNN for image classification using PyTo... -> ML engineer (0.96)
  Cleaned 100k row dataset using pandas + plotted... -> data analyst (0.88)
  Wrote SQL queries against PostgreSQL, optimised... -> backend dev (0.71)

API time: ~3-15 seconds (cold start vs warm)
```

**Trainer note:** First call may return `error: Model is currently loading. Please retry in 20 seconds`. Wait, retry. Cold-start IS the lesson — flag it.

**Acceptance:** 5 résumés classified.

---

## Step 4 — Same model locally in Colab (10 min)

Cell 3:

```python
from transformers import pipeline

# This DOWNLOADS the model on first run — ~1.6GB. Be patient.
classifier = pipeline('zero-shot-classification', model='facebook/bart-large-mnli')

start = time.time()
for r in resumes:
    res = classifier(r, candidate_labels=labels)
    print(f'  {r[:50]:50} -> {res["labels"][0]} ({res["scores"][0]:.2f})')
print(f'\nLocal time (after download): {time.time()-start:.2f}s')
```

### Expected output

Same labels and similar scores. Timing: 5-15 seconds (after the initial download which takes 60-90s on first run, instant on Colab restart with same runtime).

**Trainer note:** First-run download is the cost of local inference. Mentors will sometimes assume "local is faster" — local is faster after warm-up, slower on first call.

**Acceptance:** Both API and local return matching classifications for each résumé.

---

## Step 5 — Sentiment analysis (15 min)

Cell 4:

```python
sentiment = pipeline('sentiment-analysis',
    model='distilbert-base-uncased-finetuned-sst-2-english')

# Test data — 5 mock interview answers
answers = [
    'I really enjoyed working on the team and shipped 3 features.',
    'I was the only one writing code; everyone else was slow.',
    'I learned a lot from my mentor and grew technically.',
    "I had to redo most of my teammate's work because it was wrong.",
    'My internship was great — would recommend it to anyone.',
]

print('Sentiment scores:')
for a in answers:
    result = sentiment(a)[0]
    label = result['label']
    score = result['score']
    print(f'  [{label} {score:.2f}] {a[:60]}')
```

### Expected output

```
  [POSITIVE 0.99] I really enjoyed working on the team and shipped 3 features.
  [NEGATIVE 0.96] I was the only one writing code; everyone else was slow.
  [POSITIVE 0.99] I learned a lot from my mentor and grew technically.
  [NEGATIVE 0.97] I had to redo most of my teammate's work because it...
  [POSITIVE 0.99] My internship was great — would recommend it to anyone.
```

**Acceptance:** 5 sentiment classifications, with the 2 negative answers correctly identified.

### The teaching moment

Notice answers 2 and 4 — both contain a complaint about teammates. The model classifies as NEGATIVE despite the speaker presenting them as showing their own competence. This is the teaching moment: **the model classifies surface tone, not the speaker's intent.** Use this in interview prep — students should not let bitterness slip into mock-interview answers; the AI catches it, recruiters do too.

---

## Step 6 — Build the timing table (10 min)

Cell 5:

```python
import time

def time_call(fn, n_runs=3):
    times = []
    for _ in range(n_runs):
        start = time.time()
        fn()
        times.append(time.time() - start)
    return min(times), sum(times)/len(times)

# Time API call (warm)
def call_api():
    hf_zero_shot_api('Built React dashboards', ['frontend dev', 'backend dev'])
api_min, api_avg = time_call(call_api)

# Time local call (warm)
def call_local():
    classifier('Built React dashboards', candidate_labels=['frontend dev', 'backend dev'])
local_min, local_avg = time_call(call_local)

print(f'Inference timing comparison (3 runs each, after warm-up):')
print(f'  API:   min {api_min:.2f}s | avg {api_avg:.2f}s')
print(f'  Local: min {local_min:.2f}s | avg {local_avg:.2f}s')
```

### Expected pattern

| | min | avg | Notes |
|---|-----|-----|-------|
| API | 0.5-1.5s | 0.8-2.0s | Network round-trip, GPU at HF |
| Local | 1.5-5s | 2-8s | Colab CPU/GPU varies |

API typically faster on Colab (no local GPU); flips on a real laptop with M-series GPU.

**Acceptance:** Timing table generated.

---

## Step 7 — Reflection in README (5 min)

Push `Day5_HF.ipynb` + update README:

```markdown
## Day 5 Lab 5B — Hugging Face Pulls

### Models tested
- `facebook/bart-large-mnli` — zero-shot classification
- `distilbert-base-uncased-finetuned-sst-2-english` — sentiment

### Timing comparison

| | min | avg | Notes |
|---|-----|-----|-------|
| HF Inference API | 0.8s | 1.2s | Cold-start: 20s |
| Local in Colab | 2.1s | 3.4s | Download: 60s on first run |

### When to use each (3-line reflection)

1. **API:** for low-volume, occasional calls. Avoids download. Cold-start risk on first call after idle.
2. **Local:** for batch processing 100+ items, where you want predictable latency and don't pay per call.
3. **Production rule of thumb:** if your usage exceeds the API free tier (~30K requests/month at HF), self-host. Otherwise API.
```

**Acceptance:** Notebook + table + 3-line reflection in repo.

---

## Common bugs + recovery

- **`error: Model is currently loading`** on first API call → retry after 20s. Cold-start is real.
- **Colab disk full** during local download → `!rm -rf ~/.cache/huggingface` between models.
- **`OSError: We couldn't connect to 'https://huggingface.co'`** → corporate firewall. Switch to API (uses a different endpoint) or use a personal hotspot.
- **Sentiment label inconsistency** ("POSITIVE" vs "POS") between transformer versions → handle both: `label.upper().startswith('POS')`.
- **Timing varies wildly** between runs → use min, not single-call. Network jitter is real.

---

## Trainer notes

1. **The API vs local timing is the lesson.** Mentors expect local to be faster; it isn't always (especially on Colab CPU). The right answer is "depends on volume, latency requirements, cost model". Make this concrete with their numbers.
2. **The sentiment teaching moment** (answer 2/4 — surface tone vs speaker intent) is the most pedagogically rich moment. Pause the room. Discuss for 3 minutes. This is what they'll teach students about mock-interview tone.
3. **First-time HF download takes 60-90s.** Pre-cache via shared Colab if your Wi-Fi is slow.
4. **Stretch goal:** add a third model — `sentence-transformers/all-MiniLM-L6-v2` for embeddings. Time it. Used in Day 7's RAG lab — preview.
5. **Acceptance verification at 15:25:** ask each mentor to show cell 5 (timing table) on the projector for 30s.

---

## Acceptance check (final 5 min)

For each mentor:
- ✅ Notebook runs end-to-end (zero-shot + sentiment, both API and local)
- ✅ Timing table in README with concrete numbers
- ✅ 3-line reflection on when to use API vs local
- ✅ Notebook pushed to repo

If a mentor's local pipeline failed (Colab OOM or disk full), focus on the API path. The reflection can still note "local needed cleanup".
