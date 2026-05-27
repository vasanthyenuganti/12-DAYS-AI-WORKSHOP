# Day 7 — Capstone Sprint 2: PlacementKnowledgeRAG (Turnkey Walkthrough)

**Time:** 90 minutes (14:00 — 15:30)
**Format:** Pair work in Colab; same pair as Day 6
**Goal:** Index Day 6's 50 JDs + 5 syllabi from the lab kit. Build a LangChain RetrievalQA chain. Every answer cites the source chunk. 5 cited Q&A pairs in README.

---

## Setup (5 min)

Each pair:
- Continues from Day 7 morning notebook (or opens `Day7_RAG_Chatbot.ipynb`)
- Has `data/jds_cached.jsonl` (10 JDs from kit) + own `data/jds.jsonl` (from Day 6 Sprint 1)
- Has `data/syllabi_cached/` with 5 syllabus excerpts

### Combine to ~50 documents

To match the v3 spec (50 JDs), use the kit's 10 + the pair's own jds.jsonl from Day 6. If counts are short, also add per-syllabus chunks (5 syllabi × ~10 chunks each ≈ 50 chunks).

---

## Step 1 — Install + load (5 min)

Cell 7 (continuing from morning):

```python
!pip install -q chromadb sentence-transformers langchain langchain-google-genai langchain-community langchain-chroma langchain-huggingface pypdf

from sentence_transformers import SentenceTransformer
from chromadb import PersistentClient
import json, pathlib, os

embed = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')
client = PersistentClient(path='./chroma_db')

# Use a fresh collection for the capstone (different from morning's hello_syllabus)
col = client.get_or_create_collection('placement_kb')
print(f'Starting count: {col.count()}')
```

**Acceptance:** Empty collection created (or existing if mentor re-runs).

---

## Step 2 — Index JDs from Day 6 + cached (15 min)

Cell 8:

```python
# Combine cached JDs (from kit) with own JDs (from Day 6)
all_jds = []

# Cached
for line in pathlib.Path('../data/jds_cached.jsonl').read_text().splitlines():
    all_jds.append(json.loads(line))

# Own (if exists from Day 6)
own = pathlib.Path('data/jds.jsonl')
if own.exists():
    for line in own.read_text().splitlines():
        all_jds.append(json.loads(line))

print(f'Total JDs: {len(all_jds)}')

# Index each JD as one document
for i, jd in enumerate(all_jds):
    text = (
        f"{jd['company']} - {jd['role']}: "
        f"must-haves: {', '.join(jd['must_have_skills'])}. "
        f"nice-to-haves: {', '.join(jd.get('nice_to_have_skills', []))}. "
        f"min CGPA: {jd.get('min_cgpa', 'N/A')}. "
        f"locations: {', '.join(jd.get('locations', []))}. "
        f"package: {jd.get('package_lpa', 'N/A')} LPA."
    )
    col.add(
        documents=[text],
        embeddings=embed.encode([text]).tolist(),
        ids=[f'jd_{i}'],
        metadatas=[{
            'type': 'jd',
            'company': jd['company'],
            'min_cgpa': float(jd.get('min_cgpa') or 0),
            'package_lpa': float(jd.get('package_lpa') or 0),
        }]
    )

print(f'Indexed {col.count()} JD documents')
```

**Acceptance:** ≥10 JDs indexed (cached set). 20+ if pair has own jds.jsonl.

---

## Step 3 — Add syllabus chunks (15 min)

Cell 9:

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=['\n\n', '\n', '. ', ' ']
)

syllabi_dir = pathlib.Path('../data/syllabi_cached')
chunk_count = 0

for syllabus_path in syllabi_dir.glob('*.txt'):
    text = syllabus_path.read_text()
    chunks = splitter.split_text(text)
    for j, chunk in enumerate(chunks):
        col.add(
            documents=[chunk],
            embeddings=embed.encode([chunk]).tolist(),
            ids=[f'{syllabus_path.stem}_{j}'],
            metadatas=[{
                'type': 'syllabus',
                'source': syllabus_path.stem,
                'chunk_index': j,
            }]
        )
        chunk_count += 1

