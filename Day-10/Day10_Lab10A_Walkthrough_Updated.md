# Day 10 — Lab 10A: Hello-CrewAI Updated Walkthrough

**Time:** 90 minutes  
**Format:** Individual in Colab; trainer demonstrates one complete run first  
**Goal:** Build a **2-agent CrewAI crew** consisting of a **Researcher Agent** and **Writer Agent** to generate a 1-page **TCS Digital Placement Prep Brief**. Save both the transcript and the final markdown file.

---

## Big Idea of the Lab

Day-9 introduced a **single AI agent** that can use tools.  
Day-10 introduces **multiple agents working together**.

```text
Researcher Agent
↓
Writer Agent
↓
Final Markdown Brief
```

The core learning outcome is:

> The handoff between agents is the design quality.

---

## Setup

Each mentor opens:

- Google Colab
- `Day10_MultiAgent.ipynb`
- Gemini API key

Trainer note:

> Use a fresh runtime because CrewAI, LangChain, Gemini, NumPy and LiteLLM versions may conflict if Day-8/Day-9 packages are already mixed.

---

## Step 1 — Clean Install

Run this in a fresh Colab runtime:

```python
!pip install -q "numpy<2.0"
!pip install -q --upgrade crewai litellm google-generativeai google-genai
```

Then restart runtime:

```text
Runtime → Restart Session
```

### Why this step matters

CrewAI may fail with NumPy 2.x using errors such as:

```text
np.float_ was removed in NumPy 2.0
```

So we force NumPy below version 2.0.

Acceptance:

```text
Installation completes without red traceback.
```

Warnings are okay. Import errors are not okay.

---

## Step 2 — Set Gemini API Key

```python
import os, getpass

os.environ["GEMINI_API_KEY"] = getpass.getpass("Enter Gemini API Key: ")

print("API key set")
```

### Explanation for students

> CrewAI manages the agents. Gemini provides the intelligence.

---

## Step 3 — Import CrewAI

```python
from crewai import Agent, Task, Crew, Process

print("CrewAI imported successfully")
```

Acceptance:

```text
CrewAI imported successfully
```

---

## Step 4 — Select Gemini Model

Use Gemini through CrewAI/LiteLLM string format.

```python
llm = "gemini/gemini-2.5-flash"

print(llm)
```

### Important

Do **not** pass `ChatGoogleGenerativeAI(...)` object directly to `Agent()` in this setup, because some CrewAI versions expect either a string model name or a CrewAI-compatible LLM wrapper.

Correct:

```python
llm = "gemini/gemini-2.5-flash"
```

Avoid:

```python
llm = ChatGoogleGenerativeAI(...)
```

---

## Step 5 — Create Researcher Agent

```python
researcher = Agent(
    role="Placement Researcher",
    goal="Compile concise, factual placement-prep notes for B.Tech students",
    backstory="You are a research analyst who prepares short factual placement briefs in bullet form.",
    llm=llm,
    verbose=True,
    allow_delegation=False,
)

print("Researcher agent created")
```

### Explanation

The Researcher Agent is responsible for collecting useful placement-related information.

Trainer line:

> The Researcher prepares the raw material for the Writer.

---

## Step 6 — Create Writer Agent

```python
writer = Agent(
    role="Placement Brief Writer",
    goal="Convert research notes into a clean 1-page placement brief in markdown",
    backstory="You write clear student-friendly placement briefs with headings and actionable takeaways.",
    llm=llm,
    verbose=True,
    allow_delegation=False,
)

print("Writer agent created")
```

### Explanation

The Writer Agent converts the research notes into a student-friendly document.

Trainer line:

> Different agents must have different responsibilities.

---

## Step 7 — Create Research Task

```python
research_task = Task(
    description=(
        "Research TCS Digital placement preparation for B.Tech students. "
        "Include hiring process, technical stack, eligibility, recent hiring trends, "
        "and preparation tips. Give 3-5 bullets per section."
    ),
    agent=researcher,
    expected_output=(
        "Markdown bullet list with 5 sections: Hiring Process, Technical Stack, "
        "Eligibility, Recent Hiring Trends, Prep Tips."
    ),
)

print("Research task created")
```

### Explanation

A task is the work assigned to an agent.

Very important:

> `expected_output` is the contract between agents.

If expected output is vague, the next agent receives poor input.

---

## Step 8 — Create Writer Task

```python
write_task = Task(
    description=(
        "Using the research notes, produce a 1-page placement brief for B.Tech students. "
        "Use markdown format. Include a 3-line opening hook, 5 clear sections, "
        "and a 3-line closing call-to-action."
    ),
    agent=writer,
    expected_output='1-page markdown brief titled "TCS Digital — Placement Prep Brief".',
)

print("Writer task created")
```

### Explanation

The Writer Task uses the Researcher output and produces the final brief.

Flow:

```text
Research Task output
↓
Writer Task input
↓
Final Markdown Brief
```

---

## Step 9 — Create Crew

```python
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,
    verbose=True,
)

print("Crew created successfully")
```

### Explanation

Crew means team.

```text
Agent = worker
Task = assignment
Crew = team
Process = workflow style
```

