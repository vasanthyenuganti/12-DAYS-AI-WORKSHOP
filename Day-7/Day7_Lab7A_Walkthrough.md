# Day 7 — Lab 7A: ChromaDB Hello-World (Turnkey Walkthrough)

**Time:** 90 minutes (11:15 — 13:00)
**Format:** Individual in Colab; trainer demos pre-installed Colab once
**Goal:** Embed 10 syllabus paragraphs, index in ChromaDB, run 3 semantic queries, visualise embeddings with PCA. Foundation for the afternoon's RAG sprint.

---

## Setup (5 min)

Each mentor opens:
- Colab → load `Day7_RAG_Chatbot.ipynb` from the lab kit (Lab 7A is cells 1-4 of this notebook)
- One syllabus excerpt from `data/syllabi_cached/` (CSE Sem 5 is the default)

---

## Step 1 — Install + load embedding model (10 min)

Cell 1:

```python
!pip install -q chromadb sentence-transformers scikit-learn matplotlib

from sentence_transformers import SentenceTransformer
from chromadb import PersistentClient

# Free, local, 384-dim embeddings
embed = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')
print('Embedding dim:', embed.get_sentence_embedding_dimension())
# Expected: 384
```

**Trainer note:** First run downloads ~80MB model. Subsequent cells are instant.

**Acceptance:** Output prints `Embedding dim: 384`.

---

## Step 2 — Load 10 syllabus paragraphs (5 min)

Cell 2:

```python
with open('../data/syllabi_cached/cse_sem5.txt') as f:
    text = f.read()

# Split on blank lines into paragraphs
paragraphs = [p.strip() for p in text.split('\n\n') if p.strip()][:10]

print(f'Loaded {len(paragraphs)} paragraphs')
for i, p in enumerate(paragraphs):
    print(f'  [{i+1}] {p[:80]}')
```

### Expected output

```
Loaded 10 paragraphs
  [1] CSE Semester 5 — Syllabus Excerpt
  [2] Operating Systems: Process management, threads, scheduling algorithms...
  [3] Memory management: paging, segmentation, virtual memory, page replacement.
  [4] File systems: directory structures, file allocation methods.
  [5] Synchronisation: critical section, semaphores, monitors, classical problems.
  [6] Deadlocks: prevention, avoidance (Banker's algorithm), detection, recovery.
  [7] Database Management Systems: ER model, relational model, normalisation...
  [8] SQL: DDL, DML, joins, subqueries, views, triggers.
  [9] Transactions: ACID, concurrency control (locking, timestamp ordering)...
  [10] Indexing: B+ trees, hashing.
```

**Acceptance:** 10 paragraphs printed.

---

## Step 3 — Embed and index in ChromaDB (10 min)

Cell 3:

```python
client = PersistentClient(path='./chroma_db')
col = client.get_or_create_collection('hello_syllabus')

# Embed all 10 paragraphs
vectors = embed.encode(paragraphs).tolist()

# Add to collection (with sequential IDs)
col.add(
    documents=paragraphs,
    embeddings=vectors,
    ids=[f'p{i}' for i in range(len(paragraphs))]
)

print(f'Indexed {col.count()} documents')
```

**Trainer note:** ChromaDB persists to disk in `./chroma_db/`. Mentors who restart Colab can re-load the collection without re-indexing. Useful for the afternoon Sprint 2.

**Acceptance:** Output `Indexed 10 documents`.

---

## Step 4 — Run 3 semantic queries (15 min)

Cell 4:

```python
queries = [
    'what is dynamic programming?',
    'machine learning topics',
    'operating system processes',
]

for q in queries:
    print(f'\nQuery: {q}')
    qv = embed.encode([q]).tolist()
    results = col.query(query_embeddings=qv, n_results=3)
    docs = results['documents'][0]
    distances = results['distances'][0]
    for j, (d, dist) in enumerate(zip(docs, distances)):
        print(f'  [{j+1}] (dist={dist:.3f}) {d[:80]}')
```

### Expected pattern

For "operating system processes" — top result should be the paragraph starting "Operating Systems: Process management..." (paragraph 2). Distance ≈ 0.4-0.6 for top match.

For "what is dynamic programming?" — note this topic is NOT in the syllabus excerpt. The system returns the closest semantic match (probably scheduling algorithms or transactions). This IS the lesson — semantic search returns nearest, not exact.

For "machine learning topics" — top results will be from CSE Sem 5 which has no ML. The system returns whatever is closest. This shows the limit of semantic search on out-of-corpus queries.

**Trainer note:** When a query returns irrelevant results, mentors will assume "the embedding model is bad". Wrong — the model is doing its job (returning the closest match). The lesson is: RAG with citation-enforcement on Day 7 afternoon will catch this and refuse to answer.

**Acceptance:** 3 queries return 3 results each. Mentor notes which queries return relevant vs irrelevant top-1.

---

## Step 5 — Visualise embeddings with PCA (15 min)

Cell 5:

```python
from sklearn.decomposition import PCA
import matplotlib.pyplot as plt
import numpy as np

vectors_arr = np.array(vectors)

pca = PCA(n_components=2)
xy = pca.fit_transform(vectors_arr)

plt.figure(figsize=(10, 8))
plt.scatter(xy[:, 0], xy[:, 1], s=100, alpha=0.6)
for i, p in enumerate(paragraphs):
    label = p[:30] + '...' if len(p) > 30 else p
    plt.annotate(label, (xy[i, 0], xy[i, 1]), fontsize=8)
plt.title('Syllabus Paragraph Embeddings (PCA 2D)')
plt.xlabel('PC1')
plt.ylabel('PC2')
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

### Expected pattern

Mentors should see clusters: OS-related paragraphs cluster together; DBMS-related paragraphs cluster separately. The visual confirms semantic similarity is captured.

**Trainer note:** This is the visceral teaching moment. Mentors who have only seen embeddings as "hidden math" finally see them as "spatial layout where similar topics live near each other". Show 2 mentors' plots on the projector.

**Acceptance:** Plot rendered, mentor can identify at least 2 clusters.

---

## Step 6 — Add 1 outlier paragraph (stretch, 10 min)

Cell 6:

```python
# Add a totally unrelated paragraph to see where it lands
outlier = "Today's special at the cafeteria is butter chicken with rice and naan."

col.add(
    documents=[outlier],
    embeddings=embed.encode([outlier]).tolist(),
    ids=['outlier_food']
)

# Re-fetch all and re-plot
all_docs = col.get(include=['embeddings', 'documents'])
all_vecs = np.array(all_docs['embeddings'])
labels = all_docs['ids']

pca = PCA(n_components=2) # Re-initialize PCA to ensure it fits the new data
xy = pca.fit_transform(all_vecs)
plt.figure(figsize=(10, 8))
colors = ['red' if 'outlier' in l else 'blue' for l in labels]
plt.scatter(xy[:, 0], xy[:, 1], c=colors, s=100, alpha=0.6)
for i, l in enumerate(labels):
    short = labels[i] if 'outlier' in labels[i] else all_docs['documents'][i][:30] + '...'
    plt.annotate(short, (xy[i, 0], xy[i, 1]), fontsize=8)
plt.title('With outlier (red)')
plt.show()
```

### Expected output

The food outlier sits visibly far from all syllabus paragraphs. Demonstrates that semantic distance correlates with topical distance.

**Acceptance:** Outlier visibly off to one side of the cluster.

---

## Step 7 — Push notebook (5 min)

Update README:

```markdown
## Day 7 Lab 7A — ChromaDB Hello-World

- Embedded 10 CSE Sem 5 paragraphs with all-MiniLM-L6-v2 (384-dim, free)
- Indexed in persistent ChromaDB collection `hello_syllabus`
- Ran 3 semantic queries — observed: top-1 match is relevant when query topic is in corpus, irrelevant when not
- Plotted PCA 2D — visible OS / DBMS clusters

**Reflection:** Semantic search returns nearest, not exact. RAG must enforce citations to catch out-of-corpus queries (this afternoon's Sprint 2).
```

Push.

**Acceptance:** Notebook + README + PCA plot screenshot all in repo.

---

## Common bugs + recovery

- **`sentence-transformers download stalls`** → pre-cached version in shared Colab template. Or re-run in 1-2 minutes (network).
- **ChromaDB collection persists across runtime restarts** — that IS the design. Mentors will see "collection already exists" warning on the second run; that's fine.
- **`PCA fit_transform expects 2D array`** → convert via `np.array(vectors)`. Mentors sometimes pass list-of-lists incorrectly.
- **Plot doesn't render in Colab** → restart runtime + re-run. Matplotlib quirk.
- **Top-k returns irrelevant chunks** → flag as the design intent. Semantic search returns nearest. Citations are tomorrow's defence.

---

## Trainer notes

1. **The PCA plot is the visceral teaching moment.** Mentors who see embeddings as "hidden math" will now see them as "spatial layout". Project 2-3 plots on the screen.
2. **The "machine learning topics" query returning irrelevant results is the BEST teaching moment.** It exposes the limit of semantic search alone. Set up the afternoon: "we add citations + refuse-when-not-found to fix this."
3. **Walk the room continuously.** Mentors who get the dependency error or the model download stall waste 15 minutes; you keep them moving.
4. **Stretch goal for fast finishers:** add 5 more paragraphs from a DIFFERENT subject (e.g., Mech Sem 5). Re-plot. Watch the new cluster appear.
5. **Acceptance verification at 12:50:** project one mentor's PCA plot. Discuss what the clusters represent for 2 minutes.

---

## Acceptance check (final 5 min)

For each mentor:
- ✅ ChromaDB collection has 10 documents
- ✅ 3 semantic queries return 3 results each (top-1 distances visible)
- ✅ PCA plot rendered with visible clusters
- ✅ Reflection in README on semantic vs exact search
- ✅ Notebook pushed

If a mentor's plot doesn't render, they push the notebook anyway. The afternoon Sprint 2 doesn't depend on the plot, only on the embedding/index pattern working.