print(f'Indexed {chunk_count} syllabus chunks')
print(f'Total docs in placement_kb: {col.count()}')
```

**Acceptance:** Total ≥50 documents in collection.

---

## Step 4 — Build LangChain RetrievalQA chain (15 min)

Cell 10:

```python
from langchain_chroma import Chroma
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.chains import RetrievalQA
from langchain.prompts import PromptTemplate

# Adapter — wrap our raw chromadb collection so LangChain Chroma sees it
emb_lc = HuggingFaceEmbeddings(model_name='sentence-transformers/all-MiniLM-L6-v2')
vs = Chroma(
    collection_name='placement_kb',
    embedding_function=emb_lc,
    persist_directory='./chroma_db',
)

# Citation-enforcing prompt
prompt_template = """Use ONLY the following context to answer. Cite the chunk id you used (e.g., "per jd_3" or "per cse_sem5_2").

If the answer is not in the context, say "I do not know" — do NOT guess.

{context}

Question: {question}
Answer:"""

llm = ChatGoogleGenerativeAI(
    model='gemini-2.5-flash',
    google_api_key=os.environ['GEMINI_API_KEY'],
)

qa = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vs.as_retriever(search_kwargs={'k': 4}),
    chain_type_kwargs={'prompt': PromptTemplate.from_template(prompt_template)},
    return_source_documents=True,
)

print('QA chain ready.')
```

**Acceptance:** No error. `qa` object exists.

---

## Step 5 — Test 5 student questions (15 min)

Cell 11:

```python
questions = [
    'Which companies want Java + DSA + CGPA 7+?',
    'What are the Sem 5 OS topics?',
    'Which JDs require Python?',
    'Top 3 skills appearing across all JDs?',
    'Companies hiring in Hyderabad?',
]

for q in questions:
    result = qa.invoke({'query': q})
    print(f'\nQ: {q}')
    print(f'A: {result["result"]}')
    print(f'Sources: {[d.metadata.get("source") or d.metadata.get("company") for d in result["source_documents"]]}')
```

### Expected outputs

For "Which companies want Java + DSA + CGPA 7+?":
- Answer should name ≥2 companies (TCS Digital, Cognizant, etc.) with citations.

For "What are the Sem 5 OS topics?":
- Answer should list paging, scheduling, deadlocks, etc. Citations to `cse_sem5_*` chunks.

For "Which JDs require Python?":
- Should cite specific jd_* IDs.

For "Companies hiring in Hyderabad?":
- TCS Digital, Microsoft, Cognizant, Deloitte etc.

For an out-of-corpus question (e.g., add "What is TCS Codevita?"):
- Should respond "I do not know" (the kit data does not include TCS Codevita info).

**Trainer note:** If "I do not know" doesn't appear for the out-of-corpus question, the citation enforcement is too weak. Tighten the prompt: add "Do NOT use prior knowledge — the context above is the ONLY source you may use."

**Acceptance:** 5 cited answers. ≥1 is "I do not know" (out-of-corpus question).

---

## Step 6 — Update README with Engineer Answer (10 min)

```markdown
## Day 7 — Capstone Sprint 2: PlacementKnowledgeRAG

### Engineer Answer

1. **PROBLEM** — Frontier LLMs do not know your private data (JDs, syllabi). Students need a chatbot that answers from YOUR placement corpus, with citations they can verify.

2. **ARCHITECTURE** — 5-box RAG: embed (MiniLM 384-dim) → index (ChromaDB persistent collection with metadata) → retrieve (top-4 cosine similarity) → augment (citation-enforcing prompt) → generate (Gemini 2.5).

3. **TRADE-OFFS** —
   - Cost: free (MiniLM local + Gemini quota).
   - Accuracy: top-4 retrieval has ~80% precision on placement-relevant queries.
   - Latency: ~1-2s per query (embedding + retrieval) + 2-5s (Gemini).
   - Complexity: chunking strategy (500-token, 50-overlap) needs tuning per corpus.
   - Caveat: refuses out-of-corpus queries (good!) but only when prompt enforces "do not guess".

