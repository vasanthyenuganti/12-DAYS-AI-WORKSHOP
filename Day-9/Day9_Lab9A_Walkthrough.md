# Day 9 — Lab 9A: Hello-LangGraph (Turnkey Walkthrough)

**Time:** 90 minutes (11:15 — 13:00)
**Format:** Individual in Colab; trainer demos one full agent run live first
**Goal:** Smallest working ReAct agent with 1 tool. Print full trace. 1 deliberate failure-recovery analysis.

---

## Setup (5 min)

Each mentor opens:
- Colab → load `Day9_StudyAgent.ipynb` from the lab kit
- Their Gemini key (1Password)

---

## Step 1 — Install + key (5 min)

```python
!pip install -q langgraph langchain-google-genai langchain-community duckduckgo-search

import os, getpass
if 'GEMINI_API_KEY' not in os.environ:
    os.environ['GEMINI_API_KEY'] = getpass.getpass('Gemini API key: ')
```

**Acceptance:** No errors.

---

## Step 2 — Define one tool: web_search (10 min)

```python
from langchain_core.tools import tool
from langchain_community.tools import DuckDuckGoSearchRun

@tool
def web_search(query: str) -> str:
    """Search the web for up-to-date information.
    Use when the question requires current events, recent facts, or
    information not in static training knowledge."""
    return DuckDuckGoSearchRun().run(query)

# Test the tool directly
print(web_search.invoke({'query': 'TCS hiring 2026'})[:400])
```

### Expected output

A multi-paragraph search result mentioning TCS hiring activity.

**Trainer note:** The doc-string IS the model's instruction for when to call this tool. Walk the room and ensure every mentor wrote a doc-string. Empty doc-string = bad tool selection later.

**Acceptance:** Tool returns text without error.

---

## Step 3 — Create the ReAct agent (10 min)

```python
from langgraph.prebuilt import create_react_agent
from langchain_google_genai import ChatGoogleGenerativeAI

llm = ChatGoogleGenerativeAI(model='gemini-2.5-flash')
agent = create_react_agent(llm, tools=[web_search])

print('Agent created.')
```

**Acceptance:** No errors.

---

## Step 4 — Ask a live-fact question (15 min)

```python
result = agent.invoke({
    'messages': [('user', "What is TCS's 2026 hiring quota?")]
})

# Print every message in the conversation
for i, m in enumerate(result['messages']):
    print(f'\n[{i}] {type(m).__name__}')
    if hasattr(m, 'content'):
        print(f'    Content: {str(m.content)[:300]}')
    if hasattr(m, 'tool_calls') and m.tool_calls:
        print(f'    Tool calls: {m.tool_calls}')
```

### Expected output

```
[0] HumanMessage
    Content: What is TCS's 2026 hiring quota?

[1] AIMessage
    Tool calls: [{'name': 'web_search', 'args': {'query': 'TCS 2026 hiring quota'}}]

[2] ToolMessage
    Content: <DuckDuckGo search results — multi-paragraph>

[3] AIMessage
    Content: Based on the search results, TCS plans to hire approximately 40,000-50,000
             freshers in 2026, with focus on...
```

The trace IS the explanation. 4 messages: human → AI thinks + calls tool → tool returns → AI synthesises answer.

**Trainer note:** This is the moment mentors realise they can DEBUG agents. Pause. Read the trace aloud. "Thought, action, observation, answer." That's the loop they'll teach students.

**Acceptance:** 4-message trace printed.

---

## Step 5 — Read the trace as a story (10 min)

In a markdown cell:

```markdown
## Day 9 Lab 9A — Trace as a story

1. **Human asked:** "What is TCS's 2026 hiring quota?"
2. **Agent thought:** "I don't know recent figures. I should search."
3. **Agent acted:** called `web_search('TCS 2026 hiring quota')`.
4. **Agent observed:** got back search results mentioning 40-50K range.
5. **Agent answered:** synthesised "Based on search results, TCS plans to hire 40-50K freshers..."

This is the ReAct loop. Every agent we build follows this pattern.
```

**Acceptance:** Trace narrative in notebook.

---

## Step 6 — Trigger a deliberate failure (15 min)

```python
# Pass a question that should fail the tool
result = agent.invoke({
    'messages': [('user', 'Search this URL and tell me what it says: https://this-domain-does-not-exist-12345.example.com/jd')]
})

# Watch how the agent recovers
for i, m in enumerate(result['messages']):
    print(f'\n[{i}] {type(m).__name__}')
    if hasattr(m, 'content'):
        print(f'    {str(m.content)[:300]}')
```

### Expected behaviour

The DuckDuckGo search returns "no results" or an error. The agent observes this and either:
- (good) Reports "I could not find information about that URL"
- (bad) Hallucinates information about the URL

Document which behaviour you observed.

**Trainer note:** If your agent hallucinates here, that's a teaching moment. The doc-string told the agent to use the tool for "up-to-date information" — but didn't tell it what to do when the tool returns no results. Real-world resilience means defining failure modes.

**Acceptance:** Failure-recovery trace captured. Mentor documents in markdown cell which behaviour they saw.

---

## Step 7 — Push (5 min)

Update README:

```markdown
## Day 9 Lab 9A — Hello-LangGraph

- 1-tool ReAct agent with DuckDuckGo web_search
- 4-message trace on a live-fact question (TCS 2026 hiring)
- Failure case: bad URL → agent reported "could not find" / agent hallucinated [pick one]

### Reflection (3 lines)

1. The trace IS the explanation. Print every step.
2. The doc-string IS the prompt. Bad doc-string = bad tool selection.
3. Real agents handle tool failures gracefully — define failure modes in the doc-string.
```

Push.

**Acceptance:** Notebook + README pushed.

---

## Common bugs + recovery

- **DuckDuckGo wrapper signature changed** → check `langchain_community.tools` for the latest. May need `DuckDuckGoSearchAPIWrapper` instead.
- **Agent loops forever** → set `max_iterations=10` in `agent.invoke({'messages': ..., 'recursion_limit': 10})`.
- **Empty tool response** → agent invents an answer (silent hallucination). Tighten doc-string: "If no results found, return the string 'NO_RESULTS_FOUND'."
- **Token explosion** (long search results in context) → wrap web_search to return only top 500 chars per result.
- **Gemini quota hit during multi-call agent** → switch to backup key.

---

## Trainer notes

1. **The trace is the lesson.** Every step gets printed. Project one mentor's trace on the screen and walk through it as a story.
2. **The doc-string IS the prompt.** When a mentor's agent picks the wrong tool (later, with multiple tools), the fix is always the doc-string.
3. **Failure-recovery is the teaching moment.** When the bad-URL test produces hallucination, mentors learn that agents need explicit failure modes.
4. **Stretch goal:** add a second tool — e.g., a calculator. Watch the agent choose between tools based on doc-strings. Preview of Day 9 afternoon.
5. **Acceptance verification at 12:50:** project one mentor's full trace + their failure case. Mentors who haven't seen a failure trace in their notebook can watch the demo.

---

## Acceptance check (final 5 min)

For each mentor:
- ✅ Hello-LangGraph agent runs end-to-end
- ✅ 4-message trace printed on live-fact question
- ✅ 1 failure case captured + behaviour documented
- ✅ "Trace as a story" markdown cell in notebook
- ✅ Pushed to repo

If a mentor's agent doesn't run, pair-debug for 5 min. The afternoon Sprint 4 needs this pattern working.