Here we use:

```python
Process.sequential
```

Meaning:

```text
Researcher works first
↓
Writer works next
```

---

## Step 10 — Run Crew in Colab

Use async execution because Colab/Jupyter already runs an event loop.

```python
result = await crew.kickoff_async()

print("\n\n=== FINAL OUTPUT ===\n")
print(result)
```

### Why not `crew.kickoff()`?

In Colab, `crew.kickoff()` may fail with:

```text
Agent execution was invoked synchronously from within a running event loop
```

So we use:

```python
await crew.kickoff_async()
```

Acceptance:

```text
Crew runs successfully and final TCS Digital brief is printed.
```

---

## Step 11 — Save Transcript

```python
import pathlib

pathlib.Path("day10_lab10a_transcript.txt").write_text(str(result))

print("Saved: day10_lab10a_transcript.txt")
print("Characters:", len(str(result)))
```

### Explanation

The transcript is proof of what the AI crew produced.

Trainer line:

> The transcript is the explanation.

---

## Step 12 — Inspect Task Outputs

Some CrewAI versions expose `tasks_output`. Some do not. Use this safe version:

```python
try:
    for i, output in enumerate(result.tasks_output):
        print(f"\n=== Task {i+1} Output ===")
        print(str(output)[:800])
except Exception:
    print("Task outputs not available in this CrewAI version.")
    print("Final result saved successfully.")
```

### Explanation

This helps mentors see the handoff:

```text
Researcher output → Writer input → Final result
```

---

## Step 13 — Create Markdown File

```python
markdown_content = str(result)

with open("tcs_digital_brief.md", "w", encoding="utf-8") as f:
    f.write(markdown_content)

print("Markdown file created successfully: tcs_digital_brief.md")
```

### Explanation

The AI crew is not only chatting. It is generating a reusable document.

---

## Step 14 — Download Markdown File

```python
from google.colab import files

files.download("tcs_digital_brief.md")
```

Acceptance:

```text
tcs_digital_brief.md downloads successfully.
```

---

## Step 15 — Optional: Read Markdown File

```python
with open("tcs_digital_brief.md", "r", encoding="utf-8") as f:
    print(f.read()[:2000])
```

---

## Step 16 — README Update

Add this to README:

```markdown
## Day 10 Lab 10A — Hello-CrewAI

### Goal
Built a 2-agent CrewAI system that generates a 1-page TCS Digital placement preparation brief.

### Agents
1. **Placement Researcher** — prepares factual placement notes.
2. **Placement Brief Writer** — converts notes into a student-friendly markdown brief.

### Workflow
Researcher → Writer → Final Markdown Brief

### Files Generated
- `day10_lab10a_transcript.txt`
- `tcs_digital_brief.md`

### Reflection
1. The handoff between agents is the design quality.
2. `expected_output` is the contract between agents.
3. Verbose mode helps debug multi-agent workflows.
```

---

## Common Errors and Fixes

### Error 1 — NumPy error

```text
np.float_ was removed in NumPy 2.0
```

Fix:

```python
!pip install -q "numpy<2.0"
```

Restart runtime.

---

### Error 2 — LLM validation error

```text
Input should be a valid string
```

Reason:

CrewAI is not accepting `ChatGoogleGenerativeAI(...)` object directly.

Fix:

```python
llm = "gemini/gemini-2.5-flash"
```

Then recreate agents, tasks, and crew.

---

### Error 3 — Running event loop issue

```text
Agent execution was invoked synchronously from within a running event loop
```

Fix:

```python
result = await crew.kickoff_async()
```

---

### Error 4 — Gemini model not found

```text
gemini-1.5-flash is not found
```

or

```text
gemini-2.0-flash is no longer available
```

Fix:

```python
llm = "gemini/gemini-2.5-flash"
```

Then rerun Steps 5 to 10.

---

### Error 5 — Quota exceeded

If Gemini quota fails:

- Run only one crew execution.
- Avoid repeated retries.
- Use trainer key for demo.
- Continue explanation using saved sample output.

Trainer line:

> Quota failure is not code failure. It is cloud-resource limitation.

---

## Trainer Teaching Notes

### What to say before the lab

> Morning we understood multi-agent AI. Now we will build a real two-agent system.

### What to write on board

```text
Researcher Agent
↓
Writer Agent
↓
1-page Brief
```

### Most important concept

> The output of one agent becomes the input of the next agent.

### Best analogy

```text
Human team:
Research assistant → Content writer

AI team:
Researcher Agent → Writer Agent
```

### Most important line

> The transcript is the design quality.

---

## Final Acceptance Checklist

Each mentor should complete:

- 2-agent crew created
- Sequential process used
- Final TCS Digital brief generated
- Transcript saved
- Markdown file generated
- README updated
- Notebook pushed

---

## Final Learning Outcome

By the end of Lab 10A, mentors understand how to design a small multi-agent AI workflow using CrewAI. They learn how to define agent roles, assign tasks, control workflow order, inspect outputs, and generate reusable artifacts such as transcripts and markdown reports. This prepares them for Sprint 5, where the same concept expands from 2 agents to 4 specialized placement-preparation agents.