4. **SCALE** —
   - 50 docs (today): trivial. ChromaDB returns in <100ms.
   - 5K docs: still fine on one machine.
   - 1M docs: need vector DB optimisation (HNSW indexing, server mode), or move to Pinecone/Weaviate.

5. **INTERVIEW ANSWER** — "I built a citation-enforcing RAG over 50+ placement docs (JDs + syllabi) using free MiniLM embeddings, ChromaDB, and Gemini. The system either cites a specific chunk or refuses — no hallucinated answers. Same pattern scales to thousands of docs without retraining."

### 5 cited Q&A pairs

| # | Question | Answer (excerpt) | Sources cited |
|---|----------|------------------|---------------|
| 1 | Which companies want Java + DSA + CGPA 7+? | "Per jd_0 (TCS Digital): Java + DSA required, CGPA 7.0 cutoff..." | jd_0, jd_5, jd_8 |
| 2 | Sem 5 OS topics? | "Per cse_sem5_2: paging, segmentation, virtual memory..." | cse_sem5_2, cse_sem5_5 |
| 3 | Which JDs require Python? | "Per jd_3 (Accenture)..., per jd_5 (Cognizant)..." | jd_3, jd_5, jd_9 |
| 4 | Top 3 skills across JDs? | "Java, Python, SQL appear in 7+ of 10 JDs..." | jd_0, jd_1, jd_2 |
| 5 | What is TCS Codevita? | "I do not know — not in corpus." | (none) |
```

Push to repo.

**Acceptance:** Engineer Answer + 5 cited Q&A in README. Notebook + ChromaDB folder pushed.

---

## Common bugs + recovery

- **Top-k returns irrelevant chunks** → chunk size too big. Reduce to 250 tokens. Re-index.
- **No citations in answers** → prompt is not strict enough. Add "Cite the chunk id" mandatory + "Do NOT use prior knowledge" rule.
- **`I do not know` for in-corpus questions** → retrieval not finding the chunks. Inspect with `vs.similarity_search(q, k=8)` to debug retrieval before suspecting Gemini.
- **LangChain version drift** breaks Gemini integration → pin requirements.txt with exact versions from `requirements.txt` in the lab kit.
- **ChromaDB persistence path** different on second run → use absolute path or `pathlib.Path.cwd() / 'chroma_db'`.

---

## Trainer notes

1. **The "I do not know" answer is the most important pedagogical artefact.** When mentors see Gemini refuse to answer out-of-corpus, they viscerally understand "RAG with citations means the model says I don't know when it shouldn't guess." Project this on screen.
2. **Walk the room during Step 5 (test 5 questions).** Mentors who don't see citations in their first answer will keep tweaking; intervene quickly.
3. **The Engineer Answer must be specific.** "We built a RAG" is generic. "We built a citation-enforcing RAG with ChromaDB persistent collection, MiniLM embeddings, top-4 retrieval, and a refuse-on-out-of-corpus prompt" is specific.
4. **Prepare for tomorrow:** Day 8 morning starts with you breaking a mentor's RAG live with 3 crafted questions. The pair whose RAG you break gets the "winner" prize — they helped the cohort learn.
5. **Acceptance verification at 15:25:** show one pair's `qa.invoke({'query': '...'})` cell live on projector. Verify the citations appear.

---

## Acceptance check (final 5 min — mandatory before break)

For each pair:
- ✅ Day7_RAG_Chatbot.ipynb runs end-to-end
- ✅ ChromaDB collection has ≥50 documents
- ✅ 5 cited Q&A pairs in README (≥1 is "I do not know")
- ✅ Engineer Answer with all 5 questions answered specifically
- ✅ Notebook + chroma_db/ folder pushed to repo

If a pair has answers without citations, prompt-tightening for 5 minutes after the break. The citation rule is non-negotiable for every later sprint.
