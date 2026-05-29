# Day 10 — Capstone Sprint 5: Placement Prep Crew (Updated Turnkey Walkthrough)

## Time
90 Minutes (14:00 — 15:30)

## Goal
Build a 4-agent CrewAI placement preparation workflow:

Researcher → Interviewer → Coach → Tracker

Generate:
- Personalized placement preparation workflow
- JSON transcripts
- Markdown report
- Multi-agent orchestration understanding

---

# IMPORTANT UPDATED FIXES

## Use this model

```python
llm = "gemini/gemini-2.5-flash"
```

## Use async kickoff in Colab

```python
result = await crew.kickoff_async()
```

## Import json before transcript save

```python
import json
import pathlib
```

## IMPORTANT

After changing model:
- recreate agents
- recreate tasks
- recreate crew

Otherwise old model remains cached.

---

# Step-1 — Install Packages

```python
!pip install -q "numpy<2.0"
!pip install -q --upgrade crewai litellm google-generativeai google-genai chromadb sentence-transformers crewai-tools
```

Restart runtime after installation.

```text
Runtime → Restart Session
```

---

# Step-2 — Set Gemini API Key

```python
import os, getpass

os.environ["GEMINI_API_KEY"] = getpass.getpass("Enter Gemini API Key: ")

print("API key set")
```

---

# Step-3 — Imports

```python
from crewai import Agent, Task, Crew, Process
from crewai_tools import tool as crewai_tool

from chromadb import PersistentClient
from sentence_transformers import SentenceTransformer

import json
import pathlib
```

---

# Step-4 — Select Model

```python
llm = "gemini/gemini-2.5-flash"

print(llm)
```

---

# Step-5 — Reuse Day-7 RAG Tool

```python
client_db = PersistentClient(path='./chroma_db')

col = client_db.get_or_create_collection('placement_kb')

embed = SentenceTransformer(
    'sentence-transformers/all-MiniLM-L6-v2'
)

@crewai_tool
def rag_search(query: str) -> str:
    """
    Search placement knowledge base.
    """

    qv = embed.encode([query]).tolist()

    results = col.query(
        query_embeddings=qv,
        n_results=4
    )

    docs = results['documents'][0]

    return '\n---\n'.join(docs)
```

Acceptance:
- Tool callable successfully

---

# Step-6 — Create 4 Agents

## Researcher Agent

```python
researcher = Agent(
    role="Placement Researcher",

    goal="Research company placement requirements for students",

    backstory="You prepare factual placement research notes using RAG search.",

    llm=llm,

    tools=[rag_search],

    verbose=True,

    allow_delegation=False,
)
```

---

## Interviewer Agent

```python
interviewer = Agent(
    role="Mock Interviewer",

    goal="Generate placement interview questions",

    backstory="You generate technical and HR interview questions.",

    llm=llm,

    verbose=True,

    allow_delegation=False,
)
```

---

## Coach Agent

```python
coach = Agent(
    role="Answer Coach",

    goal="Create strong sample answers and preparation guidance",

    backstory="You coach students with strong placement answers.",

    llm=llm,

    verbose=True,

    allow_delegation=False,
)
```

---

## Tracker Agent

```python
tracker = Agent(
    role="Progress Tracker",

    goal="Create structured student progress summaries",

    backstory="You generate JSON-style student progress summaries.",

    llm=llm,

    verbose=True,

    allow_delegation=False,
)
```

Acceptance:
- 4 agents created successfully

---

# Step-7 — Create Student Profiles

```python
profiles = [
    {
        "student_id": "S001",
        "name": "Ravi Kumar",
        "branch": "CSE",
        "cgpa": 7.8,
        "skills": ["Python", "Java", "SQL"],
        "target_company": "TCS Digital"
    },

    {
        "student_id": "S002",
        "name": "Sneha Reddy",
        "branch": "ECE",
        "cgpa": 8.1,
        "skills": ["Python", "DBMS"],
        "target_company": "Cognizant"
    },

    {
        "student_id": "S003",
        "name": "Arun Pillai",
        "branch": "IT",
        "cgpa": 8.5,
        "skills": ["Java", "DSA", "OOPs"],
        "target_company": "Amazon"
    }
]
```

---

# Step-8 — Create Sequential Tasks

