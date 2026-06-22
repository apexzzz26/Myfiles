# Log Analyzer AI Agent — Design

**Date:** 2026-06-22
**Status:** Approved design (simple version)
**Build window:** 4 weeks

---

## What it does

You give it your **code**, your **past logs**, and your **design doc**. You then paste in an **error and its log lines**. It tells you three things:

1. **Why and how** the error happened
2. **Which files** to look at to fix it (ranked)
3. **How that error shows up in the logs**

Every answer points to a real `file:line` or a real log line, so the operator can trust it.

## Built for this setup

- **Fully on-prem** — runs on your own machine; no log data leaves the network.
- **Local models** — `Qwen3:8b` via Ollama for reasoning, `all-MiniLM-L6-v2` for search.
- **Language-agnostic** — works across programming languages.
- **Input** — log files / batches (semi-structured text).
- **Surface** — an internal web app (chat-style input + saved report history).

**Key design principle:** the model is small (8B), so the leverage is in **good retrieval + tight structure**, not raw model cleverness. We feed it the *right* code and log pieces, force it to answer in a fixed format with citations, and verify every claim. That keeps hallucination low and runs fast on local hardware.

---

## Agent Structure

Five simple parts.

1. **Indexer** — reads code, past logs, and the design doc once and turns them into a searchable library.
   - Code is split into small pieces (by function), and the location of every logging statement is recorded.
   - Log lines are grouped into repeating **patterns**.
   - Everything is stored so it can be searched **by keyword and by meaning**.
2. **Retriever** — when an error is submitted, it finds the matching pieces: the exact code from the stack trace, similar past logs, and the related part of the design doc.
3. **Analyzer (Qwen3)** — reads the error plus those pieces and writes the answer (why/how, ranked suspect files, log-pattern explanation) in a fixed format, with citations.
4. **Checker** — confirms every `file:line` and log line it cited actually exists. Anything unverifiable is flagged. This keeps the small model honest.
5. **Web app** — paste an error → get a clear report (click a suspect file to view it, see the log timeline, see citations). Past reports are saved.

### Flow

```
code + logs + doc  →  Indexer  →  searchable library
                                       │
you paste an error  →  Retriever  →  Analyzer  →  Checker  →  Report
                                                  (why · which files · log pattern)
```

The "orchestrator" that connects these is a plain Python function — roughly five calls, read top to bottom:

```
normalize(error) → retrieve(pieces) → ask_qwen(structured) → check(answer) → render(report)
```

---

## Tech Stack

| Part | Tool | Why |
|---|---|---|
| The "brain" (LLM) | Ollama + `qwen3:8b`, wrapped by LangChain **`ChatOllama`** | Local; `.with_structured_output()` gives clean JSON answers with no manual parsing. |
| Search by meaning | `all-MiniLM-L6-v2` via LangChain **`HuggingFaceEmbeddings`** | Your pick — small, fast, runs on CPU (256-token cap → keep chunks small). |
| Search storage | Chroma via LangChain **`Chroma`** | Simplest vector store to set up; standard retriever interface. |
| Keyword search | LangChain **`BM25Retriever`** (`rank_bm25`) | Catches exact tokens (error codes, identifiers) that meaning-search misses. |
| Hybrid search | LangChain **`EnsembleRetriever`** | Merges keyword + meaning results automatically — less code than hand-rolling it. |
| Read code | **tree-sitter** (custom, no LangChain) | Splits code by function in any language **and** extracts every log call-site (powers the WOW feature). |
| Read logs | **Drain3** (custom) | Groups raw log lines into repeating patterns / signatures. |
| Backend | Python + **FastAPI** + **SQLite** | One language, no heavy database. |
| Frontend | **React** | Clickable report + saved history. |
| Orchestrator | Plain Python (no LangChain agents / LCEL chains) | Keeps control flow visible and debuggable. |
| Run it all | **Docker Compose** | One command starts everything (Ollama, Chroma, API, web). |

