# Day 9 — Capstone Sprint 4: Career Agent (Turnkey Walkthrough)

**Time:** 90 minutes (14:00 — 15:30)
**Format:** Pair work in Colab; same pair as Days 6-8
**Goal:** LangGraph agent with 3 tools wired to Day 6/7 components. Run on 3 student profiles. 1 deliberate failure-recovery analysis.

---

## Setup (5 min)

Each pair:
- Continues Day-9 morning notebook (or opens `Day9_StudyAgent.ipynb`)
- Has Day-6 PlacementProcessor working (or its `data/jds.jsonl`)
- Has Day-7 RAG working (or its `chroma_db/`)
- Has `data/student_profiles.json` from the lab kit (3 profiles)

---

## Step 1 — Tool 1: jd_fetcher (10 min)

Reuses the Day 6 fetch_jd. Wraps as @tool.

```python
import requests
from bs4 import BeautifulSoup
from langchain_core.tools import tool

@tool
def jd_fetcher(url: str) -> str:
    """Fetch a job description from a URL and return clean plain text.
    Use when the user provides a job posting URL and you need the JD content.
    Returns first 4000 characters of the cleaned page text."""
    try:
        r = requests.get(url, headers={'User-Agent': 'Mozilla/5.0'}, timeout=10)
        r.raise_for_status()
        soup = BeautifulSoup(r.text, 'html.parser')
        for tag in soup(['script', 'style']):
            tag.decompose()
        return soup.get_text(separator='\n', strip=True)[:4000]
    except Exception as e:
        return f'ERROR: failed to fetch URL — {e}'
```

**Trainer note:** The doc-string says "Returns first 4000 characters" — be specific in doc-strings. The model uses these to decide tool selection.

**Acceptance:** Tool callable, returns text on a sample URL OR an ERROR string on bad URL.

---

## Step 2 — Tool 2: skills_gap (5 min)

Pure function, no LLM. Set difference.

```python
@tool
def skills_gap(student_skills: str, must_have_skills: str) -> str:
    """Compare a student's skills (comma-separated) to a job's must-have skills (comma-separated).
    Returns missing skills, comma-separated, or 'none' if student has all.
    Use when the user provides a student profile and a JD's required skills."""
    a = set(s.strip().lower() for s in student_skills.split(',') if s.strip())
    b = set(s.strip().lower() for s in must_have_skills.split(',') if s.strip())
    missing = sorted(b - a)
    return ', '.join(missing) if missing else 'none'

# Test
print(skills_gap.invoke({
    'student_skills': 'Python, Java, SQL',
    'must_have_skills': 'Python, Java, SQL, Spring Boot, AWS',
}))
# Expected: 'aws, spring boot'
```

**Acceptance:** Returns expected missing skills.

---

## Step 3 — Tool 3: answer_scorer (10 min)

Uses Gemini to score a student's interview answer.

```python
from langchain_google_genai import ChatGoogleGenerativeAI
llm = ChatGoogleGenerativeAI(model='gemini-2.5-flash')

@tool
def answer_scorer(question: str, answer: str) -> str:
    """Score a student's answer to a placement interview question, 1-10, with one-line rationale.
    Use when evaluating how well a student answered a specific interview question.
    Returns format: 'Score: X/10. Rationale: <reason>'."""
    prompt = (f'Score this placement interview answer 1-10 with one-line rationale.\n'
              f'Question: {question}\n'
              f'Answer: {answer}')
    return llm.invoke(prompt).content

# Test
print(answer_scorer.invoke({
    'question': 'Why TCS Digital?',
    'answer': 'Because TCS is big and they pay well.',
}))
# Expected: low score (~3-4/10), rationale about lack of specificity / cultural fit
```

**Acceptance:** Returns Score + Rationale.

---

## Step 4 — Wire all 3 tools as ReAct agent (10 min)

```python
from langgraph.prebuilt import create_react_agent

tools = [jd_fetcher, skills_gap, answer_scorer]
agent = create_react_agent(llm, tools=tools)
print(f'Agent created with {len(tools)} tools.')
```

**Acceptance:** No errors.

---

## Step 5 — Run on 3 student profiles (20 min)

```python
import json, pathlib
profiles = json.loads(pathlib.Path('../data/student_profiles.json').read_text())

for i, p in enumerate(profiles):
    print(f'\n{"="*70}')
    print(f'Student {i+1}: {p["name"]} — {p["branch"]} CGPA {p["cgpa"]} → {p["target_company"]}')
    print(f'{"="*70}')

    msg = (f"I am {p['name']}, B.Tech {p['branch']} CGPA {p['cgpa']}, "
           f"skills: {', '.join(p['skills'])}. Target: {p['target_company']}. "
           f"Plan 3 mock interview questions for me, score one of my sample answers, "
           f"and tell me what skills I need to add to be a strong fit.")

    result = agent.invoke({'messages': [('user', msg)]}, config={'recursion_limit': 10})

    for j, m in enumerate(result['messages']):
        print(f'\n  [{j}] {type(m).__name__}')
        if hasattr(m, 'content') and m.content:
            print(f'      {str(m.content)[:300]}')
        if hasattr(m, 'tool_calls') and m.tool_calls:
            for tc in m.tool_calls:
                print(f'      → tool_call: {tc.get("name")}({tc.get("args")})')
```

### Expected pattern

For each profile, the agent should:
1. Realise it needs the JD (calls `jd_fetcher` if a URL is in the message, otherwise infers from training knowledge — limit of single-tool-call planning)
2. Compute skills gap (calls `skills_gap`)
3. Generate 3 mock questions (no tool — uses LLM directly)
4. Score one answer if an example is provided
5. Synthesise final advice