```python
def build_tasks(profile):

    research = Task(
        description=(
            f"Research placement preparation details for "
            f"{profile['target_company']} "
            f"for a {profile['branch']} student."
        ),

        agent=researcher,

        expected_output="Short bullet research notes."
    )

    interview = Task(
        description=(
            f"Generate EXACTLY 10 mock interview questions "
            f"for {profile['name']} targeting "
            f"{profile['target_company']}."
        ),

        agent=interviewer,

        expected_output="Exactly 10 numbered interview questions."
    )

    coaching = Task(
        description=(
            f"Pick question 3 and create strong sample answer "
            f"for {profile['name']}."
        ),

        agent=coach,

        expected_output="One strong answer and 3 preparation tips."
    )

    tracking = Task(
        description=(
            f"Create JSON-style progress summary "
            f"for {profile['student_id']}."
        ),

        agent=tracker,

        expected_output="Valid JSON-style summary."
    )

    return [research, interview, coaching, tracking]
```

Acceptance:
- Function returns 4 tasks

---

# Step-9 — Run for One Student First

```python
p = profiles[0]

crew = Crew(
    agents=[
        researcher,
        interviewer,
        coach,
        tracker
    ],

    tasks=build_tasks(p),

    process=Process.sequential,

    verbose=True,
)
```

---

# Step-10 — Execute Crew

IMPORTANT:
Use async kickoff in Colab.

```python
result = await crew.kickoff_async()

print("\n=== FINAL OUTPUT ===\n")

print(result)
```

Acceptance:
- Crew executes successfully

---

# Step-11 — Run for All Students

```python
transcripts = []

for p in profiles:

    print("\n" + "="*60)

    print(f"Running for {p['name']} → {p['target_company']}")

    print("="*60)

    crew = Crew(
        agents=[
            researcher,
            interviewer,
            coach,
            tracker
        ],

        tasks=build_tasks(p),

        process=Process.sequential,

        verbose=True,
    )

    result = await crew.kickoff_async()

    transcripts.append({
        "student": p["name"],
        "target": p["target_company"],
        "final_output": str(result),
    })

print("Completed:", len(transcripts))
```

---

# Step-12 — Save JSON Transcripts

```python
pathlib.Path(
    "day10_sprint5_transcripts.json"
).write_text(
    json.dumps(transcripts, indent=2)
)

print("Saved transcripts successfully")
```

---

# Step-13 — Create Markdown Report

```python
md = "# Day10 Sprint5 Report\n\n"

for t in transcripts:

    md += f"## {t['student']} → {t['target']}\n\n"

    md += t["final_output"]

    md += "\n\n---\n\n"

pathlib.Path(
    "day10_sprint5_report.md"
).write_text(md)

print("Markdown report created")
```

---

# Step-14 — Download Files

```python
from google.colab import files

files.download("day10_sprint5_transcripts.json")

files.download("day10_sprint5_report.md")
```

---

# Common Errors + Fixes

## Error

```text
NameError: json is not defined
```

Fix:

```python
import json
```

---

## Error

```text
kickoff() event loop error
```

Fix:

```python
await crew.kickoff_async()
```

---

## Error

```text
model not found
```

Fix:

```python
llm = "gemini/gemini-2.5-flash"
```

---

## Error

```text
old model still used
```

Fix:
Recreate:
- agents
- tasks
- crew

---

# Important Trainer Notes

1. The transcript IS the architecture.
2. Read handoffs between agents.
3. Multi-agent means:
   - clear roles
   - clear outputs
   - clear orchestration
4. Researcher uses tools.
5. Other agents use outputs from previous agents.

---

# Acceptance Checklist

- 4 agents created
- Sequential workflow executed
- 3 students processed
- JSON transcripts saved
- Markdown report created
- Multi-agent orchestration understood

---

# Final Engineering Answer

## Problem
Single-agent systems struggle with multi-step placement preparation workflows.

## Architecture
Researcher → Interviewer → Coach → Tracker sequential pipeline.

## Trade-offs
- More agents = more cost
- More agents = more explainability
- More orchestration = better modularity

## Scale
- 3 students easy
- 300 students requires batching

## Interview Answer
"I built a 4-agent CrewAI placement preparation workflow with orchestration, transcripts, structured outputs, and reusable reporting."
