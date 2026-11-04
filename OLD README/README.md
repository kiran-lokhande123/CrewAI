# Agentic RAG: Router–Retriever System with PDF and Web Search Tools

A two-agent Retrieval-Augmented Generation (RAG) system built with **CrewAI**. A **Router Agent** classifies each question, and a **Retriever Agent** answers it using the right source:

- **PDF search** (static knowledge): the paper *"Attention Is All You Need"* (the Transformer paper)
- **Web search** (dynamic knowledge): live internet results through Tavily
- **Direct LLM** (optional): simple general questions that need no retrieval

---

## 1. System architecture

```
                    ┌──────────────────────┐
  User question ──► │     Router Agent     │  (no tools, temperature 0)
                    │ classify → JSON      │
                    └──────────┬───────────┘
                               │ RouteDecision {route, reason, search_query}
                ┌──────────────┼──────────────────┐
                ▼              ▼                  ▼
         pdf_search       web_search          direct_llm
      PDFSearchTool     Tavily search        (no tool)
     (vector search)   (TavilySearchResults)
                └──────────────┼──────────────────┘
                               ▼
                    ┌──────────────────────┐
                    │   Retriever Agent    │  gets ONLY the chosen tool
                    │ retrieve → answer    │
                    └──────────┬───────────┘
                               ▼
              Grounded answer + sources + trace log
```

| Component | Technology |
|---|---|
| Orchestration | CrewAI (`Agent`, `Task`, `Crew`, `Process.sequential`) |
| LLM | OpenAI `gpt-4o-mini` (via `crewai.LLM`) |
| PDF retrieval | `crewai_tools.PDFSearchTool` (chunks + embeddings + vector store) |
| Web retrieval | `langchain_community.tools.tavily_search.TavilySearchResults`, wrapped with CrewAI `@tool` |
| Structured routing | Pydantic model `RouteDecision` (`output_pydantic`) |
| Logging | CrewAI `step_callback` / `task_callback`, `output_log_file`, Python `logging`, JSONL trace file |
| Visualisation | pandas tables and matplotlib charts |

---

## 2. Agents: logic and responsibilities

### Router Agent: "Query Router"
- **Input:** the user's natural-language question.
- **Logic:** its backstory describes exactly what the PDF covers (architecture, attention, positional encoding, training, BLEU results and so on). It chooses:
  - `pdf_search` for anything about the Transformer paper
  - `web_search` for current, recent or external facts (news, latest models, versions, events after 2017)
  - `direct_llm` for trivial general questions (for example, unit conversion)
- **Output:** `RouteDecision` JSON with `route`, `reason` and an optimised `search_query`.
- **Tools:** none. It only reasons, which keeps routing fast, cheap and predictable.

### Retriever Agent: "Retrieval Specialist"
- **Input:** the question plus the router's decision.
- **Logic:** calls its single assigned tool, reads the evidence and writes a 3–8 sentence answer using **only** that evidence. It cites PDF sections or web URLs and says so when the evidence is insufficient.
- **Output:** the final answer with a `Sources:` line.
- **Tools:** whichever tool the router picked, and nothing else.

---

## 3. Coordination flow

1. `answer_question(question)` creates a run id and logs the question.
2. **Routing crew** (Router Agent + routing task) runs and returns a `RouteDecision`.
3. The coordinator validates the decision. If the LLM output can't be parsed, a **keyword fallback router** is used.
4. A Retriever Agent is created with the tool for that route (`TOOLS_BY_ROUTE`).
5. **Retrieval crew** (Retriever Agent + retrieval task that includes the router decision) runs and returns the answer.
6. Every hand-off, thought, tool call, tool result and final answer is logged to:
   - `TRACE` (in memory, shown as a table and charts in the notebook)
   - `logs/agentic_rag_trace.jsonl` (machine-readable audit trail)
   - `logs/agentic_rag.log`, `logs/crew_router.log`, `logs/crew_retriever.log`
7. The notebook shows reasoning-trace visualisations: a step-by-step trace, swim-lane timelines per run, route distribution, latency per route, agent/event counts and an architecture diagram.

### Why CrewAI?
- Role, goal and backstory map directly to a role-based multi-agent design.
- Native tools (`PDFSearchTool`) and an easy `@tool` wrapper for LangChain tools such as Tavily.
- Structured outputs (`output_pydantic`) make agent-to-agent hand-offs machine-readable.
- Built-in callbacks, verbose mode and log files give traceability with little code.
- Lighter than hand-building a state graph (for example, LangGraph) for a simple two-step sequential pipeline.

---

## 4. How to run

1. Put these files in one folder:
   - `Agentic_RAG_Router_Retriever.ipynb`
   - `trasformer_research_paper-dataset.pdf`
2. Create a `.env` file:
   ```
   OPENAI_API_KEY=sk-...
   TAVILY_API_KEY=tvly-...
   ```
   (If the keys are missing, the notebook prompts for them securely.)
3. Use Python 3.10–3.13. Open the notebook and choose **Run All**. The first cell installs:
   `crewai crewai-tools langchain-community tavily-python pydantic pandas matplotlib python-dotenv`
4. The demo runs six questions: three PDF, two web and one direct. Section 9 lets you ask your own questions.

---

## 5. Challenges faced and trade-offs made

| Challenge | Approach / trade-off |
|---|---|
| **Routing errors on ambiguous questions** (for example, "What is a Transformer?" could be the paper or the web) | The router backstory lists the PDF scope in detail; temperature 0 keeps decisions consistent. A keyword fallback covers malformed output. Trade-off: one LLM call per question adds latency and cost. |
| **Retriever using the wrong tool** | The Retriever Agent is rebuilt per question with **only** the chosen tool. Trade-off: it can't switch sources on its own if evidence is weak; the router is the single point of decision. |
| **Hallucination / ungrounded answers** | The prompts require answers to use only the retrieved evidence, with citations and an explicit "insufficient evidence" option. Trade-off: answers can be shorter or more cautious. |
| **Two crews instead of one** | The Python-coordinated hand-off makes the input/output flow explicit, testable and fully logged. Trade-off: slightly more code than a single crew with `Process.hierarchical`. |
| **Tool and library version changes** (CrewAI callbacks return different step objects; `TavilySearchResults` is being replaced by `langchain-tavily`) | The step callback reads fields defensively with `getattr`; the Tavily call is wrapped in try/except. |
| **PDF indexing cost** | `PDFSearchTool` embeds the PDF once per session, so the first query is slower. Trade-off: the default chunking is simple, and tables or equations in the paper may be split across chunks. |
| **Web results quality** | Limited to the top 4 results with source URLs, which balances context size against coverage. |
| **Cost / latency** | `gpt-4o-mini` keeps cost low; a larger model would route and write more accurately at a higher cost. |

---

## 6. Possible improvements
- A hybrid route (PDF + web) for questions that compare the paper with newer models
- A self-check step (a third "Grader" agent) that re-routes when evidence is weak
- Result caching, and evaluation with a labelled routing test set (routing accuracy)
- CrewAI Flows, or LangGraph for conditional loops