Total: 4-7 messages per profile.

**Trainer note:** Watch how the agent uses ALL THREE tools in different runs. This is the design — different student questions invoke different tools. The agent's choice IS the demonstration.

**Acceptance:** 3 successful agent runs with full traces.

---

## Step 6 — Trigger 1 deliberate failure (10 min)

```python
# Pass a bad URL — see how agent recovers
result = agent.invoke({
    'messages': [('user', 'Fetch this JD and tell me the must-have skills: '
                          'https://this-does-not-exist-99999.example.com/jd')]
}, config={'recursion_limit': 5})

print('Failure recovery trace:')
for j, m in enumerate(result['messages']):
    print(f'\n[{j}] {type(m).__name__}')
    if hasattr(m, 'content') and m.content:
        print(f'    {str(m.content)[:300]}')
    if hasattr(m, 'tool_calls') and m.tool_calls:
        for tc in m.tool_calls:
            print(f'    → {tc.get("name")}({tc.get("args")})')
```

### Expected behaviour

- Agent calls `jd_fetcher` → tool returns 'ERROR: ...' string
- Agent observes the error, decides not to retry (good) OR retries with a different URL (bad — wasteful)
- Agent reports "I could not fetch the URL — please provide a working JD URL"

If the agent silently invents JD content from the bad URL, that's the bug to flag in the README.

**Acceptance:** Failure trace captured.

---

## Step 7 — Update README + push (10 min)

```markdown
## Day 9 — Capstone Sprint 4: Career Agent

### 3 tools wired
1. **jd_fetcher** — wraps Day 6's fetch_jd. Returns clean text or ERROR string.
2. **skills_gap** — pure function set difference. Deterministic.
3. **answer_scorer** — Gemini-backed scoring 1-10 with rationale.

### 3 successful runs
| # | Student | Tools used | Outcome |
|---|---------|-----------|---------|
| 1 | Ravi Kumar (CSE) → TCS | skills_gap, answer_scorer | Skill-gap: Spring Boot, AWS |
| 2 | Sneha Reddy (ECE) → Cognizant | skills_gap | Strong match — focus on interview practice |
| 3 | Arun Pillai (IT) → Amazon | skills_gap, answer_scorer | Strong match — score 8/10 on sample |

### 1 failure-recovery analysis

Bad URL passed to jd_fetcher. Agent received `ERROR:` from tool. Agent correctly responded:
"I could not fetch the JD URL. Please provide a working URL." No hallucinated JD content.
This is the safe behaviour. If the agent had hallucinated, the fix would be to tighten the doc-string of jd_fetcher.

### Engineer Answer

1. **PROBLEM** — A static RAG cannot take actions. Students need an assistant that fetches JDs, computes skill gaps, and evaluates their answers — autonomously.

2. **ARCHITECTURE** — LangGraph ReAct agent with 3 specialised tools. Each tool is a plain Python function with a precise doc-string. Agent reasons about which tool to call.

3. **TRADE-OFFS** —
   - Cost: 5-15 LLM calls per task (each ~1-3K tokens). ~20K tokens per student session.
   - Latency: 5-10s per task (LLM calls dominant).
   - Reliability: tools must return predictable strings. ERROR returns are part of the contract.
   - Complexity: doc-strings are now part of the prompt. Bad doc-string = wrong tool.

4. **SCALE** —
   - 1 student / minute: free quota OK.
   - 50 students / day: hits free quota. Switch to paid.
   - 1K students / day: needs caching + parallel inference.

5. **INTERVIEW ANSWER** — "I built a 3-tool LangGraph agent that takes a student profile and produces tailored placement prep — JD analysis, skill gap, answer scoring. Each tool is a plain function; the agent picks which to call. Failure recovery is built into tool contracts."
```

Push.

**Acceptance:** Sprint 4 components in repo.

---

## Common bugs + recovery

- **Agent uses wrong tool** → fix the doc-string. Be specific about WHEN to use each tool.
- **Token explosion (5K tokens/run)** → set `recursion_limit=10`. Cap tool output length (already 4000 in jd_fetcher).
- **Tool returns None silently** → agent invents result. Wrap each tool with explicit error returns.
- **Agent calls jd_fetcher when no URL provided** → doc-string should say "Use ONLY when the user provides a URL."
- **`recursion_limit` exceeded** → reduce to 5 for diagnosis. The trace shows where the loop is.

---

## Trainer notes

1. **Walk the room continuously.** Sprint 4 is the most complex sprint so far — multiple tools, multiple LLM calls, multiple failure modes.
2. **The teaching moment is "tools first, agents second".** Make sure each tool was tested standalone (Steps 1-3) BEFORE wrapping into the agent (Step 4).
3. **Token budget alert.** A 4-iteration agent on a long question burns 8-15K tokens. Show the cell-output token counts and discuss free-tier headroom.
4. **The failure-recovery test is the most important step.** A pair whose agent hallucinates on bad URL has not learnt the lesson. Re-do.
5. **Acceptance verification at 15:25:** project one pair's full trace from Step 5 (any of the 3 student runs). Read it as a story for 2 minutes.

---

## Acceptance check (final 5 min — mandatory before break)

For each pair:
- ✅ 3 tools defined + tested standalone
- ✅ ReAct agent created
- ✅ 3 successful runs on 3 different student profiles (full traces in notebook)
- ✅ 1 failure-recovery trace captured + analysed
- ✅ Engineer Answer + 5 specific answers in README
- ✅ Pushed to repo

If a pair's agent hallucinated on the bad URL, sit with them after break to fix the tool contract. Day 10 multi-agent depends on solid single-agent foundation.