---

## How LangChain is used (and where it isn't)

LangChain is used as a **library of components**, not as the framework that owns the control flow. This adds real, demonstrable LangChain usage while keeping the project simple.

**Used (because it removes boilerplate):**
- `ChatOllama` + `.with_structured_output(schema)` — clean JSON answers against a fixed schema.
- `EnsembleRetriever` (combining `BM25Retriever` + the Chroma vector retriever) — hybrid keyword+meaning search with built-in result merging.
- `HuggingFaceEmbeddings` and the `Chroma` wrapper — thin glue with standard interfaces.

**Deliberately NOT used (because it adds complexity):**
- LangChain **Agents** / `AgentExecutor`, **LangGraph**, and long **LCEL pipe-chains**. The control flow stays in one plain Python function.

**Kept custom (LangChain can't do these):**
- tree-sitter code parsing + **log call-site extraction** (the WOW feature).
- Drain3 log-pattern mining.
- The grounding **Checker** and the web app.

**Why:** the whole pipeline stays as ~five readable function calls; LangChain appears only where it shortens the code. **Caveat:** LangChain's API changes often, so pin versions in `requirements.txt`.

> This is intentional architecture, not an oversight: LangChain/LangGraph would only earn a larger role if the project later grows the bounded multi-step agent described under "Where it goes next."

---

## Unique Use Cases

Tools like Splunk, Datadog, and Elastic watch logs but **don't know your code**. That gap is the differentiation:

1. **Points to the actual code** — not "errors went up," but "look at `charge.py:142`, here's why."
2. **Ranks the files to fix** — the core goal.
3. **Explains the log pattern** — teaches new operators where to look and what the lines mean.
4. **Knows intended behavior** — compares logs to the design doc ("doc says 3 retries, logs show 7").
5. **Runs fully offline** — for teams that can't send logs to the cloud (finance, defense, regulated industries).

---

## WOW Factor

**Click a log line → jump to the exact line of code that printed it.**

While indexing, the agent records every logging statement in the code. So when it sees a log line, it traces it back to the precise `log.error(...)` that produced it — and lays the error out as a **clickable timeline** (request → log → code → next log → crash). No commercial tool does this, it answers all three questions at once, and it is striking in a demo.

*Later stretch:* the agent remembers confirmed fixes, so it recognizes a repeat error instantly with a link to the prior fix — the "it gets smarter over time" story.

---

## 4-Week Plan (overview)

| Week | Focus | Deliverable |
|---|---|---|
| 1 | **Indexing** — Docker + Ollama; build the Indexer for code, logs, doc. | `POST /ingest` builds all indexes on a sample repo + logs. |
| 2 | **Analysis** — parse the error, retrieve the right pieces, get a checked report from Qwen3. | `POST /analyze` returns a verified report. |
| 3 | **Web app + WOW** — report UI + click-log-to-code timeline + feedback capture. | Full browser demo. |
| 4 | **Test & polish** — measure file-localization accuracy, fix rough edges, prepare the demo. | Polished demo + metrics + "where it goes next." |

**If time runs short (MVP cut-line):** ship Weeks 1–2 plus a basic report page; Python-only; make click-to-code the stretch goal.

**Top risks & mitigations:** 8B JSON reliability (→ structured output + retries); tree-sitter language coverage (→ simple line-window fallback); Drain3 tuning (→ start with known formats); latency (→ cache embeddings, keep retrieval `k` small).

## Where it goes next (phase 2)

- **Bounded multi-step agent** — let Qwen3 take a few capped tool steps (search code, read file, search logs) for multi-hop bugs. *This* is where LangGraph would earn its place.
- **Self-learning incident memory** — confirmed fixes become reusable "incident cards" the agent recognizes instantly.

---

*The step-by-step build guide for the 4 weeks above lives in `build-guide.md`.*
